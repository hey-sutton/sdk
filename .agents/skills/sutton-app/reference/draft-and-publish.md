# Draft and publish behavior

Sutton has two deployment channels for a custom app:

- **Draft** is an owner-only environment for the current app author.
- **Published** is the production environment visible to everyone who can access the app.

The platform keeps the channels separate with different Workers, D1 databases, R2 buckets,
sessions, and bundle pointers. A draft deployment does not change what the published app serves.
The `app.zip` produced by `sutton studio pack` is channel-neutral: the same reviewed build can be
promoted without rebuilding it.

## Deployment flow

For the Sutton App Builder:

```text
edit → npm run pack → publishAppZip → owner reviews draft → explicit web Publish → published app
```

For standalone MCP clients, `publishApp` accepts `channel: "preview" | "live"`. The omitted
value remains `live` for backwards compatibility, so a caller must pass `channel: "preview"`
when it wants a non-production deploy.

Never describe a successful draft deployment as a production publish. If a draft deploy
fails, fix the bundle and deploy a new upload. If the user wants to ship, use the Sutton web
app's explicit Publish action for the current draft build. Publish ships the reviewed
build and applies pending migrations on the published app only — it is not a config promote. **Promote does
not wipe published env/secrets** — draft shares published env and secrets, and Sutton Settings write
the published app only. Nothing is copied from draft onto the published app for env or secrets.

## What a draft does and does not test

- Draft migrations run against the draft D1. Promotion applies the build's pending
  migrations to the published D1. Migrations are additive and forward-only: rollback runs
  older code against the newer schema. See [d1-migrations.md](d1-migrations.md).
- **Schedules run on the published app only.** Draft cron is not registered, so timer-driven behavior is
  not production-equivalent. On-demand jobs use the draft job namespace when invoked by
  the draft app.
- Inbound webhooks are published-only and are not delivered to a draft Host. Draft cron,
  webhooks, and other timer/webhook side effects stay off. Test webhook handlers with signed
  local/unit fixtures or a deliberate test against the published app.
- Draft sessions use the viewer's Sutton permissions for GraphQL reads and writes,
  still limited to `manifest.permissions`. App-owned D1/R2 writes are isolated to
  draft resources.
- Draft uses the **published env vars and published secrets**. Sutton Settings that edit env or
  secrets update the published app only. Nothing is copied from draft onto the published app for env or secrets —
  draft already shares them, and promote is not a config copy.
- Do not seed published data into draft by default. Published data may contain sensitive information;
  use synthetic data unless the user explicitly requests a permitted seed operation.

## External side effects

Because a draft always has published credentials, draft code must not assume that external
calls are harmless. Guard third-party writes, email, payments, and other irreversible
actions. Changing Settings to test credentials would also change the published app.

There is currently no documented app-facing `isPreview()` runtime helper. Do not infer the
channel from internal platform headers. If app behavior must differ between channels, track a
platform/SDK channel signal as a separate product change.
