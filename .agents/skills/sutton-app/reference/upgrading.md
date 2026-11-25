# Upgrading an app's SDK

Each note below starts with what to look for in the app. After `npx -y -p @heysutton/cli@latest
sutton upgrade` and `npm install`, go through every note and apply the ones that match, whatever
version the app came from. A note that doesn't match needs nothing.

This skill already describes the current SDK, so new features need no note here: what the rest
of the skill says is what the upgraded app can use.

## `package.json` lists `@digit/lib-*` packages

The app was built before the SDK was renamed. `upgrade` has already renamed the dependencies
and imports (`@digit/lib-common` → `@heysutton/sdk`, `lib-frontend` → `@heysutton/sdk-react`,
`lib-backend` → `@heysutton/sdk-server`, `lib-build` → `@heysutton/cli`) and removed the
vendored copies under `packages/`.

- Every old export still exists, so the app should build as it is. The `Digit*` names are
  deprecated aliases. Rename them when you touch that code (see the next notes).
- The app now uses the current theme, which follows the Sutton design system. Colours,
  spacing and typography can shift: open every screen in the preview and fix anything that
  no longer lines up, following [design-system.md](design-system.md).
- Search for any remaining `@digit/` string (comments, `tsconfig.json`, docs) and update it.

## Imports `DigitThemeProvider`, `useDigitApiQuery`, `useDigitApiMutation` or `DigitResult`

Deprecated aliases. Use `ThemeProvider`, `useApiQuery`, `useApiMutation` and `ApiResult` from
`@heysutton/sdk-react`. They behave the same.

## Imports `digitJobs` or the `DigitJob*` / `DigitSchedule` types

Deprecated aliases. Use `appJobs` and the matching `AppJobs`, `AppJobRun`, `AppJobRunStatus`,
`AppJobKind` and `AppSchedule` types from `@heysutton/sdk-server`.

## Uses `DigitHost`, `DigitProxyClient` or the `DigitHost*` types

Deprecated aliases. Use `AppHost` and `AppProxy` (and `AppHostSettings`,
`AppHostDownloadOptions`, `AppHostPrintOptions`). For `window.DigitHost.download(...)` or
`window.DigitHost.print(...)`, call `AppHost.invoke("download", ...)` or
`AppHost.invoke("print", ...)` instead; see [iframe-constraints.md](iframe-constraints.md).

## A job, schedule or webhook handler calls the Sutton API with a token from env or secrets

From `@heysutton/sdk-server` 1.2.0, every backend handler gets an `api` argument that calls
Sutton as the app's bot. Switch the calls to `api.query(...)`, which takes mutations too
([proxy-and-api.md](proxy-and-api.md#sutton-api-from-the-backend)), then remove the token from
the manifest's env and secrets and tell the user they can revoke it.

<!--
Maintainers: add a note when a release renames, removes or changes the behaviour of anything an
app can use. Start the heading with what to grep for in the app, not a version number, and keep
old notes: an app can upgrade from any earlier version.
-->
