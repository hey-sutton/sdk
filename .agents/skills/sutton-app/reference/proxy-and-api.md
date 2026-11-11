# Proxy and Digit API

Apps run on a per-app origin. The only egress the frontend CSP allows is same-origin, so
Digit data and app backends are reached through Digit-hosted proxies.

Prefer the React hooks from `@heysutton/sdk-react` — they wrap the harness client and
normalize errors for `AppErrorAlert`. The package’s public data API is **hooks only**
(imperative fetch helpers stay private).

## Look up the GraphQL schema (required)

Digit MCP is required. When writing or changing Digit GraphQL operations, use these MCP
**resources** — do **not** invent field or type names, and do **not** load the full schema
into context:

1. `graphql-schema://index` — compact list of queries, mutations, and type names
2. `graphql-schema://type/{TypeName}` — that type’s SDL (e.g. `graphql-schema://type/Item`)
3. `graphql-schema://search/{query}` — search (optional `?limit=1-50`, default 25), then
   fetch matching type URIs

This is the public API schema (same surface API tokens can introspect). Pair schema lookup
with MCP **`appPermissions`** so `manifest.permissions` covers the operations you call —
see [permissions.md](permissions.md).

## Hooks

```ts
import {
  AppErrorAlert,
  useApiQuery,
  useBackendQuery,
  useBackendMutation,
} from '@heysutton/sdk-react';

const { data, error, loading, refetch } = useApiQuery({
  query: `
    query Items($connection: ConnectionInput) {
      items(connection: $connection) {
        nodes { id name sku }
      }
    }
  `,
  variables: { connection: { first: 10 } },
});

if (error) {
  // <AppErrorAlert error={error} onRetry={() => void refetch()} />
}

const notes = useBackendQuery<{ notes: Note[] }>({ path: '/notes' });
const [mutateNote] = useBackendMutation();
await mutateNote({ path: '/notes', method: 'POST', body: { title: 'Hi' } });
```

Confirm root fields and selection sets against `graphql-schema://…` before shipping.
`AppHost` and its types (`AppHostSettings`) are exported from `@heysutton/sdk-react`. Prefer the
hooks over calling `window.AppProxy` yourself.

## Sort, filter, and pagination

Prefer API (GraphQL or backend) inputs for sorting, filtering, and paging when the schema
or route supports them. Do not load an entire collection into the browser to sort/filter
locally if the field accepts those args — look them up on `graphql-schema://type/…`.

When the UI is a table (or any multi-row list that can grow), paginate it: pass
`connection` / page args (e.g. `first` + `after`, or page size + cursor) and wire
next/previous (or equivalent) controls. Unbounded `nodes` dumps are not acceptable for
tables.

Fetch related records in one request. When a screen needs details for many records (every
item on an order or project, say), pass their ids to the list query's id filter, e.g.
`items(itemIds: […])`, instead of calling `item` once per id. The index shows a list query's
id filter as `items(itemIds?)`, and each single-record query names the list query to use.
Single-record queries are for one record's full details. See [Request limits](#request-limits).

## Digit GraphQL API

`useApiQuery` / `useApiMutation` POST `/proxy/digit` with
`credentials: 'include'` and `X-Digit-Proxy-Client: 1`.

- Credentials stay server-side (HttpOnly session cookie + scoped token in Redis)
- Your query must be covered by `manifest.permissions` ∩ the user's live permissions
- Do **not** call Digit GraphQL with a bearer token from app JS
- Do **not** invent a different proxy URL

On a draft Host, Sutton GraphQL reads and writes use the viewer's permissions, still
limited to `manifest.permissions`. App-owned D1/R2 writes stay in draft resources.

## App backend

When `manifest.backend` is set, `useBackendQuery` / `useBackendMutation` hit
`/proxy/backend/...`. Rules:

- Always go through these helpers so `X-Digit-Proxy-Client` is set
- Paths starting with `/__` are reserved and refused
- The platform sets `X-Digit-App-Id` from the Host header — the browser cannot spoof another app's worker
- Pair with `@heysutton/sdk-server` on the Worker (`createHandler`, `backendPath`, `ok` / `err`)

