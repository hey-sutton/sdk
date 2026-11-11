# Design system

Sutton apps run inside the Sutton platform, next to its own screens. They must look like those
screens. The **Figma design system is the source of truth** for what every component looks like,
and `@heysutton/sdk-react` implements it: `ThemeProvider` restyles every MUI component to the
Figma values (light and dark), and the library adds the composite pieces the product mockups use.

Your job is to **compose** those pieces, never to style them. If an app looks like stock MUI, or
like a generic SaaS template, something below was skipped.

Read this file before you write any UI. Then:

1. Pick a screen recipe from [design-patterns.md](design-patterns.md) and build from it.
2. Look up each component's props and conventions in [components.md](components.md) as you use it.
3. Run the [self-check](#self-check-before-you-finish) on every screen before you finish.

## Contents

- [Non-negotiables](#non-negotiables)
- [Component picker](#component-picker)
- [Typography](#typography)
- [Colour](#colour)
- [Spacing and layout](#spacing-and-layout)
- [Icons](#icons)
- [Copy and values](#copy-and-values)
- [Self-check before you finish](#self-check-before-you-finish)

## Non-negotiables

1. **Use the library component when one exists.** `PageHeader`, `PageBody`, `Section`,
   `DataTable`, `StatusPill`, `RecordName`, `RecordCard`, `PropertyGrid`, `Modal`, `SidePanel`,
   `ConfirmDialog`, `FormField`, `SettingsSection`, `SettingsRow`, `SwitchField`, `RadioCard`,
   `EmptyState`, `ProgressBar`, `FileDropZone`, `FileUploadRow` and `AppErrorAlert` exist so you
   never rebuild them from `Box` and `Typography`. Import them from `@heysutton/sdk-react`.
   Everything else is a themed MUI component (`Button`, `TextField`, `Chip`, `Tabs`, …): use it
   as it is.
2. **Never style a themed component.** No `sx`, `style` or `styled()` that sets colour,
   background, border, border radius, box shadow, font size, font weight, line height, height or
   padding on a `Button`, `IconButton`, `TextField`, `Select`, `Checkbox`, `Switch`, `Radio`,
   `Tabs`, `Tab`, `Chip`, `Alert`, `Dialog`, `Menu`, `Tooltip`, `Card` or `DataTable`. The theme
   already matches Figma; an override breaks it. Layout props (`gap`, `flex`, `width`,
   `maxWidth`, `mt`, `alignSelf`) are fine, and so is content padding on a `Card` (`p: 2`),
   which the theme leaves to you.
3. **No hex, `rgb()`, px font sizes or hand-picked greys.** Colours come from the palette tokens
   listed under [Colour](#colour). Text styles come from the `Typography` variants listed under
   [Typography](#typography).
4. **Lucide icons only, from `@heysutton/sdk-react`**, at 12, 16 or 20px. Never MUI icons, never
   `lucide-react` directly, never emoji as icons.
5. **Sentence case** for every title, label, button and menu item.
6. **Left-aligned.** Titles, labels, cards and buttons sit on the left. Only `EmptyState` and
   `FileDropZone` centre their content.
7. **Buttons size to their label.** Never `fullWidth` on a `Button`, and never stretch one with
   `width: "100%"`, `flex: 1` or a full-width stack (the two answers in `ConfirmDialog` are the
   built-in exception).
8. **No cards around pages, lists or forms.** A list page is a `PageHeader` and a full-bleed
   `DataTable` on white. A `Card` is only for a grouped block (a dashboard tile, an assistant message).
9. **Light and dark both work for free** if you follow rules 2 and 3. Do not branch on
   `theme.palette.mode` in app code.
10. **Never** browser `alert`/`confirm`/`prompt`; use `ConfirmDialog`. Never `CircularProgress`
    page spinners for a list; `DataTable` has `loading`.

## Component picker

| You need | Use |
| --- | --- |
| The title bar of any screen | `PageHeader` (`variant` page or record). `Modal` and `SidePanel` draw their own |
| Content under the header that is not a full-bleed table | `PageBody` (page gutter and spacing) |
| A list of records | `DataTable` (server paged, sorted, searched) |
| A short fixed list (roles, config entries) | Plain MUI `Table` + `TableRow hover` + `TableCell`, no `TableHead` |
| A record's name in a list or header | `RecordName` |
| A record's status | `StatusPill` (one of six Figma stages) |
| A category, tag, linked record or delta | `Chip` (themed as the Figma pill) |
| A single record's page | `PageHeader variant="record"` + `RecordCard` with a `Section` per block |
| Details or settings beside a list | `SidePanel variant="overlay"` |
| Details beside a record | `SidePanel variant="docked"` |
| A heading over a block of content | `Section` (h6 title, overline count or hint, actions) |
| Label/value pairs in a bordered block | `PropertyGrid` |
| A settings page | `SettingsSection` + `SettingsRow variant="setting"` |
| Grants or permissions | `SettingsSection selection` + `SettingsRow variant="list"` + `Checkbox` |
| Fields in a side panel or form dialog | `FormField horizontal` (label left) |
| Fields on a record or a page form | `FormField` (label above) around a `TextField` or `Select` |
| On/off with a label | `SwitchField` (with optional `description`) |
| Two or three options that each need a sentence | `RadioGroup` + `RadioCard` |
| A segmented filter or view switch | `ToggleButtonGroup exclusive` (`size="small"` for the outlined track) |
| A form that creates or edits something | `Modal` (title, `Section` children, footer) |
| "Are you sure?" before a destructive or blocked action | `ConfirmDialog` |
| More actions for a row or record | `IconButton` with `Ellipsis` opening a `Menu` |
| Nothing to show yet | `EmptyState` |
| A failed request | `AppErrorAlert` |
| A short inline message | `Alert` (standard variant) |
| A transient result ("Saved", "Copied") | `Snackbar` + `SnackbarContent` |
| Progress toward a total | `ProgressBar` |
| File upload | `FileDropZone` + `FileUploadRow` per file |
| Dashboard tiles | `Card` (outlined, radius 16) in a CSS grid, with `Section` headings |

## Typography

Inter, with tabular figures and a slashed zero on every style. Pick the variant by role; never set
`fontSize` or `fontWeight`.

| Role | Variant | Size / line / weight |
| --- | --- | --- |
| Page, panel and dialog title | `PageHeader title` (h5) | 16/24 600 |
| Record title in its card | `RecordCard title` (h2) | 22/28 600 |
| A headline number on a dashboard tile (KPI) | `h2` | 22/28 600 |
| Section title, name cell, card label | `h6` | 13/16 600 |
| Labels, values, table cells, paragraphs | `body1` | 13/16 400 |
| List-row titles, card titles, emphasised values | `body2` | 13/16 500 |
| A link inside text | `body1Link` | 13/16 600, underline on hover |
| Meta lines, hints, counts beside headings, descriptions under labels, timestamps | `overline` | 11/12 400 |
| Code-like strings only (API keys, raw ids) | `mono` | 12/15 IBM Plex Mono |

- `h1` (24) and `h3`/`h4` (20/18) exist but native screens rarely use them. Do not use `h1` as a
  page title; the page title is `PageHeader`.
- Numbers, money, dates and record ids are `body1` in `text.primary`, never `mono`. Inter's
  tabular figures line them up.
- `caption` (12px) is not used by native screens; use `overline` for small text.
- Text colour: `text.primary` for content, `text.secondary` for supporting facts (email, address,
  SKU, descriptions in panels), `text.tertiary` for hints, descriptions under setting labels,
  "None", computed read-only values and timestamps.

## Colour

Charcoal is the action colour. Blue means selected or focused (checkbox, radio, switch, focus
ring). Red means destructive or error. Everything else is neutral stone. Category colour only ever
appears inside a `Chip` or `StatusPill`.

| Token | Use |
| --- | --- |
| `text.primary`, `text.secondary`, `text.tertiary`, `text.disabled` | Text and icons |
| `background.default` | Index and list pages (white; #292524 in dark) |
| `background.secondary` | Record pages behind the `RecordCard`, docked panels, an in-app section nav column |
| `background.paper` | Cards, dialogs, menus, panels |
| `border.solidStrong` | Page chrome lines: header bottom, menu outline, nav edge |
| `border.solidDefault` | Section dividers inside a page or card, table toolbar line |
| `border.strong` | Card, panel and dialog outlines (10% alpha) |
| `border.default` / `divider` | Table row and column lines (5% alpha) |
| `error.main` | Error text only. A destructive button gets red from `color="error"` |

Use them through `sx` (`sx={{ color: "text.secondary", borderColor: "border.solidDefault" }}`).
Never use MUI's `grey.*`, `primary.light` or `info.main`. Accent colours (`violet`, `sky`,
`emerald`, …) appear only as a `Chip` `color` and as chart series colours
(`theme.palette.emerald.main`); a good or bad delta on a tile is a `Chip` (`green` / `red`), not
coloured text.

## Spacing and layout

- The spacing unit is 8px on desktop and 10px below 600px; `sx` numbers (`p: 2`, `gap: 1`) scale
  with it automatically. Use spacing numbers, not px.
- **Page gutter**: 36px on desktop, 20px below 600px. `PageHeader`, `DataTable` (`inset="page"`) and
  `PageBody` apply it. Put every other block under a `PageHeader` inside `<PageBody>` (add
  `maxWidth={720}` for settings and forms); do not pad a `Box` by hand, since `px: 4.5` comes out
  at 45px on mobile.
- **Panel and dialog gutter**: 16px (`p: 2`). `SidePanel`, `Modal` and `DataTable inset="panel"`
  apply it.
- `Section`s inside a `Modal`, `SidePanel` or `RecordCard` become bands: they stack with **no
  gap**, separated by a 1px line, each padded 16px vertically. The container does this; add no
  padding or dividers yourself.
- Heights: buttons, fields, menu items and side-panel rows are 28px; table rows and headers 36px
  (52px with a `RecordName` meta line); page headers 60px; toolbars and footers 44px.
- Widths that recur: side panel 448 (overlay) / 367 (docked); record card max 850; dialogs 336
  (xs), 480 (sm), 680 (md), 960 (lg); label column 128; setting control column 256.

## Icons

Lucide, imported from `@heysutton/sdk-react` (see `icons.ts` in the `@heysutton/sdk-react` package, or its type declarations under
`node_modules/@heysutton/sdk-react/dist`, for the list; `Box`, `Link` and `Text` are exported as `BoxIcon`, `LinkIcon`, `TextIcon`). If you need one
that is not exported, say so and name it.

- **16** inline with 13px text, inside buttons, fields, menu items and tabs (the theme sizes icons
  inside `Button` and `IconButton` for you).
- Trends and deltas: `TrendingUp`, `TrendingDown`, `ArrowUp`, `ArrowDown`; clocking in and out:
  `LogIn`, `LogOut`; durations: `Timer`.
- **12** next to 11px `overline` text, in table headers, in `Chip`s and meta lines.
- **20** standing alone (the `ConfirmDialog` icon, dashboard task rows).
- Trailing chevrons, "View all" arrows and other helper icons sit at 60% opacity
  (`sx={{ opacity: (t) => t.palette.opacity.helper }}`).
- Icon-only buttons need `aria-label` and a `Tooltip`.

**Domain icons**

| Domain | Icon | Domain | Icon |
| --- | --- | --- | --- |
| Deals | `Handshake` | Inventory | `Layers` |
| Quotes | `ReceiptText` | Receiving | `PackageOpen` |
| Price lists | `FileBox` | Planning | `CalendarRange` |
| Sales orders | `ShoppingCart` | Production | `Factory` |
| Purchase orders | `BaggageClaim` | Shipping | `Truck` |
| Items | `BoxIcon` | Accounts | `BookUser` |
| Reporting | `FileChartLine` | Users | `Users` |
| Settings | `Cog` | App integrations | `Puzzle` |
| Learning center | `BookOpenText` | Chat | `MessageSquareMore` |

**Data type icons** (table column headers, meta lines, field rows)

| Data | Icon | Data | Icon |
| --- | --- | --- | --- |
| Quantity | `Cylinder` | Unit of measure | `Ruler` |
| Status | `Loader` | Date | `Calendar` |
| Time | `Clock` | Money | `DollarSign` |
| Percent | `Percent` | Count / number | `Hash` |
| Text | `TextIcon` | SKU | `Bookmark` |
| Person | `CircleUserRound` | Company | `Building2` |
| Custom field | `TextCursorInput` | Actions | `Zap` |
| Tags / types | `Tags` | Linked records | `LinkIcon` |

## Copy and values

- Sentence case everywhere ("New bin", "Select all"). Labels are nouns; buttons are verbs.
- Missing text is "None" in `text.tertiary`. A value not fetched yet shows "None" too (or a
  `Skeleton` while it loads), never an empty box. Not applicable is an empty cell. Never "null",
  "N/A" or "-".
- Numeric zero is "0.00", right-aligned. Lists of values are comma separated ("640.00 ft², 300.00 lb").
- Several values in one cell show "Multiple" with a chevron that opens a menu of them.
- Dates are `body1` in `text.primary`; relative times ("1d ago") are `overline` `text.tertiary`.
- Explain, don't just disable: when something is unavailable, say why next to it.

## Self-check before you finish

Go through this list on every screen you built. Fix anything that fails.

- [ ] Every screen starts with `PageHeader`; there is no hand-built header, no `h1`, no centred
      title block.
- [ ] Lists use `DataTable` (or a plain MUI `Table` for a short fixed list), full bleed, with no card
      around them; the create action is in the page header.
- [ ] Every status is a `StatusPill` on the right stage, every category a `Chip`; no coloured
      text for status.
- [ ] No `sx` sets colour, background, border, radius, shadow, font or height on a themed MUI
      component; no hex values; no px font sizes; no `fontWeight`.
- [ ] Buttons: one primary per surface, last in its group; secondary is the default `Button`;
      Cancel is a text button; no `fullWidth`.
- [ ] Icons are Lucide from `@heysutton/sdk-react` at 12/16/20; icon-only buttons have an
      `aria-label` and a `Tooltip`.
- [ ] Destructive actions go through `ConfirmDialog destructive`; forms with fields are a
      `Modal` or `SidePanel`, not a custom dialog.
- [ ] Fields are the themed grey or invisible fields with labels from `FormField` or
      `SettingsRow`; no floating labels, no outlined strokes.
- [ ] Empty, loading and error states exist and use `EmptyState`, `loading` and `AppErrorAlert`.
- [ ] The screen follows the closest recipe in [design-patterns.md](design-patterns.md).
- [ ] It looks right in dark mode too (if you followed the rules, it does).
