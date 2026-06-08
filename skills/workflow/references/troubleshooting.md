# Troubleshooting — Vercel Workflow SDK with `@platformatic/world`

Common failure modes and how to diagnose them.

---

## Runs stuck in `pending` forever

The Workflow Service accepted the run but no pod is processing it.

**Likely causes:**

1. **`world.start()` was never called.** The pod never registered as a
   handler, so queue messages have nowhere to go.
   - Next.js: check `instrumentation.ts` exists at the project root and
     calls `world.start()` inside `register()`.
   - Generic Node / Express / Koa: add an explicit `await world.start?.()`
     on boot before `app.listen()`.
   - Fastify with plugin: confirm the plugin is registered and `register`
     option isn't `false`.

2. **Wrong `PLT_WORLD_DEPLOYMENT_VERSION`.** Each run pins to the version
   that created it. If your pods report a different version than the one
   the run was created on, the message is held until a matching pod appears.
   - Local: leave the var unset (defaults to `'local'`).
   - K8s: confirm the pod has the `plt.dev/version` label and the K8s API
     is reachable from inside the pod.

3. **The Workflow Service can't reach the pod's callback URL.** The pod
   registered, but the URL is wrong (e.g. localhost when running in Docker).
   - Check the `POST /handlers` request body in service logs — the
     `endpoints.workflow` URL must be reachable from the service.
   - In Docker Compose, use the service name, not `localhost`.

**Diagnose:**

```bash
# List runs and their status
curl -fsS "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/runs" | jq '.data[] | {runId, status, workflowName}'

# List registered handlers
curl -fsS "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/handlers" | jq
```

---

## `Error: WORKFLOW_TARGET_WORLD is not set`

The Vercel SDK doesn't know which world to use.

Set:

```
WORKFLOW_TARGET_WORLD=@platformatic/world
PLT_WORLD_SERVICE_URL=http://localhost:3042
```

In Next.js, env vars in `.env.local` are picked up automatically. In other
hosts, make sure they're set before `process.env` is read by
`createWorld()`.

---

## `Error: workflow handler {name} not found in {dir}` (Fastify plugin)

The standalone build hasn't been run, or `buildDir` points to the wrong
place.

```bash
npx workflow build --target standalone
ls .well-known/workflow/v1
# expect: flow.{mjs,js} step.{mjs,js} webhook.{mjs,js} manifest.json
```

If you keep the build elsewhere (e.g. `dist/`), pass it explicitly:

```ts
await app.register(workflowFastify, { buildDir: 'dist' })
```

---

## `CorruptedEventLogError` from the SDK

The SDK's events consumer rejected a replay because it saw an unexpected
event ordering. Usually one of:

1. **Duplicate event from a retry.** The Workflow Service receives two
   identical events for the same correlation_id under concurrency. The
   service serializes these via `SELECT ... FOR UPDATE` on the step row —
   if you see this error against a recent service version, file a bug with
   the failing run's events log:
   ```bash
   curl -fsS "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/runs/$RUN_ID/events" | jq
   ```

2. **Mixed deployment versions in flight.** If two pod versions claim the
   same `deploymentVersion`, both compete for the same run. Use distinct
   versions per deploy.

---

## `[workflow-sdk] Event cursor missing after initial load`

Harmless warning. The events list endpoint returns `cursor: null` on the
last page because PostgreSQL SERIAL ids are not commit-ordered under
concurrent inserts — incremental `WHERE id > cursor` could skip events that
commit late with smaller ids. The SDK falls back to a full reload, which is
correct behavior.

If you see this warning during normal operation, ignore it.

---

## Next.js: workflow files not transformed

Symptoms: `'use workflow'` files run as ordinary code, side effects fire
twice on replay, `start()` complains it can't resolve the workflow.

- Check the file starts with the directive on the **first non-comment
  line** as a quoted string literal: `'use workflow'`. Indented strings,
  template literals, or non-string-literal expressions are not recognized.
- Confirm you're on a supported `workflow` SDK version (4.2.x or 5.0.0-beta.x).
- For Next 16 with Turbopack: the SDK's dynamic require uses a
  `/* turbopackIgnore: true */` annotation introduced in `@workflow/core`
  beta.8. Older versions need to disable Turbopack or pin to a webpack
  build.

---

## Step throws but doesn't retry

- Confirm the step throws an `Error` (or subclass) — non-Error throws
  bypass retry logic.
- Confirm `maxAttempts` is set (default is the SDK default; check your
  `step()` config).
- A step marked `maxAttempts: 1` intentionally never retries.
- The Workflow Service's response code `425 Too Early` is the SDK's
  re-queue signal for `retry_after`; if you see 4xx other than 425, the
  retry path may be aborting early. Check the service response body.

---

## Hook never resumes

- Confirm `resumeHook(token, payload)` is being called with the **exact**
  token returned from `hooks.token(...)`. Tokens are opaque; URL-encoding
  twice or trimming whitespace breaks them.
- Check the hook still exists:
  ```bash
  curl -fsS "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/hooks/by-token/$TOKEN" | jq
  ```
- A hook on a completed/cancelled run is disposed and cannot be resumed.

---

## Local dev: port conflicts

`world.start()` registers the callback URL using `PORT`. If `PORT` is unset
or wrong, the service can't reach the pod.

```bash
PORT=3000 npx next start -p 3000
```

If you see `EADDRINUSE` after a previous run, the kernel hasn't released
the port yet (TIME_WAIT). Wait a few seconds or use a different port.

---

## More help

- Service logs: tail the Workflow Service for `[POLLER]` lines — they show
  every queue dispatch and the response from the pod.
- App logs: the SDK logs to `console` by default. Look for
  `[workflow-sdk]` prefixes for SDK-level messages.
- File an issue: https://github.com/platformatic/platformatic-world/issues
