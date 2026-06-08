# Authoring Workflows with the Vercel Workflow SDK

This reference covers the application-author surface of the SDK: how to write
workflows and steps, configure retries, suspend on hooks and sleeps, stream
partial output, and trigger runs.

It is framework-agnostic. The same authoring rules apply whether you host the
SDK with Next.js, the `@platformatic/workflow-fastify` plugin, or a plain
Node server using the standalone build.

For configuration of the world adapter, see [world.md](world.md).

---

## Core Model

A **workflow** is a durable async function. The SDK records its progress so
that crashes, restarts, and timeouts cannot lose work or duplicate side
effects.

A **step** is the unit of side effect. Each step runs at most once per
attempt and its return value is persisted, so when the workflow is replayed
the step is skipped and its previous return value is used instead.

Hard rule: **all side effects must live inside steps.** Workflow code that
isn't a step is replayed on every restart and must be deterministic.

```ts
// workflows/signup.ts
'use workflow'

import { sendEmail } from './steps/email'
import { provisionUser } from './steps/users'

export async function handleSignup (email: string) {
  const user = await provisionUser(email)   // step: writes to DB
  await sendEmail(email, 'welcome')         // step: calls SMTP
  return { userId: user.id }                // workflow: pure return
}
```

```ts
// workflows/steps/email.ts
'use step'

export async function sendEmail (to: string, template: string) {
  await fetch('https://smtp.example.com/send', {
    method: 'POST',
    body: JSON.stringify({ to, template }),
  })
}
```

---

## Directives

Every workflow source file starts with one of two string directives at the
very top:

| Directive | File contains | Rule |
|---|---|---|
| `'use workflow'` | One or more workflow functions | The body must be deterministic. Side effects only via steps, hooks, sleeps, and streams. |
| `'use step'` | One or more step functions | Side effects allowed. Return values must be JSON-serializable (plus `Uint8Array` over CBOR). |

A single file is one mode only — don't mix workflows and steps in the same
file.

The SDK's build transform inspects these directives. Without them, the file
is treated as ordinary code and won't be reachable as a workflow/step.

---

## Steps

### Defining a step

```ts
// workflows/steps/charge.ts
'use step'

export async function chargeCard (customerId: string, amount: number) {
  const res = await fetch(`https://billing.example/charge`, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ customerId, amount }),
  })
  if (!res.ok) throw new Error(`charge failed: ${res.status}`)
  return await res.json()
}
```

### Retries

Steps are retried automatically on thrown errors. Configure retries with
`step()` from `workflow/api`:

```ts
'use step'
import { step } from 'workflow/api'

export const chargeCard = step(
  async (customerId: string, amount: number) => {
    // ...
  },
  {
    maxAttempts: 5,
    initialDelay: 1000,
    maxDelay: 30_000,
    backoffFactor: 2,
  }
)
```

To opt a step **out** of retries, set `maxAttempts: 1`.

### Step idempotency

Because a step may be retried, its side effect must be idempotent or guarded
by an idempotency key. For HTTP calls, pass an `Idempotency-Key` header. For
DB writes, use unique constraints + `ON CONFLICT DO NOTHING` or upsert.

### Return values

Step return values are persisted to the Workflow Service. Use JSON-compatible
shapes (plus `Uint8Array` over CBOR). Don't return class instances,
non-plain objects, functions, or anything with cycles.

---

## Hooks (human-in-the-loop / webhooks)

A hook suspends the workflow until an external caller resumes it.

### Inside the workflow

```ts
'use workflow'
import { hooks } from 'workflow/api'

export async function approveExpense (expenseId: string) {
  const token = hooks.token('manager-approval')
  // Send the token to the approver out of band — e.g. email a link
  await sendApprovalEmail(expenseId, token)

  // Suspend until someone calls resumeHook(token, ...). Returns whatever
  // payload they passed.
  const { approved, comment } = await hooks.wait(token)

  if (!approved) throw new Error(`rejected: ${comment}`)
  await markApproved(expenseId)
}
```

### Resuming from application code

From any host that has the SDK installed:

```ts
import { resumeHook } from 'workflow/api'

await resumeHook(token, { approved: true, comment: 'looks good' })
```

The token is opaque — store it wherever you need (DB row, email link, JWT).

### Webhook hooks

A webhook hook resumes when an HTTP request hits a known URL. The Workflow
Service forwards the request body to the suspended workflow. Configure the
hook with `isWebhook: true`:

```ts
const token = hooks.token('payment-webhook', { isWebhook: true })
const event = await hooks.wait(token)
```

The Workflow Service exposes a URL containing the token. Configure your
external system (Stripe, GitHub, etc.) to POST to that URL.

---

## Sleeps

Pause a workflow for a duration that survives restarts:

```ts
'use workflow'
import { sleep } from 'workflow/api'

