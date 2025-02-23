# explore-backend

The `explore-backend` plugin provides a backend service for the Explore plugin.
This allows your organizations tools to be surfaced in the Explore plugin
through an API. It also provides a search collator to make it possible to search
for these tools.

## Getting started

### Install the Package

```bash
# From your Backstage root directory
yarn add --cwd packages/backend @backstage/plugin-explore-backend
```

### Adding the plugin to your `packages/backend`

You'll need to add the plugin to the router in your `backend` package. You can
do this by creating a file called `packages/backend/src/plugins/explore.ts`

```ts
import {
  createRouter,
  StaticExploreToolProvider,
} from '@backstage/plugin-explore-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

// List of tools you want to surface in the Explore plugin "Tools" page.
const tools: ExploreTool[] = [
  {
    title: 'New Relic',
    description:'new relic plugin',
    url: '/newrelic',
    image: 'https://i.imgur.com/L37ikrX.jpg',
    tags: ['newrelic', 'proxy', 'nerdGraph'],
  },
  ...
];

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    logger: env.logger,
    toolProvider: StaticExploreToolProvider.fromData(exploreTools),
  });
}
```

With the `explore.ts` router setup in place, add the router to
`packages/backend/src/index.ts`:

```diff
+import explore from './plugins/explore';

async function main() {
  ...
  const createEnv = makeCreateEnv(config);

  const catalogEnv = useHotMemoize(module, () => createEnv('catalog'));
+  const exploreEnv = useHotMemoize(module, () => createEnv('explore'));

  const apiRouter = Router();
+  apiRouter.use('/explore', await explore(exploreEnv));
  ...
  apiRouter.use(notFoundHandler());
```

### Wire up Search Indexing

To index explore tools you will need to register the search collator in the
`packages/backend/src/plugins/search.ts` file.

```diff
+import { ToolDocumentCollatorFactory } from '@backstage/plugin-explore-backend';

async function createSearchEngine(
  env: PluginEnvironment,
): Promise<SearchEngine> {
  ...
+  indexBuilder.addCollator({
+    schedule,
+    factory: ToolDocumentCollatorFactory.fromConfig(env.config, {
+      discovery: env.discovery,
+      logger: env.logger,
+    }),
+  });
  ...
}
```

### Wire up the Frontend

See [the explore plugin README](../explore/README.md) for more information.

## Explore Tool Customization

The `explore-backend` uses the `ExploreToolProvider` interface to provide a list
of tools used in your organization and/or within your Backstage instance. This
can be customized to provide tools from any source. For example you could create
a `CustomExploreToolProvider` that queries an internal for tools in your
`packages/backend/src/plugins/explore.ts` file.

```ts
import {
  createRouter,
  StaticExploreToolProvider,
} from '@backstage/plugin-explore-backend';
import { Router } from 'express';
import { PluginEnvironment } from '../types';

class CustomExploreToolProvider implements ExploreToolProvider {
  async getTools(
    request: GetExploreToolsRequest,
  ): Promise<GetExploreToolsResponse> {
    const externalTools = await queryExternalTools(request);

    const tools: ExploreTool[] = [
      ...externalTools,
      // Backstage Tools
      {
        title: 'New Relic',
        description: 'new relic plugin',
        url: '/newrelic',
        image: 'https://i.imgur.com/L37ikrX.jpg',
        tags: ['newrelic', 'proxy', 'nerdGraph'],
      },
    ];

    return { tools };
  }
}

export default async function createPlugin(
  env: PluginEnvironment,
): Promise<Router> {
  return await createRouter({
    logger: env.logger,
    toolProvider: new CustomExploreToolProvider(),
  });
}
```