```js
import { AppErrorCode } from '@heysutton/sdk';
import { backendPath, createHandler, ok, err, requireEnv } from '@heysutton/sdk-server';

export default createHandler({
  fetch: async ({ request, env }) => {
    const path = backendPath(request);
    const { method } = request;

    if (method === 'GET' && path === '/greeting') {
      return ok({ data: { message: requireEnv({ env, key: 'WELCOME_MESSAGE' }) } });
    }
    if (method === 'PUT' && path.startsWith('/notes/')) {
      return ok({ data: { id: path.slice('/notes/'.length) } });
    }

    return err({ code: AppErrorCode.NOT_FOUND, message: 'Not found.', status: 404 });
  },
});
```

Import codes / validation from `@heysutton/sdk`; Worker Response helpers from
`@heysutton/sdk-server` (no re-exports between packages).

## Request limits

Every call from the app goes through Sutton's proxy, which enforces these limits:

| Limit | Value | When exceeded |
| --- | --- | --- |
| Request body (Sutton API and backend) | 250KB of JSON | `413 PAYLOAD_TOO_LARGE` (currently a 413 HTML page with no error code) |
| Backend response size | 5MB | `502 BACKEND_RESPONSE_TOO_LARGE` |
| Backend response time | 10s | `502 BACKEND_UNAVAILABLE` |
| Requests per user session | 60 per minute | `429 RATE_LIMITED` |
| Requests per app, all users | 600 per minute | `429 RATE_LIMITED` |

The per-session budget is shared by every Sutton API and backend call the app makes for that
user, including the Sutton API calls the backend makes on the user's behalf. Opening a screen
that fires 15 queries uses a quarter of it.

Design for these limits from the start:

- Keep each user action to a handful of requests. Never split one save into many small
  requests, and never add throttle queues, request spacing or retry loops to get under a limit.
  If you are tempted to, the data model is wrong: change it.
- Store files (images, PDFs, logos) as objects in a `bucket` binding (see
  [manifest.md](manifest.md)), one object per file, keyed by a hash of the content so a file
  used by many records is stored once. Records keep the key, not the file. Never put
  `data:` URLs or other file contents in D1 rows or in records that list routes return.
- Shrink images in the browser before uploading (draw to a canvas at the size you need and
  export WebP or JPEG). Backend requests are JSON only, so the file travels base64-encoded,
  which adds about a third: keep each exported image under about 180KB so the upload fits in
  one request.
- List routes return small summaries. Fetch a large record or file one at a time, by id, when
  it is needed.
- Load data when a screen opens or the user acts. Don't poll on a timer, and don't refetch
  whole collections after every save.

## Sutton API from the backend

The backend `fetch` handler gets an `api` argument for Sutton GraphQL. Never ask the user for an API token,
org id or anything similar: the platform brokers every call.

```js
import { createHandler, ok } from '@heysutton/sdk-server';

export default createHandler({
  fetch: async ({ api }) => {
    const result = await api.query(`{ purchaseOrders(connection: { first: 20 }) { nodes { id } } }`);
    return ok({ data: result.data });
  },
});
```

- Scope is `manifest.permissions`, exactly as for `/proxy/digit` — declare what the backend needs
- It acts as **the user whose request it is serving**. Jobs, schedules and webhooks have no
  Sutton API for now: do Sutton work in `fetch`, or pass the data a job needs in its payload
- Always use the `api` argument — never `env.__API`
- Results come back as GraphQL `{ data, errors }` — check `errors`

## Host display settings

```ts
import { AppHost } from '@heysutton/sdk-react';

AppHost.getSettings(); // AppHostSettings | null
AppHost.onSettingsChange((settings) => { /* ... */ });
```

`data-theme` and `lang` are also set on `<html>`. Apps using `@heysutton/sdk-react` get
light/dark sync automatically via `ThemeProvider` — see [theming.md](theming.md).
