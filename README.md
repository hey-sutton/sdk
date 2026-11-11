# Sutton agent plugin

Agent skills for building Sutton apps with Claude Code, Cursor and Codex. The `sutton-app` skill
builds, drafts and publishes Sutton Studio and custom apps through the Digit MCP server.

## Install

- **Claude Code:** `/plugin marketplace add hey-sutton/sdk`, then `/plugin install sutton@sutton`
- **Codex:** `codex plugin marketplace add hey-sutton/sdk`, then `codex plugin add sutton@sutton`
- **Cursor:** a team admin imports this repo under **Import from Repo** in the team marketplace,
  or clone it into `~/.cursor/plugins/local/sutton`
- **Any other agent:** `npx skills add hey-sutton/sdk`

Installed copies update when the plugin version changes.

## Build an app

```bash
npx @heysutton/cli@latest templates
npx @heysutton/cli@latest create my-app --template hello-world
cd my-app
npm run pack        # writes app.zip
```

The SDK packages are on npm: `@heysutton/sdk`, `@heysutton/sdk-react`, `@heysutton/sdk-server`
and `@heysutton/cli`.

## About this repository

This repository is published automatically from Sutton's main repository and holds only the
agent plugin. Pull requests here can't be merged directly. Please open an issue instead.
