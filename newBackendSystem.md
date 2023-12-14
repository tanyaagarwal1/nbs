# New Backend System
# Comparison

## 1. Plugins
### Creating new backend plugin:
(packages/backend/src/plugins/plugin_name.ts)

```js
import { createRouter } from '@internal/plugin-carmen-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  // Here is where you will add all of the required initialization code that
  // your backend plugin needs to be able to start!

  // The env contains a lot of goodies, but our router currently only
  // needs a logger
  return await createRouter({
    logger: env.logger,
  });
}
```
---
```js
import {
  configServiceRef,
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';

// export type ExamplePluginOptions = { exampleOption: boolean };
export const examplePlugin = createBackendPlugin({
  // unique id for the plugin
  pluginId: 'example',
  // register(env, options: ExamplePluginOptions) {
  register(env) {
    env.registerInit({
      deps: {
        logger: coreServices.logger,
        router: httpRouterServiceRef,
      },
      async init({ logger ,router}) {
        logger.info('Hello from example plugin');
        // plugin specific setup code.
        router.use('/hello', async (_req, res) =>
          res.json({ message: 'Hello World' }),
        );
      },
    });
  },
});
```

The env.registerInit method is used to register an initialization function that is run when the backend starts up. The deps argument is used to declare service dependencies, and the init callback is passed an object with the resolved dependencies. 

### Integrating the plugin to DCP:
(index.ts)

```js
import carmen from './plugins/carmen';
// ...
async function main() {
  // ...
  const carmenEnv = useHotMemoize(module, () => createEnv('carmen'));
  apiRouter.use('/carmen', await carmen(carmenEnv));
}
```
---
```js
import { createBackend } from '@backstage/backend-defaults';
import { catalogPlugin } from '@backstage/plugin-catalog-backend';

// Create your backend instance
const backend = createBackend();

// Install all desired features
backend.add(catalogPlugin());

// Start up the backend
await backend.start();
```

### Migrating old plugins to use the new backend system:

- Use LegacyPlugin (from '@backstage/backend-common') to add your plugins in the backend.
- Further, the Legacy plugin can also be customized to add custom environment variables.

```js
import { createBackend } from '@backstage/backend-defaults';
import { legacyPlugin } from '@backstage/backend-common';

const backend = createBackend();
backend.add(legacyPlugin('todo', import('./plugins/todo')));
backend.start();
```

## 2. Adding custom processor and providers (Modules):

```js
import { FrobsProvider,CustomProcessor } from '../path/to/class';

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  const builder = CatalogBuilder.create(env);
  const frobs = new FrobsProvider('production', env.reader);
  const customProcessor = new CustomProcessor('production', env.reader);
  builder.addEntityProvider(frobs);
  builder.addEntityProcessor(customProcessor);

  const { processingEngine, router } = await builder.build();
  await processingEngine.start();
}
```
---
We need to create a backend module and add these processor/provider in the catalog using the catalogProcessingExtensionPoint.

```js
import { createBackendModule } from '@backstage/backend-plugin-api';
import { catalogProcessingExtensionPoint } from '@backstage/plugin-catalog-node';
import { MyCustomProcessor } from './processor';
import { MyCustomProvider } from './provider';

export const exampleCustomProcessorCatalogModule = createBackendModule({
  pluginId: 'catalog',
  moduleId: 'example-custom-processor',
  register(env) {
    env.registerInit({
      deps: {
        catalog: catalogProcessingExtensionPoint,
      },
      async init({ catalog }) {
        catalog.addProcessor(new MyCustomProcessor());
        catalog.addEntityProvider(new MyCustomProvider());
      },
    });
  },
});
```

## 4. Extension Points:

- Each plugin is an independent separate micro-service.
- There can be no direct communication between plugins through code.
- Plugins need to export and use extension points.

### Defining an extension points -
```js
import { createExtensionPoint } from '@backstage/backend-plugin-api';

export interface ScaffolderActionsExtensionPoint {
  addAction(action: ScaffolderAction): void;
}

export const scaffolderActionsExtensionPoint =
  createExtensionPoint<ScaffolderActionsExtensionPoint>({
    id: 'scaffolder.actions',
  });
```

### Registering an extension points -
```js
export const scaffolderPlugin = createBackendPlugin(
  {
    pluginId: 'scaffolder',
    register(env) {
      const actions = new Map<string, TemplateAction<any>>();

      env.registerExtensionPoint(
        scaffolderActionsExtensionPoint,
        {
          addAction(action) {
            if (actions.has(action.id)) {
              throw new Error(`Scaffolder actions with ID '${action.id}' has already been installed`);
            }
            actions.set(action.id, action);
          },
        },
      );

      env.registerInit({
        deps: { ... },
        async init({ ... }) {
          // Use the registered actions when setting up the scaffolder ...
          const installedActions = Array.from(actions.values());
        },
      });
    },
  },
);
```
## Module

