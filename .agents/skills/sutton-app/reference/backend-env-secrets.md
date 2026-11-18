# Backend env vars and secrets

Env vars and secrets are configured on the **app record in Digit** (UI only — there are no
MCP tools to manage them). They are **injected into the app's Cloudflare Worker**, not
into the frontend bundle.

## Key rules

- Keys: `UPPER_SNAKE_CASE`, pattern `^[A-Z][A-Z0-9_]{0,63}$`
- Keys must be unique across env vars **and** secrets for an app
- Values max 4096 bytes
- Secrets are write-only in the API (owners see keys, not values)
- Frontend must never hard-code secret values

## Draft: published env and published secrets

Draft has a separate Worker and separate app-owned storage (D1/R2), but it uses the
**same env vars and secrets as the published app**.

Sutton Settings that edit env vars or secrets update **the published app only**. A change made while
reviewing a draft still writes the published record, and both the draft Worker and the published
Worker then receive those values.

**Promote does not wipe published env/secrets.** Draft shares published env and secrets; Sutton
Settings write the published app only. Nothing is copied from draft onto the published app for env or secrets —
draft already shares them, and promote is not a config copy.

Because a draft therefore always holds published credentials, guard or disable external writes,
email, payments, and other irreversible side effects. Do not infer the channel from internal
`X-Digit-*` headers: there is not yet a documented app-facing `isPreview()` helper. A supported
runtime channel signal would be a platform/SDK change, not an app convention.

## Worker access

Prefer `@heysutton/sdk-server`: wrap with `createHandler`, read bindings with `requireEnv`
(or `optionalEnv` when absence is a valid branch). Missing required keys become structured
`{ ok: false, error }` responses — do not read `env.KEY` ad hoc and return plain text.

```js
import { AppErrorCode } from '@heysutton/sdk';
import {
  backendPath,
  createHandler,
  err,
  ok,
  requireEnv,
} from '@heysutton/sdk-server';

export default createHandler({
  fetch: async ({ request, env }) => {
    const path = backendPath(request);

    if (request.method === 'GET' && path === '/external-status') {
      const apiBase = requireEnv({ env, key: 'API_BASE_URL' });
      const apiKey = requireEnv({ env, key: 'THIRD_PARTY_API_KEY' });
      const response = await fetch(`${apiBase.replace(/\/$/, '')}/status`, {
        headers: { Authorization: `Bearer ${apiKey}` },
      });
      if (!response.ok) {
        return err({
          code: AppErrorCode.UPSTREAM_ERROR,
          message: `Upstream failed (HTTP ${response.status}).`,
          status: 502,
        });
      }
      return ok({ data: await response.json() });
    }

    return err({ code: AppErrorCode.NOT_FOUND, message: 'Not found.', status: 404 });
  },
});
```

D1 databases and R2 buckets (when declared in the manifest) appear under the binding
names you chose — also via `requireEnv`:

```js
const db = requireEnv({ env, key: 'MY_APP_DB' });
await db.prepare('SELECT 1').first();

const assets = requireEnv({ env, key: 'MY_APP_ASSETS' });
await assets.put('reports/latest.csv', csvText);
const object = await assets.get('reports/latest.csv');
```

Never put secret values or raw upstream bodies into `error.message` / success `data`.

## Frontend pattern

Frontend reads env-backed data only through the backend proxy, via hooks:

```ts
import { AppErrorAlert, useBackendQuery } from '@heysutton/sdk-react';

const { data, error, loading, refetch } = useBackendQuery<{ authenticated: boolean }>({
  path: '/external-status',
});
```

Do not hand-roll `/proxy/backend` fetches without `X-Digit-Proxy-Client` (the hooks set it).

## When you need a backend

| Situation | Backend required? |
| --- | --- |
| Pure UI | No |
| Sutton GraphQL via `useApiQuery` / `AppProxy` | No |
| Read non-secret config from Sutton app settings | Yes |
| Call third-party APIs with secrets | Yes |
| App-own persistence (D1) | Yes (`backend.d1`) |
| App-own file/blob storage (R2) | Yes (a `"bucket"` binding) |

## Setup for users

1. Create the app in Sutton
2. Set env vars / secrets in Sutton Settings (published values; draft uses them too)
3. Publish a bundle whose manifest declares `backend.kind: "cloudflare-worker"`
4. Ship `backend/index.js` that reads those keys via `requireEnv` inside `createHandler`
