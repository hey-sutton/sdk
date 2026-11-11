# Inbound webhooks

Backend Workers can receive **third-party webhooks** (WooCommerce, GitHub, payment
providers, …) on public URLs:

```
POST https://{app-id}.<apps domain>/webhooks/{path}
```

Each `path` must be declared in `manifest.json` — anything undeclared 404s at the platform
edge before reaching your Worker. Frontend-only apps cannot receive webhooks.

Inbound webhooks are **published-only**. A draft Host returns `404` and never delivers the request
to the draft Worker. Use signed unit/local fixtures for draft development, or a deliberate
test strategy against the published endpoint; never point a production provider at a draft URL.

## Declare paths in the manifest

```json
{
  "permissions": [],
  "backend": {
    "kind": "cloudflare-worker",
    "webhooks": [{ "path": "note-created" }]
  }
}
```

Rules (publish-validated):

- `path`: lowercase `[a-z0-9-]`, max 32 chars, unique within the app; single segment only
- At most **10** webhooks per app
- POST only — other methods are refused at the platform edge

## Handle deliveries with `createHandler({ webhooks })`

```js
import { createHandler, requireEnv, verifyWebhookSignature } from '@heysutton/sdk-server';

export default createHandler({
  webhooks: {
    'note-created': async ({ headers, body, env }) => {
      const valid = await verifyWebhookSignature({
        secret: requireEnv({ env, key: 'WEBHOOK_SECRET' }),
        body,
        signature: headers['x-webhook-signature'] ?? '',
      });
      if (!valid) return { status: 401 };

      const payload = JSON.parse(new TextDecoder().decode(body));
      // …act on it, or enqueue for heavier work (see below)…
      return { status: 200 };
    },
  },
  fetch: async ({ request, env }) => {
    /* normal routes */
  },
});
```

- Handler args: `{ path, method, query, headers, body, env, ctx }` — `body` is a
  `Uint8Array` of the **exact bytes** the provider sent, `headers` are lower-cased,
  `query` is the raw query string (no leading `?`)
- Return `{ status, headers?, body? }` — this goes back to the provider verbatim, so
  provider challenge/echo handshakes work; non-2xx statuses generally make providers retry
- Finish within the 10s delivery budget

## ALWAYS verify the signature

The endpoint is public and unauthenticated — the provider's signature is the **only**
authentication, and your handler is the only place it can be checked (only you hold the
secret). Verify over the raw `body` bytes before acting on anything; on failure return
401 and do nothing else. Never `JSON.parse` first and re-serialise for verification — the
platform delivers the exact bytes so HMAC checks work.

`verifyWebhookSignature({ secret, body, signature, algorithm?, encoding? })` is a
timing-safe WebCrypto HMAC check:

- WooCommerce: `encoding: 'base64'` over the body, header `x-wc-webhook-signature`
- GitHub: `encoding: 'hex'`, header `x-hub-signature-256` — strip its `sha256=` prefix
- Stripe-style `timestamp.payload` schemes: build the signed string yourself and pass it
  as `body`

Store the secret via the app's Secrets tab (config vars) and read it with `requireEnv`.

## Heavier work: verify, then enqueue

Do not do slow work inside the delivery — verify, enqueue a job, and return 200:

```js
const { runId } = await appJobs({ env }).submit({
  name: 'process-order',
  payload: { orderId },
  idempotencyKey: headers['x-delivery-id'], // provider redeliveries collapse to one run
});
return { status: 200 };
```

Enqueue **only after** verification passes — the job queue is shared with your schedules,
and unverified traffic must never consume it. Use the provider's delivery id as the
`idempotencyKey` so retries don't double-process.

## Local dev

There is no local webhook ingress: `triggerWebhook` is never invoked under `wrangler dev`.
Keep handlers as plain exported functions (see
`src/backend/webhooks.js` in the `full-featured` template) so you can unit-test them directly —
`verifyWebhookSignature` runs anywhere WebCrypto exists.

Working example: the `full-featured` template — `manifest.json` (webhook declaration),
`src/backend/webhooks.js` (signed note-created handler).
