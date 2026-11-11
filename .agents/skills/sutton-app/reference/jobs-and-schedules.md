# Jobs & schedules

Backend Workers get background execution through the **platform scheduler**: recurring
**schedules** declared in `manifest.json`, and on-demand **jobs** submitted from your own
Worker code. Both run the same way — the platform calls into your Worker over RPC and
records the run. Frontend-only apps have none of this.

## Declare schedules in the manifest

```json
{
  "permissions": [],
  "backend": {
    "kind": "cloudflare-worker",
    "bindings": { "MY_APP_DB": "database" },
    "schedules": [
      { "name": "prune-notes", "everySeconds": 3600, "payload": { "maxAgeDays": 30 } }
    ]
  }
}
```

Rules (publish-validated):

- `name`: lowercase `[a-z0-9-]`, max 32 chars, unique within the app
- `everySeconds`: integer, **300–86400** (5 minutes to 1 day)
- `payload`: optional JSON, max 4KB — passed to every tick
- At most **5** schedules per app
- **Schedules run on the published app only.** A published deployment replaces the published schedule set wholesale; a
  published deployment with no `schedules` clears it. Draft does not register cron.

## Draft behavior

**Schedules run on the published app only.** Draft Workers receive the jobs binding under the draft app
identity, but the platform does not register the manifest's recurring schedules for draft.
Draft cron, inbound webhooks, and other timer/webhook side effects stay off. This prevents
a draft from running a second copy of production cron work. An on-demand job submitted by
draft code is scoped to the draft job namespace, but it still sees published env/secrets and
is a real background run.

Promotion registers the manifest schedules against the published Worker. Do not use a successful
draft to claim that timer-driven behavior has been tested in production conditions; test the
handler directly and verify the schedule after promotion.

## Handle runs with `createHandler({ jobs })`

```js
import { createHandler, requireEnv } from '@heysutton/sdk-server';

export default createHandler({
  jobs: {
    'prune-notes': async ({ payload, env }) => {
      const db = requireEnv({ env, key: 'MY_APP_DB' });
      const maxAgeDays = Number(payload?.maxAgeDays) || 30;
      const result = await db
        .prepare(`DELETE FROM notes WHERE created_at < datetime('now', ?) RETURNING id`)
        .bind(`-${maxAgeDays} days`)
        .all();
      return { deleted: result.results?.length ?? 0 };
    },
  },
  fetch: async ({ request, env }) => {
    /* normal routes */
  },
});
```

- One handler per name — schedules AND submitted jobs share the map
- Handler args: `{ name, kind: 'job' | 'schedule', runId, attempt, payload, deadlineMs, env, ctx }`
- The **return value** (JSON-serialisable) is stored as the run's `result`
- A **throw** fails the attempt; the platform retries (up to 3 attempts per run)
- Finish within `deadlineMs` (30s budget per invocation) — long work must be chunked
- A schedule that fails ~10 consecutive runs is **auto-paused**; fix the handler and
  republish to resume

Under the hood the scheduler calls a `triggerJob(invocation)` method on your Worker's
default entrypoint over RPC — `createHandler` emits the WorkerEntrypoint class with that
method wired to your `jobs` map. Only the platform can make that call (it is unreachable
over HTTP), so it needs no auth of its own. Consequence: a backend Worker's default
export must be the `createHandler(...)` class (or your own WorkerEntrypoint with a
`triggerJob` method) — a plain `{ fetch }` object cannot receive jobs and every run fails
`NO_HANDLER`.

## Submit and inspect jobs: `appJobs(env)`

Every published backend Worker gets a `DIGIT_JOBS` binding (platform-injected — never
declare it; the `DIGIT_` prefix is reserved). `appJobs({ env })` returns it typed:

```js
import { appJobs, ok } from '@heysutton/sdk-server';

// In a route: queue work and return immediately.
const { runId } = await appJobs({ env }).submit({
  name: 'note-stats',
  payload: { requestedBy: 'ui' },
  idempotencyKey: 'stats-2026-08-06', // optional: same key → same run
});
return ok({ data: { runId }, status: 202 });
```

Surface (all scoped to this app only):

- `submit({ name, payload?, idempotencyKey? })` → `{ runId }` — payload max 32KB
- `get(runId)` → run or `null`
- `list({ limit? })` → recent runs, newest first (ring of 50; **successful schedule ticks
  are not kept** — the ring is for jobs and failures)
- `cancel(runId)` → `{ cancelled }` — `false` if already finished; an in-flight
  invocation is not interrupted, but its outcome is discarded
- `schedules()` → declared schedules with `nextDueAt`, `autoPaused`, `consecutiveFailures`

Run shape: `{ runId, name, kind, status, createdAt, startedAt, endedAt, result, error }`;
`status` is `queued | running | succeeded | failed | cancelled`; `error` is a
`CODE: detail` string — match on the prefix (`HANDLER_ERROR`, `INVOCATION_FAILED`, …).

## Patterns

- **Fire-and-forget from the UI**: frontend calls a normal route → route `submit()`s →
  return the `runId` with 202 → UI polls a route that calls `get(runId)`
- **Recurring maintenance**: manifest schedule + handler (pruning, syncing, digests)
- **Don't**: busy-wait on a run inside a route (poll from the frontend instead), or use
  schedules under 300s as a poor man's queue — submit jobs when the work actually exists

## Local dev

There is no local scheduler: `appJobs` throws MISSING_CONFIG under `wrangler dev` and
`triggerJob` is never invoked. Keep handlers as plain exported functions (see
`src/backend/jobs.js` in the `full-featured` template) so you can unit-test them directly.

Working example: the `full-featured` template — `manifest.json` (schedule),
`src/backend/jobs.js` (handlers), `src/backend/index.js` (submit/list routes),
`src/frontend/panels/JobsPanel.tsx` (UI consuming them).
