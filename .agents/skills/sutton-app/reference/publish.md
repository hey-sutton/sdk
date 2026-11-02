# Deploy a Digit app

Requires the org `CUSTOM_APPS` feature flag and `publish:app` permission. If either is
missing, stop — there is no alternate publish path. Digit MCP is required for the standalone
upload workflow below.

Packing is local; the Digit platform performs the deployment. There is no local Digit runtime
draft because Workers, env/secrets, D1, and R2 are injected by the platform. The same
channel-neutral `app.zip` can be deployed to draft and later promoted to the published app.

## Choose the workflow

### Digit App Builder

The builder's `publishAppZip` tool owns upload, draft deployment, and polling:

```text
npm run pack -w apps/app
publishAppZip({ zipPath: "apps/app/app.zip" })
```

`publishAppZip` always deploys a draft. It waits for the draft deployment to reach a
terminal state and reports failure details. A successful result means **the published app is
unchanged**. The user promotes that exact draft build with the explicit Publish action in
the Sutton web app. The builder must not publish to the published app or describe the draft as
production.

### Standalone MCP

Standalone agents and scripts use the upload-link flow. Digit MCP is required for this
workflow.

```text
1. apps                     → find appId
2. generateAppUploadLink    → id, uploadUrl, uploadFields
3. HTTP POST zip to uploadUrl (multipart; NOT via MCP)
4. publishApp               → appId + appUploadLinkId + channel
5. appPublish               → poll until succeeded | failed
```

Call MCP `apps`. Match by `name`. Use the returned `id` as `appId`. If the app does not
exist, stop and ask the user to create it in Digit.

`publishApp` accepts `channel: "preview" | "live"`. Use `channel: "preview"` when testing
without changing production. If `channel` is omitted, it defaults to `live` for backwards
compatibility with existing callers. Promotion is currently an explicit Digit web action;
do not assume a draft is published just because its `appPublish` row succeeded.

## Prerequisites

1. The user has **created the app in Digit** (UI). Deployment never creates apps — there are
   no MCP tools to create, update, or delete apps. If the app does not exist, stop and ask
   the user to create it in Digit.
2. You know the app `id` — resolve it with MCP `apps`, or use the builder's request context.
   If the app does not exist, stop and ask the user to create it in Digit.
3. GraphQL operations were checked against `graphql-schema://…`, and
   `manifest.permissions` contains the `key` values returned by `appPermissions`.
4. `app.zip` is ready via `npm run pack` (`sutton studio pack` from `@heysutton/cli`).

## Standalone MCP upload

Call MCP `generateAppUploadLink` and save:

- `id` — both `appUploadLinkId` and the later `appPublishId`
- `uploadUrl`
- `uploadFields` — array of `{ key, value }`

MCP cannot carry binary bodies. POST multipart form-data with every `uploadFields` entry first,
then the zip as the final `file` field. The zip must be no larger than **10MB**. If the upload
cannot be completed, stop; do not call `publishApp` against an empty upload.

```bash
# Pseudocode — expand uploadFields into -F key=value pairs, then -F file=@app.zip
curl -X POST "$UPLOAD_URL" \
  -F "key=...from uploadFields..." \
  -F "Content-Type=...from uploadFields..." \
  # ...all other uploadFields...
  -F "file=@app.zip"
```

Upload **`app.zip` as produced**:

```text
app.zip
├── manifest.json
├── frontend/
│   ├── index.js
│   └── assets/          # self-hosted Inter; served at /app/assets/
├── backend/                  # when manifest.backend is set
│   ├── index.js
│   └── migrations/
└── project/                  # required — source, SPEC, vendored @heysutton/sdk*
```

Do not modify the zip after packing. Digit deploys `frontend/` and `backend/`; `project/`
must remain in the published zip so the app can be restored for a later iteration.

## Polling and failures

Call `appPublish` with `appId` and `appPublishId` (the upload-link `id`) until the state is
`succeeded` or `failed`. Intermediate states include `queued`, `validating`,
`deployingBackend`, and `publishingBundle`. During `deployingBackend`, Digit applies pending
migrations. They must be additive and forward-only — see [d1-migrations.md](d1-migrations.md).

For a draft, a succeeded row means the owner can open the draft environment; it does not
mean the published app changed. For a publish to the published app or a later promotion, the published app changes
only after the platform has completed that deployment.

Publish ships the reviewed build and applies pending migrations on the published app only — it is not a
config promote. **Promote does not wipe published env/secrets.** Draft shares published env and
secrets; Sutton Settings write the published app only. Nothing is copied from draft onto the published app for env
or secrets — draft already shares them, and promote is not a config copy.

Published D1 migrations are **fail-closed** on Time Travel. Sutton captures a D1 Time Travel
bookmark before migrating production; if bookmark capture fails, promotion fails and does
not migrate. Sutton does not auto-restore from that bookmark if a later step fails — traffic on
the published app keeps writing after the bookmark, so a restore would drop those rows. The bookmark
stays on the promote row as an operator recovery aid.

On failure, report the returned `error`, fix the app, repack, and start a fresh upload. Each
upload is single-use.

## Zip validation reminders

- `manifest.json` at the zip root and `frontend/index.js`
- When `manifest.backend` is set: `backend/index.js`
- `project/` with source, `SPEC.md`, and vendored `@heysutton/sdk*` including `@heysutton/cli`
- `manifest.permissions` are **`key`** values from `appPermissions`

See [draft-and-publish.md](draft-and-publish.md) for channel isolation, migrations,
schedules, webhooks, env/secrets, and side-effect guidance.