export async function dailyReminder (userId: string) {
  while (true) {
    await sleep('24h')
    await sendReminder(userId)
  }
}
```

Durations accept `'30s'`, `'5m'`, `'24h'`, `'7d'`, or a number of
milliseconds. The Workflow Service schedules a queue message to wake the
run; the process can exit during the sleep and a different pod can resume it.

---

## Streams

Streams emit partial results from a step while it runs — useful for LLM
token-by-token output, progress, or long-running file uploads.

```ts
'use step'
import { stream } from 'workflow/api'

export async function generateAnswer (prompt: string) {
  const out = stream('answer')
  for await (const token of llmTokens(prompt)) {
    await out.append(token)
  }
  await out.close()
  return { ok: true }
}
```

The application can read the stream by name from the run:

```ts
import { getStreamChunks } from 'workflow/api'

for await (const chunk of getStreamChunks(runId, 'answer')) {
  res.write(chunk)
}
```

CBOR transport preserves `Uint8Array` chunks natively, so binary streams
don't need base64 wrapping when targeting `@platformatic/world`.

---

## Triggering Runs

Import from `workflow/api`:

```ts
import { start, getRun, cancelRun } from 'workflow/api'
```

### Start

```ts
// By imported reference — works inside SDK-transformed files
import { handleSignup } from '@/workflows/signup'
const run = await start(handleSignup, [email])
```

```ts
// By workflowId — works anywhere
const run = await start(
  { workflowId: 'workflow//./workflows/signup//handleSignup' },
  [email]
)
```

`start()` returns `{ runId, ... }`. The trigger call is a plain HTTP POST to
the Workflow Service — no special transport in the calling process.

### Wait for completion

```ts
const result = await start(handleSignup, [email]).result
```

Or poll later:

```ts
const run = await getRun(runId)
if (run.status === 'succeeded') return run.result
```

### Cancel

```ts
await cancelRun(runId)
```

Cancellation propagates: the next step boundary or sleep wake observes the
cancellation and the run terminates.

### Replay

Replay a completed run with the same input on the deployment version that
originally executed it (useful for debugging):

```bash
curl -X POST "$PLT_WORLD_SERVICE_URL/api/v1/apps/default/runs/$RUN_ID/replay"
```

---

## Determinism Rules (workflow code)

The body of a `'use workflow'` function is replayed from scratch every time
the run is resumed. To keep replay safe:

- **Don't read time, randomness, or env directly.** Wrap them in steps:
  `await currentTime()`, `await randomId()`. Otherwise replay sees a
  different value than the first run.
- **Don't perform I/O outside steps.** No `fetch`, no DB calls, no file
  system access in workflow code.
- **Don't mutate shared module state.** Locals are fine; globals are not.
- **Branch on step return values, not on external state.** A workflow that
  asks "is today Friday?" inline will diverge on replay; one that calls
  `await isFriday()` (a step) records the answer.
- **Loops are fine** as long as the iteration count and break conditions
  derive from step outputs.

The SDK's transform will flag many violations at build time, but not all —
think of workflow code as pure orchestration.

---

## Errors

- A step throwing past its retry budget fails the run with that error.
- A workflow throwing fails the run.
- Catch errors inside the workflow if you want recovery logic:
  ```ts
  try {
    await chargeCard(customerId, amount)
  } catch (err) {
    await refundOrder(orderId)
    throw err
  }
  ```
- The error message is persisted. Don't put secrets in error messages.

---

## File Layout

There is no enforced layout, but a common convention:

```
src/
├── workflows/
│   ├── signup.ts          # 'use workflow'
│   ├── checkout.ts        # 'use workflow'
│   └── steps/
│       ├── email.ts       # 'use step'
│       ├── billing.ts     # 'use step'
│       └── users.ts       # 'use step'
└── server.ts              # registers world / mounts handlers
```

The SDK build picks up directives anywhere — the layout above is convention,
not requirement.

---

## Build Targets

| Host | Build | Notes |
|---|---|---|
| Next.js | None — the Next.js plugin transforms on the fly | Just author files and run `next dev` / `next build`. |
| Fastify with `@platformatic/workflow-fastify` | `npx workflow build --target standalone` | Emits `.well-known/workflow/v1/{flow,step,webhook,manifest.json}`. |
| Other Node hosts | `npx workflow build --target standalone` | Mount the emitted handlers manually. |

Re-run the standalone build whenever a workflow or step source file changes.

---

## Resources

- [Vercel Workflow DevKit docs](https://useworkflow.dev/)
- [Workflow SDK GitHub](https://github.com/vercel/workflow)
- [world.md](world.md) — `@platformatic/world` adapter reference
- [fastify.md](fastify.md) — Fastify plugin reference
- [troubleshooting.md](troubleshooting.md) — common pitfalls
