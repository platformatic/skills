# @platformatic/world — World Adapter for the Vercel Workflow SDK

`@platformatic/world` is a drop-in [World](https://useworkflow.dev/docs/deploying)
implementation for the Vercel Workflow SDK that routes workflow state through
a self-hosted Platformatic Workflow Service. It pins each run to the deployment
version that started it, so in-flight runs keep executing on the code version
they started on.

This reference is for application authors using the SDK. It does not cover
hosting the Workflow Service itself — assume one is reachable at the URL
the user supplies as `PLT_WORLD_SERVICE_URL`.

---

## Install

```bash
npm install workflow @platformatic/world
```

- `workflow` — the Vercel Workflow SDK.
- `@platformatic/world` — the World adapter (this package).

Requires Node.js >= 22.19.0.

---

## Environment Variables

The SDK discovers the world via environment variables. Set these in `.env`,
`.env.local`, Kubernetes secrets, or your process manager:

| Variable | Required | Default | Description |
|---|---|---|---|
| `WORKFLOW_TARGET_WORLD` | Yes | — | Set to `@platformatic/world` |
| `PLT_WORLD_SERVICE_URL` | Yes | — | Workflow Service base URL |
| `PLT_WORLD_APP_ID` | No | `package.json` `name` | Application identifier |
| `PLT_WORLD_DEPLOYMENT_VERSION` | No | K8s label or `'local'` | Version each run pins to |
| `PORT` | No | — | Used by `world.start()` to register the local callback URL |

In Kubernetes, `PLT_WORLD_DEPLOYMENT_VERSION` is auto-detected from the pod's
`plt.dev/version` label — usually leave it unset there.

For local development, the Workflow Service runs in single-tenant mode and any
`appId` is accepted; the implicit app is named `default`.

---

## Next.js wiring

Next.js bundles workflows automatically via its SDK transform. You only need
to register the queue handler on boot. Create `instrumentation.ts` at the
project root:

```ts
// instrumentation.ts
export async function register () {
  if (process.env.PLT_WORLD_SERVICE_URL) {
    const { createWorld } = await import('@platformatic/world')
    const world = createWorld()
    await world.start?.()
  }
}
```

Next runs `register()` once per server process before accepting requests.

Trigger a workflow from a route handler or server action:

```ts
// app/api/signup/route.ts
import { start } from 'workflow/api'
import { handleSignup } from '@/workflows/signup'

export async function POST (req: Request) {
  const { email } = await req.json()
  const run = await start(handleSignup, [email])
  return Response.json({ runId: run.runId })
}
```

Workflow files use the standard Vercel directives:

```ts
// workflows/signup.ts
'use workflow'
import { sendEmail } from './steps/email'

export async function handleSignup (email: string) {
  await sendEmail(email, 'welcome')
}
```

```ts
// workflows/steps/email.ts
'use step'

export async function sendEmail (to: string, template: string) {
  // ...
}
```

### Notes on Next 16 / Turbopack

The Vercel SDK's combined route (V2, beta.8+) ships a single
`/.well-known/workflow/v1/flow` route that handles both workflow and step
execution. Older guides referencing a separate `/step` route are stale.

Turbopack is the default in Next 16. Dynamic `require()` of SDK bundles uses
the `/* turbopackIgnore: true */` annotation introduced in `@workflow/core`
beta.8 — older versions need a webpack fallback.

---

## Manual `world.start()`

For any Node host that is not Next.js (Express, Koa, Hono, Fastify without
the plugin, plain `http`), call `world.start()` explicitly on boot:

```ts
import express from 'express'
import { createWorld } from '@platformatic/world'

const app = express()
// ... routes

const world = createWorld()
await world.start?.()

app.listen(Number(process.env.PORT))
```

`world.start()`:
- Registers the pod's callback endpoints (`/.well-known/workflow/v1/{flow,step,webhook}`)
  with the Workflow Service.
- Builds the callback URL from `PORT` and (in K8s) the pod's IP.
- Is a no-op under ICC (the Platformatic control plane registers handlers).
- Is idempotent — calling it twice is safe but wasteful.

If you skip `world.start()`, queue messages from the Workflow Service will
never reach this process and runs will hang in `pending` forever.

For a non-Next host that authors its own `'use workflow'` / `'use step'`
files, the Vercel SDK transform doesn't run automatically. Use the
standalone build (`npx workflow build --target standalone`) and either:

- Mount the resulting `flow`/`step`/`webhook` handlers manually on your
  routes, or
- Use [`@platformatic/workflow-fastify`](fastify.md), which mounts them for
  you on Fastify.

---

## Triggering Runs

Always import from `workflow/api`, not from `@platformatic/world`:

```ts
import { start, resumeHook } from 'workflow/api'
```

### By imported reference

Inside files the SDK transforms (Next.js routes, server actions, transformed
modules), pass the workflow function directly:

```ts
import { handleSignup } from '@/workflows/signup'
const run = await start(handleSignup, [email])
```

### By workflowId string

Anywhere — including untransformed files like plain Express routes:

```ts
const run = await start(
  { workflowId: 'workflow//./workflows/signup//handleSignup' },
  [email]
)
```

With `@platformatic/workflow-fastify`, the workflow ID map is exposed on the
Fastify instance:

```ts
const run = await start({ workflowId: app.workflows.handleSignup }, [email])
```

### Resume a hook

For human-in-the-loop or webhook resumption:

```ts
await resumeHook(token, payload)
```

The token is the value returned from `hooks.token(...)` inside the workflow.

---

## Factory functions

### `createWorld(options?)`

High-level factory with automatic config resolution from environment variables.
This is what you usually want.

```ts
import { createWorld } from '@platformatic/world'

const world = createWorld()
// equivalent to:
// createWorld({
//   serviceUrl: process.env.PLT_WORLD_SERVICE_URL,
//   appId: process.env.PLT_WORLD_APP_ID ?? <package.json name>,
//   deploymentVersion: process.env.PLT_WORLD_DEPLOYMENT_VERSION ?? <K8s label or 'local'>,
// })
```

Any field passed explicitly overrides the env var.

### `createPlatformaticWorld(config)`

Low-level factory — all fields required, no env var resolution. Use when you
want full control (tests, advanced wiring):

```ts
import { createPlatformaticWorld } from '@platformatic/world'

const world = createPlatformaticWorld({
  serviceUrl: 'http://localhost:3042',
  appId: 'my-app',
  deploymentVersion: 'v1',
})
```

The returned `world` implements the full World interface: `storage` (runs,
events, steps, hooks), `queue`, `streams`, `encryption`. Apps almost never
call these directly — the Vercel SDK does.

---

## SDK Compatibility

`@platformatic/world` is typed against `@workflow/world@4.1.1` stable but
works at runtime against the v5 beta line:

| Installed `workflow` SDK | Status | Notes |
|---|---|---|
| `workflow@4.2.x` (stable) | OK | The declared API. Streamer calls go through flat methods. |
| `workflow@5.0.0-beta.x` | OK | The v5 SDK calls `world.streams.*` (nested namespace). Exposed at runtime alongside the v4 methods. |

Both lines are exercised in CI: v5 beta against the Vercel-compat suite,
v4 stable against the user-facing path.

---

## Spec Version

`@platformatic/world` declares `specVersion: 3` (CBOR queue transport):

- Runs created by `start()` are tagged with spec v3.
- Queue messages use CBOR framing, which preserves `Uint8Array` natively.
- The world accepts both CBOR and JSON inbound, so a v3 client and a v2-only
  server (or vice versa) coexist during rollout.

Peer dependency: `@workflow/world` ≥ 4.1.1.

---

## What `@platformatic/world` does NOT do

- It does not run workflows. Workflows run in your application process,
  invoked by the Workflow Service via the registered callback handlers.
- It does not store state. All state lives in the Workflow Service
  (PostgreSQL). The world is a thin HTTP client.
- It does not transform `'use workflow'` / `'use step'` files. That is the
  Vercel SDK's job, either via the Next.js plugin or
  `workflow build --target standalone`.
- It does not host the Workflow Service. That is a separate package
  (`@platformatic/workflow`).

---

## Resources

- [Platformatic World repo](https://github.com/platformatic/platformatic-world)
- [Vercel Workflow DevKit docs](https://useworkflow.dev/)
- [`@platformatic/workflow-fastify`](fastify.md) — optional Fastify plugin
- [Troubleshooting](troubleshooting.md)