## 5. Services
- Backend services provide shared functionality available to all backend plugins and modules.
- To use a service in your plugin or module you request an implementation of that service using the service reference.

### Service refrences - 
This will create a ServiceRef instance, which is a reference that you export in order to allow users to interact with your service. Conceptually this is very similar to ApiRefs in the frontend system. 
```js
import { createServiceRef } from '@backstage/backend-plugin-api';

export interface FooService {
  foo(options: FooOptions): Promise<FooResult>;
}

export const fooServiceRef = createServiceRef<FooService>({
  id: 'example.foo', // the owner of this service is in this case the 'example' plugin
});
```
The fooServiceRef that we create above should be exported, and can then be used to declare a dependency on the FooService interface and receive an implementation of it at runtime.

### Service factory - 
```js
import { createServiceFactory } from '@backstage/backend-plugin-api';

class DefaultFooService implements FooService {
  async foo(options: FooOptions): Promise<FooResult> {
    // ...
  }
}

export const fooServiceFactory = createServiceFactory({
  service: fooServiceRef,
  deps: { bar: barServiceRef },
  factory({ bar }) {
    return new DefaultFooService(bar);
  },
});
```

```js
backend.add(fooServiceFactory());
```


## Core services

- The default backend provides several core services out of the box like access to configuration, logging, URL Readers, databases and more.
- All core services are available through the coreServices namespace in the @backstage/backend-plugin-api package.


### HTTP Router Service

HTTP router service is used to expose HTTP endpoints for other plugins to consume.

```js
import { coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { Router } from 'express';

createBackendPlugin({
  pluginId: 'example',
  register(env) {
    env.registerInit({
      deps: { http: coreServices.httpRouter },
      async init({ http }) {
        const router = Router();
        router.get('/hello', (_req, res) => {
          res.status(200).json({ hello: 'world' });
        });
        // Registers the router at the /api/example path
        http.use(router);
      },
    });
  },
});

```


### Permissions

This service allows your plugins to ask the permissions framework for authorization of user actions.

packages/backend/src/plugins/permission.ts
```js

import { createRouter } from '@backstage/plugin-permission-backend';
import {
  AuthorizeResult,
  PolicyDecision,
} from '@backstage/plugin-permission-common';
import { PermissionPolicy } from '@backstage/plugin-permission-node';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

class TestPermissionPolicy implements PermissionPolicy {
  async handle(): Promise<PolicyDecision> {
    return { result: AuthorizeResult.ALLOW };
  }
}

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    config: env.config,
    logger: env.logger,
    discovery: env.discovery,
    policy: new TestPermissionPolicy(),
    identity: env.identity,
  });
}
```

```js
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { Router } from 'express';

createBackendPlugin({
  pluginId: 'example',
  register(env) {
    env.registerInit({
      deps: {
        permissions: coreServices.permissions,
        http: coreServices.httpRouter,
      },
      async init({ permissions, http }) {
        const router = Router();
        router.get('/test-me', (request, response) => {
          // use the identity service to pull out the token from request headers
          const { token } = await identity.getIdentity({
            request,
          });

          // ask the permissions framework what the decision is for the permission
          const permissionResponse = await permissions.authorize(
            [
              {
                permission: myCustomPermission,
              },
            ],
            { token },
          );
        });

        http.use(router);
      },
    });
  },
});
```

### Database

- This service lets your plugins get a knex client hooked up to a database which is configured in your app-config YAML files, for your persistence needs.

The following example shows how to get access to the database service in an example backend plugin and getting a client for interacting with the database. It also runs some migrations from a certain directory for your plugin.

(plugin.ts)
```js
import {
  coreServices,
  createBackendPlugin,
} from '@backstage/backend-plugin-api';
import { resolvePackagePath } from '@backstage/backend-common';

createBackendPlugin({
  pluginId: 'example',
  register(env) {
    env.registerInit({
      deps: {
        database: coreServices.database,
      },
      async init({ database }) {
        const client = await database.getClient();
        const migrationsDir = resolvePackagePath(
          '@internal/my-plugin',
          'migrations',
        );
        if (!database.migrations?.skip) {
          await client.migrate.latest({
            directory: migrationsDir,
          });
        }
      },
    });
  },
});
```




