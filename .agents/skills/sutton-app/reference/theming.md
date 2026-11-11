# Theming

Digit apps should look like Digit. Use **one stack only**:

**React + MUI + `@heysutton/sdk-react` (`ThemeProvider`)**

Do not invent a parallel design system, skip the frontend package, or ship vanilla
HTML/CSS UI for new apps.

## Package

`@heysutton/sdk-react` implements the Sutton Figma design
system as an MUI theme (palette, typography, component overrides), exported as `ThemeProvider`,
plus the components that group themed parts. Use it as-is: don't copy its tokens into your app or
build a second design system on top. It is self-contained: the private `digit-web` repo is
**not** a dependency. Wrap the app in this `ThemeProvider`, not MUI's own `ThemeProvider` (it
already wraps MUI's). If a file needs MUI's too, import it under another name.

Depend on it from an app:

```json
{
  "dependencies": {
    "@heysutton/sdk-react": "^1.0.0",
    "@emotion/react": "^11.14.0",
    "@emotion/styled": "^11.14.1",
    "@mui/material": "^9.3.1",
    "react": "^19.2.8",
    "react-dom": "^19.2.8"
  }
}
```
`sutton create` writes these for you. Apps created inside App Builder, or vendored with
`sutton studio vendor`, link `file:./packages/sdk-react` instead; leave whichever form the app
already uses.

For Worker helpers (`createHandler`, `backendPath`, `ok`/`err`, `requireEnv`), depend on
`@heysutton/sdk-server`. Bundling is handled by `@heysutton/cli` (`sutton studio pack`). See the
`full-featured` template.

## Provider

```tsx
import { ThemeProvider } from '@heysutton/sdk-react';

createRoot(rootEl).render(
  <ThemeProvider>
    <App />
  </ThemeProvider>,
);
```

`ThemeProvider`:

1. Reads light/dark from `AppHost` (exported from `@heysutton/sdk-react` with its
   `AppHostSettings` type; importing the package augments `Window`)
2. Falls back to `document.documentElement.dataset.theme`, then `prefers-color-scheme`
3. Calls `createTheme(themeOptions(darkMode))` and renders MUI `CssBaseline`

Do not add a local `digit.d.ts` for the harness globals. Prefer hooks over calling
`window.AppProxy` yourself.

## Host settings

```ts
import { AppHost } from '@heysutton/sdk-react';

AppHost.getSettings(); // AppHostSettings | null
AppHost.onSettingsChange((settings) => { /* ... */ });
```

`ThemeProvider` bundles self-hosted Inter (`inter-ui`). Do not add a Google Fonts or other CDN stylesheet. The harness sets
`data-theme` and `lang` on `<html>`. App look-and-feel comes from MUI +
`ThemeProvider`, not a parallel CSS design system.

## Building UI

This file covers wiring the theme up. For how screens should look, follow
[design-system.md](design-system.md) (rules, component picker, tokens),
[components.md](components.md) (props per component) and
[design-patterns.md](design-patterns.md) (screen recipes).

Do not assume the harness injects React, MUI or fonts; it injects only host APIs.
