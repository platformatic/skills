# @platformatic/workflow-fastify

A Fastify plugin that mounts Vercel Workflow SDK handlers, backed by
[`@platformatic/world`](world.md).

The plugin is **optional**. You can host workflows on any Node server by
calling `world.start()` and mounting the standalone build's handlers
manually. The plugin just does it for you on Fastify.

---

## What it does

1. Loads the `flow`, `step`, and `webhook` handlers emitted by
   `npx workflow build --target standalone` under
   `<buildDir>/.well-known/workflow/v1/`.
2. Mounts them on Fastify at `/.well-known/workflow/v1/{flow,step,webhook}`,
   adapting Fastify `request`/`reply` to the Web `Request`/`Response` the
   handlers expect.
3. Parses the build's `manifest.json` and decorates the Fastify instance with:
   - `app.workflowManifest` — the full manifest object.
   - `app.workflows` — flat `name -> workflowId` map for triggering runs.
4. Calls `@platformatic/world`'s `start()` once on boot to register the
   queue handler with the Workflow Service. A no-op under ICC.

The plugin only adds routes, decorators, and a startup hook. Your app keeps
full ownership of its lifecycle (`fastify.listen()`, hooks, plugins).

---

## Install

```bash
npm install fastify workflow @platformatic/world @platformatic/workflow-fastify
```

Requires Node.js >= 22.19.0.

---

## Project Layout

The plugin only needs the build output at
`<buildDir>/.well-known/workflow/v1/`. No fixed source layout:

```
my-app/
├── server.ts                  # registers the plugin, triggers via app.workflows.<name>
├── workflows/
│   ├── signup.ts              # 'use workflow'
│   └── steps/
│       └── email.ts           # 'use step'
└── .well-known/workflow/v1/   # emitted by `workflow build` (don't edit/commit)
```

`buildDir` defaults to `process.cwd()`.

---

## Usage

```ts
import Fastify from 'fastify'
import { start } from 'workflow/api'
import workflowFastify from '@platformatic/workflow-fastify'

const app = Fastify()
await app.register(workflowFastify)

app.post('/api/signup', async (req) => {
  const { email } = req.body as { email: string }
  // Trigger by workflowId — server.ts is not SDK-transformed, so we can't
  // import the workflow function reference directly.
  const run = await start(
    { workflowId: app.workflows.handleSignup },
    [email]
  )
  return { runId: run.runId }
})

await app.listen({ port: Number(process.env.PORT) })
```

Build the workflows before starting (re-run when they change):

```bash
npx workflow build --target standalone
```

---

## Options

| Option | Default | Description |
|---|---|---|
| `buildDir` | `process.cwd()` | Directory containing `.well-known/workflow/v1` |
| `register` | `true` | Register the queue handler on boot. Set to `false` under ICC, or in tests. |

Example with explicit options:

```ts
await app.register(workflowFastify, {
  buildDir: '/app/build',
  register: process.env.NODE_ENV === 'production',
})
```

---

## Environment Variables

Same set as the world adapter (see [world.md](world.md)):

```
WORKFLOW_TARGET_WORLD=@platformatic/world
PLT_WORLD_SERVICE_URL=http://<workflow-engine>:3042
PLT_WORLD_APP_ID=<app id>                # optional, defaults to package name
PLT_WORLD_DEPLOYMENT_VERSION=<version>   # optional; auto-detected in K8s
PORT=<port>                              # used to register the local callback URL
```

---

## Build Notes

The standalone build emits one bundle per handler. Extensions and module
format vary by SDK version:

- `workflow@5.0.0-beta.x` — emits ESM `.mjs`.
- `workflow@4.2.x` — emits `.js` (flow/step CommonJS, webhook ESM).

The plugin sniffs the file, normalizes ambiguous `.js` to `.cjs` / `.mjs`,
and imports the correct `POST` export. You do not need to special-case the
SDK version in your code.

`manifest.json` shape (the fields the plugin uses):

```json
{
  "version": "1",
  "workflows": {
    "workflows/signup.ts": {
      "handleSignup": { "workflowId": "workflow//./workflows/signup//handleSignup" }
    }
  }
}
```

`app.workflows` flattens this to:

```ts
{ handleSignup: 'workflow//./workflows/signup//handleSignup' }
```

---

## When NOT to use the plugin

- **Next.js** — the Next.js plugin handles transform and routing. The
  Fastify plugin is for non-Next hosts.
- **Hosts other than Fastify** — use the generic Node setup in
  [SKILL.md](../SKILL.md) and mount the standalone-build handlers yourself.
- **You do not author your own workflows in this app** — if the app only
  triggers runs against workflows hosted elsewhere, you don't need the
  plugin. Trigger via `start()` directly.

---

## Resources

- [Package on npm](https://www.npmjs.com/package/@platformatic/workflow-fastify)
- [Source](https://github.com/platformatic/platformatic-world/tree/main/packages/workflow-fastify)
- [world.md](world.md) — adapter reference
- [authoring.md](authoring.md) — how to write workflows and steps
