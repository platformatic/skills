---
name: workflow
description: |
  Build applications that use the Vercel Workflow SDK against a self-hosted
  Platformatic Workflow Service via `@platformatic/world`.
  Use when users ask to:
  - "add workflow to my app", "use workflow sdk", "self-host vercel workflow"
  - "set up @platformatic/world", "platformatic world", "PLT_WORLD_SERVICE_URL"
  - "trigger a workflow run", "start a workflow", "resume a hook"
  - "wire workflow in Next.js", "instrumentation.ts for workflow"
  - "use workflow with Fastify", "@platformatic/workflow-fastify"
  - "workflow build --target standalone", "callback handlers", ".well-known/workflow"
  - "queue handler", "world.start()", "register handlers"
  Covers `@platformatic/world` (the World adapter), the Vercel Workflow SDK
  `'use workflow'` / `'use step'` authoring model, framework wiring
  (Next.js via `instrumentation.ts`, Node/Express/etc. via explicit
  `world.start()`), and the optional `@platformatic/workflow-fastify` plugin
  for mounting standalone-build handlers on Fastify.
  Does NOT cover deploying or operating the Workflow Service itself — assumes
  one is already reachable at `PLT_WORLD_SERVICE_URL`.
argument-hint: "[init|next|fastify|node|trigger|build|status] [framework-hint]"
allowed-tools: Read, Glob, Grep, Bash, Write, Edit
---

# Platformatic Workflow SDK Skill

You are an expert in building applications with the Vercel Workflow SDK
(`workflow` / `@workflow/*`) against [`@platformatic/world`](https://github.com/platformatic/platformatic-world),
the self-hosted World adapter for Platformatic.

This skill helps users build apps that **use** workflows. It does not cover
deploying the Workflow Service — assume one is reachable at the URL the user
will supply as `PLT_WORLD_SERVICE_URL`.

## Prerequisites Check

Before any workflow setup, verify:

1. **Node.js Version**: `@platformatic/world` requires Node.js v22.19.0+
   ```bash
   node --version
   ```

2. **Workflow Service URL**: The user needs a reachable Workflow Service.
   - Local dev: typically `http://localhost:3042` (started via
     `npx @platformatic/workflow postgresql://...`).
   - K8s: provided by the Platformatic control plane (ICC) or operator.
   If the user does not have one, point them at
   [`@platformatic/workflow` quick start](https://github.com/platformatic/platformatic-world#quick-start)
   before continuing — but do not try to start the service from this skill.

3. **Existing package.json**: This skill installs into an existing Node app.

## Command Router

Based on user input ($ARGUMENTS), route to the appropriate workflow:

| Input Pattern | Action |
|---|---|
| `init`, `setup`, `add`, (empty) | Run **Detect Framework + World Setup** |
| `next`, `nextjs`, `next.js` | Run **Next.js Setup** |
| `node`, `express`, `koa`, `generic` | Run **Generic Node.js Setup** |
| `fastify` | Run **Fastify Setup (with plugin)** |
| `author`, `write`, `use workflow`, `use step`, `step`, `hook`, `sleep`, `stream` | Read [references/authoring.md](references/authoring.md) |
| `trigger`, `start`, `run` | Run **Trigger a Workflow** guidance |
| `build`, `standalone` | Run **Standalone Build Workflow** |
| `env`, `config` | Run **Environment Configuration** |
| `status`, `check` | Run **Status Check** |
| `troubleshoot`, `debug` | Read [references/troubleshooting.md](references/troubleshooting.md) |

---

## Detect Framework + World Setup

When the user asks to "add workflow" without specifying a framework:

1. Detect the framework by inspecting the project:
   - `next.config.{js,ts,mjs}` or `next` in `package.json` → **Next.js Setup**
   - `fastify` in `package.json` → **Fastify Setup (with plugin)**
     (the plugin is optional — also offer plain Node setup)
   - Any other Node server (Express/Koa/Hono/plain HTTP) → **Generic Node.js Setup**
2. Confirm the detection with the user before generating files.

In all cases, the core install is the same:

```bash
npm install workflow @platformatic/world
```

`workflow` is the Vercel SDK; `@platformatic/world` is the World adapter that
points the SDK at the Platformatic Workflow Service.

Then read:

- [references/world.md](references/world.md) — full client reference (env
  vars, `createWorld`, `world.start()`, framework wiring).
- [references/authoring.md](references/authoring.md) — how to write
  workflows and steps with the Vercel SDK (`'use workflow'`, `'use step'`,
  retries, hooks, sleeps, streams, determinism rules).

Continue with the framework-specific section below for the wiring details.

---

## Next.js Setup

When the user runs `'use workflow'` / `'use step'` files inside a Next.js app:

1. Install:
   ```bash
   npm install workflow @platformatic/world
   ```
2. Read [references/world.md](references/world.md) — section "Next.js wiring".
3. Create `instrumentation.ts` at the project root:
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
   This registers the app's queue handler endpoints (`/.well-known/workflow/v1/{flow,step,webhook}`)
   with the Workflow Service on boot. In K8s + ICC, `world.start()` is a no-op
   (ICC registers handlers via the control plane).
4. Set environment variables in `.env.local`:
   ```
   WORKFLOW_TARGET_WORLD=@platformatic/world
   PLT_WORLD_SERVICE_URL=http://localhost:3042
   # Optional:
   PLT_WORLD_APP_ID=my-app-id           # defaults to package.json name
   PLT_WORLD_DEPLOYMENT_VERSION=local   # auto-detected in K8s
   ```
5. Author workflows the standard Vercel way — the SDK transforms
   `'use workflow'` / `'use step'` files automatically inside Next.js.
6. Trigger runs from a route handler or server action:
   ```ts
   import { start } from 'workflow/api'
   import { handleSignup } from '@/workflows/signup'

   await start(handleSignup, [email])
   ```
7. Verify with `npm run dev` and watch the Next.js logs for the
   `[workflow]` handler registration line.

---

## Generic Node.js Setup

For Node servers without a framework-specific transform (Express, Koa, Hono,
plain `http`), the SDK still works but you call `world.start()` explicitly.

1. Install:
   ```bash
   npm install workflow @platformatic/world
   ```
2. Read [references/world.md](references/world.md) — section "Manual world.start()".
3. Call `world.start()` once during server boot, before accepting traffic:
   ```ts
   import { createWorld } from '@platformatic/world'

   const world = createWorld()
   await world.start?.()
   // ...then app.listen(PORT)
   ```
4. Set environment variables (see [references/world.md](references/world.md)).
5. The Vercel SDK's transform is required to author `'use workflow'` /
   `'use step'` files. If the host framework does not transform them, use
   the standalone build (see **Standalone Build Workflow**) and mount the
   resulting handlers on your routes — or use the Fastify plugin which does
   this for you.

---

## Fastify Setup (with plugin)

When the user wants to host Vercel Workflow SDK workflows inside a Fastify
app, recommend `@platformatic/workflow-fastify`. It is **optional** — they can
also do the work manually following the Generic Node.js Setup.

1. Install:
   ```bash
   npm install fastify workflow @platformatic/world @platformatic/workflow-fastify
   ```
2. Read [references/fastify.md](references/fastify.md).
3. Author workflow files (e.g. `workflows/signup.ts`) with
   `'use workflow'` / `'use step'`.
4. Build the standalone bundles (see **Standalone Build Workflow** below):
   ```bash
   npx workflow build --target standalone
   ```
   This emits `flow`, `step`, `webhook`, and `manifest.json` under
   `.well-known/workflow/v1/`.
5. Register the plugin in your Fastify app:
   ```ts
   import Fastify from 'fastify'
   import { start } from 'workflow/api'
   import workflowFastify from '@platformatic/workflow-fastify'

   const app = Fastify()
   await app.register(workflowFastify)

   app.post('/api/signup', async (req) => {
     const run = await start(
       { workflowId: app.workflows.handleSignup },
       [req.body.email]
     )
     return { runId: run.runId }
   })

   await app.listen({ port: Number(process.env.PORT) })
   ```
6. Set environment variables (same set as Next.js / generic Node).
7. Run the build before each start (or wire it into your build script):
   ```bash
   npx workflow build --target standalone && node server.js
   ```

The plugin:
- Mounts `flow`, `step`, `webhook` handlers under `/.well-known/workflow/v1/`.
- Decorates the Fastify instance with `app.workflows` (name → workflowId map).
- Calls `world.start()` on boot to register the queue handler.

---

## Trigger a Workflow

When the user asks how to start a workflow run from application code:

1. Import the trigger API from the Vercel SDK, not from `@platformatic/world`:
   ```ts
   import { start } from 'workflow/api'
   ```
2. Two ways to identify the workflow:
   - **By imported reference** (works inside files the SDK transforms — typically
     Next.js routes/actions):
     ```ts
     import { handleSignup } from '@/workflows/signup'
     const run = await start(handleSignup, [email])
     ```
   - **By workflowId string** (works anywhere — required outside transformed
     files, e.g. plain Fastify route handlers):
     ```ts
     const run = await start({ workflowId: app.workflows.handleSignup }, [email])
     ```
3. To resume a hook (webhook or human-in-the-loop):
   ```ts
   import { resumeHook } from 'workflow/api'
   await resumeHook(token, payload)
   ```
4. Tell the user the trigger call is just an HTTP POST to the Workflow
   Service — no special transport is required in the calling process.

---

## Standalone Build Workflow

The Vercel SDK's `workflow build --target standalone` produces self-contained
bundles for hosts that don't have a built-in SDK transform (e.g. Fastify,
plain Node, scripts).

1. The build reads `'use workflow'` / `'use step'` files from your source tree.
2. It writes to `<buildDir>/.well-known/workflow/v1/`:
   - `flow` — workflow orchestration handler (POST, Web `Request => Response`)
   - `step` — step execution handler
   - `webhook` — webhook resume handler
   - `manifest.json` — workflow/step ids and graph
3. Extensions vary by SDK version: v5 emits `.mjs`; v4 emits `.js` (flow/step
   CommonJS, webhook ESM). `@platformatic/workflow-fastify` handles both
   transparently.
4. Re-run the build whenever workflow source files change. In CI, run it as
   a pre-build step before the host app is packaged.
5. Don't commit `.well-known/workflow/v1/` — add it to `.gitignore` alongside
   `dist/`.

```bash
npx workflow build --target standalone
```

Inside Next.js you do **not** need this — the Next plugin transforms
workflows inline. The standalone build is for non-Next hosts.

---

## Environment Configuration

The SDK and `@platformatic/world` read these environment variables:

| Variable | Required | Default | Description |
|---|---|---|---|
| `WORKFLOW_TARGET_WORLD` | Yes | — | Set to `@platformatic/world` to select this adapter |
| `PLT_WORLD_SERVICE_URL` | Yes | — | Workflow Service base URL (e.g. `http://localhost:3042`) |
| `PLT_WORLD_APP_ID` | No | `package.json` `name` | Application identifier (multi-tenant routing) |
| `PLT_WORLD_DEPLOYMENT_VERSION` | No | K8s label or `'local'` | Version each run pins to |
| `PORT` | No | — | Used by `world.start()` to register the callback URL locally |

In K8s, `PLT_WORLD_DEPLOYMENT_VERSION` is auto-detected from the pod's
`plt.dev/version` label via the K8s API — usually leave it unset there.

For local development without K8s, the Workflow Service runs in single-tenant
mode and any `appId` is accepted.

---

## Status Check

When user runs `/workflow status`:

1. **Node.js Version**
   ```bash
   node --version
   ```
   Check if >= v22.19.0.

2. **Dependencies present**
   ```bash
   node -e "require('workflow/package.json')"
   node -e "require('@platformatic/world/package.json')"
   ```

3. **Required env vars set**
   ```bash
   node -e "console.log({world:process.env.WORKFLOW_TARGET_WORLD, url:process.env.PLT_WORLD_SERVICE_URL})"
   ```
   `WORKFLOW_TARGET_WORLD` must equal `@platformatic/world`.
   `PLT_WORLD_SERVICE_URL` must be a reachable URL.

4. **Service reachable**
   ```bash
   curl -fsS "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/runs" >/dev/null && echo OK
   ```

5. **For Next.js**: `instrumentation.ts` exists at the project root and
   calls `world.start()` inside `register()`.

6. **For Fastify with plugin**: `.well-known/workflow/v1/manifest.json` exists
   (i.e. the standalone build has been run).

Report findings in a clear format:

```
Workflow SDK Configuration Status
=================================
Node.js Version:             vX.X.X [OK/UPGRADE NEEDED]
workflow installed:          [vX.X.X/Missing]
@platformatic/world:         [vX.X.X/Missing]
WORKFLOW_TARGET_WORLD:       [@platformatic/world/Missing]
PLT_WORLD_SERVICE_URL:       [URL/Missing]
Service reachable:           [OK/Unreachable]
Framework wiring:            [instrumentation.ts/Fastify plugin/manual]
Standalone build:            [Present/Not built/N/A]
```

---

## Important Notes

- **`@platformatic/world` is a runtime drop-in for `@workflow/world`.** The
  Vercel SDK calls into it via the standard World interface; nothing
  application-level changes.
- **`world.start()` must run exactly once per process before accepting
  traffic.** Calling it twice is harmless (idempotent) but wastes a call.
  Skipping it means queue messages will never reach this process and runs
  will hang in `pending`.
- **Under K8s + ICC**, `world.start()` is a no-op — ICC registers handlers
  centrally via the control plane.
- **Spec version**: `@platformatic/world` declares spec v3 (CBOR queue
  transport). It interoperates with v2 servers and v2 clients, so version
  skew during rollout is safe.
- **SDK compatibility**: works at runtime against both `workflow@4.2.x`
  (stable) and `workflow@5.0.0-beta.x` (next). The world is typed against
  `@workflow/world@4.1.1` but exposes the v5 `streams.*` namespace at
  runtime alongside the v4 flat methods.

## Troubleshooting

For common issues (runs stuck in `pending`, `world.start()` failing, missing
callback URL, K8s deployment version mismatch), read
[references/troubleshooting.md](references/troubleshooting.md).
