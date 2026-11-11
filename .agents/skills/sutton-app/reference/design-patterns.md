# Screen recipes

[design-system.md](design-system.md) has the rules and says which component to use;
[components.md](components.md) has each component's props. These recipes say how a whole screen
is put together. They are taken from the Sutton product mockups, so a screen built from one sits
naturally next to the platform's own screens.

## How to use this file

1. Find the recipe closest to what the user asked for. Most apps combine a few: a list page, a
   details panel and a form dialog, say.
2. Copy its structure: the same components, in the same order, with the same props.
3. Fill in the app's data and copy.
4. If none fits, build from the closest one; do not invent a new layout.

## Index

| Recipe | Use it for |
| --- | --- |
| [List page](#list-page) | Any index of records: orders, items, bins, logs |
| [Details panel over a list](#details-panel-over-a-list) | Opening one row without leaving the list |
| [Record page](#record-page) | One document with sections: an order, a quote, a job |
| [Settings page](#settings-page) | An app's settings, a handful of options |
| [Permissions groups](#permissions-groups) | Grants, scopes, feature toggles in groups |
| [Form dialog](#form-dialog) | Creating or editing something with a few fields |
| [Confirm and blocked](#confirm-and-blocked) | Before a destructive action, or when an action is blocked |
| [Dashboard](#dashboard) | A home or overview screen with tiles and task lists |
| [Status and action](#status-and-action) | One current state, one action, and its history (a time clock, a machine on/off) |
| [Empty, loading and error](#empty-loading-and-error) | Every screen's other states |
| [Row and record actions](#row-and-record-actions) | Edit, duplicate, delete, export |
| [Checklist with progress](#checklist-with-progress) | Ordered tasks that get marked complete |
| [Chat composer](#chat-composer) | A box where the user talks to an assistant |

---

## List page

The canonical index screen (Figma "Scorecard"): a page header, then a full-bleed table on
white. Nothing else, no card, no intro paragraph.

```
PageHeader            60px · title (+ Chip) · secondary · primary "New …"
  tabs?               only if the list has sibling datasets
Table toolbar         44px · saved views left · search, refresh, download right
DataTable             36px rows · page gutter on first/last column
Footer                44px · "1–10 of 240" · ‹ ›
```

```tsx
import {
  Bookmark, Calendar, DataTable, DollarSign, Loader, PageHeader, Plus, RecordName, StatusPill,
  type DataTableColumn, type RecordStatus,
} from "@heysutton/sdk-react"
import Box from "@mui/material/Box"
import Button from "@mui/material/Button"

// Every status word the API returns, mapped to its Figma stage.
const orderStage: Record<Order["status"], RecordStatus> = {
  DRAFT: "preliminary", OPEN: "open", IN_PROGRESS: "active", ON_HOLD: "stopped",
  FULFILLED: "done", CANCELLED: "closed",
}

const columns: DataTableColumn<Order>[] = [
  { field: "name", headerName: "Order", flex: 1, minWidth: 240, icon: Bookmark,
    renderCell: (p) => <RecordName name={p.row.name} link /> },
  { field: "status", headerName: "Status", width: 140, icon: Loader, sortable: false,
    renderCell: (p) => <StatusPill status={orderStage[p.row.status]} label={p.row.statusLabel} /> },
  { field: "total", headerName: "Total", width: 120, icon: DollarSign, type: "number" },
  { field: "dueAt", headerName: "Due", width: 120, icon: Calendar },
]

export function OrdersPage() {
  // paging, sort, search, view state; useApiQuery with those as variables
  return (
    <Box sx={{ minHeight: "100vh", bgcolor: "background.default" }}>
      <PageHeader
        title="Orders"
        actions={
          <Button variant="contained" color="primary" startIcon={<Plus />} onClick={openCreate}>
            New order
          </Button>
        }
      />
      <DataTable
        rows={rows} columns={columns} rowCount={total} loading={loading}
        paginationModel={paging} onPaginationModelChange={setPaging}
        sortModel={sort} onSortModelChange={setSort}
        search={{ value: search, onChange: setSearch }}
        views={{ options: [{ value: "open", label: "Open" }, { value: "all", label: "All" }], value: view, onChange: setView }}
        onRefresh={refetch}
        onRowClick={(row) => setSelectedId(row.id)}
      />
      <OrderPanel id={selectedId} onClose={() => setSelectedId(null)} />
    </Box>
  )
}
```

- Paging, sorting and search run on the server. Pass the models and refetch in the handlers. Do
  not sort or filter a full list in the browser; confirm the API's arguments in
  `graphql-schema://…`.
- `rowCount` is the total across all pages; the pager depends on it.
- Tabs that each have their own primary action: keep one `PageHeader` in the app shell and let
  each tab's config supply its `actions` (see `src/frontend/App.tsx` in the `full-featured` template).
- One primary action, in the header. Bulk actions go in `selection.actions`. Row actions go in
  an overflow menu (see [Row and record actions](#row-and-record-actions)).
- Saved views (Open / All, or the user's own) are `views`. Sibling datasets that are not the same
  records (Inventory bins / Work centers) are `PageHeader tabs` instead.
- Clicking a row opens the [details panel](#details-panel-over-a-list), or navigates to the
  [record page](#record-page) when the record has a lot of content.
- A small fixed set (roles, a few config entries) uses MUI's plain `Table` with no head:
  `TableRow hover`, `TableCell`, the name in `h6`, a `StatusPill` or `Chip`, and a trailing
  `ChevronRight` (16px) when the row opens something.

## Details panel over a list

One row's details, beside the list, so the list stays visible (Figma "Side peek").

```tsx
<SidePanel
  open={Boolean(id)}
  onClose={onClose}
  title="Details"
  actions={<IconButton aria-label="More actions" onClick={openMenu}><Ellipsis /></IconButton>}
  footer={dirty ? { onCancel: reset, primary: <Button variant="contained" color="primary" startIcon={<Save />} loading={saving} loadingPosition="start" onClick={save}>Save</Button> } : undefined}
>
  <Section title="General">
    <FormField horizontal label="Customer">
      <Select variant="standard" fullWidth displayEmpty>…</Select>
    </FormField>
    <FormField horizontal label="Reference">
      <TextField hiddenLabel fullWidth placeholder="Enter value" />
    </FormField>
    <FormField horizontal label="Due by">
      <TextField hiddenLabel fullWidth type="date" />
    </FormField>
  </Section>
  <Section title="Notes">
    <TextField hiddenLabel multiline minRows={3} fullWidth placeholder="Add a note" />
  </Section>
</SidePanel>
```

- Each `Section` is a band; the panel pads and divides them. Rows are `FormField horizontal`
  (128px label, 28px control). Selects are invisible (`variant="standard"`); free text is the
  grey field.
- A read-only panel has no footer. An editing panel shows Cancel and Save once something changed.
- One panel component per app, reused for every list. Never a new panel per screen.

## Record page

One document (Figma "Sales order"): a breadcrumb header, the document in a white card on the
grey page, and optionally a docked details panel on the right.

```
PageHeader variant="record"   Orders › SO-1234 · icon buttons · primary Save
┌ background.secondary ─────────────────────────────────┬ SidePanel docked 367 ┐
│   RecordCard 850 · h2 title · owner · date · StatusPill │ Details · Layout tabs │
│   ├ Section "Customer"                                 │ form rows             │
│   ├ Section "Shipping & delivery"                      │ Custom fields         │
│   ├ Section "Items" (+ Add items)                      │ Internal notes        │
│   └ Section "Notes"                                    │ Files                 │
└───────────────────────────────────────────────────────┴──────────────────────┘
```

```tsx
<Box sx={{ display: "flex", flexDirection: "column", height: "100vh" }}>
  <PageHeader
    variant="record"
    title={order.number}
    trail={[{ label: "Orders", onClick: goBack }]}
    actions={<Button variant="contained" color="primary" startIcon={<Save />} disabled={!dirty}>Save</Button>}
  />
  <Box sx={{ display: "flex", flex: 1, minHeight: 0 }}>
    <Box sx={{ flex: 1, overflow: "auto", bgcolor: "background.secondary", px: 2 }}>
      <RecordCard
        title={`Sales order ${order.number}`}
        meta={<><Avatar sx={{ "--digit-avatar-size": "16px" }}>SB</Avatar> Sean B.</>}
        status={<StatusPill status={orderStage[order.status]} label={order.statusLabel} />}
      >
        <Section title="Shipping & delivery">
          <Box sx={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: 1 }}>
            <FormField label="Deliver by"><TextField variant="standard" type="date" fullWidth /></FormField>
            <FormField label="Carrier"><Select variant="standard" fullWidth displayEmpty>…</Select></FormField>
            <FormField label="Service"><Select variant="standard" fullWidth displayEmpty>…</Select></FormField>
          </Box>
        </Section>
        <Section title="Items" actions={<Button startIcon={<Plus />}>Add items</Button>}>
          <DataTable inset="none" rowHeight={52} … />
        </Section>
      </RecordCard>
    </Box>
    <SidePanel variant="docked" open title="Details" onClose={collapse}>…</SidePanel>
  </Box>
</Box>
```

- Each `Section` inside the `RecordCard` is a band; the card pads and divides them.
- On the record, labels sit **above** fields (`FormField`), fields in equal columns, gap 8.
- Section titles are h6; hints beside them ("Shared with customer") are `subtitle`.
- Read-only computed values (subtotals) are `text.tertiary`.

## Settings page

An app's settings (Figma "Settings › General"). Few settings fit in a [side panel](#details-panel-over-a-list);
use this when there are several groups.

```tsx
<PageHeader title="Settings" />
<PageBody maxWidth={720}>
  <SettingsSection title="Details" size="sm" collapsible={false}>
    <SettingsRow variant="setting" label="Workspace name"
      description="Shown on documents you send."
      control={<TextField variant="standard" fullWidth defaultValue={name}
        slotProps={{ htmlInput: { "aria-label": "Workspace name" } }} />} />
    <SettingsRow variant="setting" label="Prevent over shipping"
      description="Block shipments above the ordered quantity."
      control={<Switch checked={value} onChange={toggle}
        slotProps={{ input: { "aria-label": "Prevent over shipping" } }} />} />
  </SettingsSection>
  <SettingsSection title="Defaults" size="sm" collapsible={false}>
    …
  </SettingsSection>
</PageBody>
```

- Label (body1) and description (`overline`, tertiary) on the left; the control in the 256px
  column on the right. The row draws the label but does not link it, so give every control an
  `aria-label` with the label's text. The switch here is MUI's bare `Switch`; outside a row,
  use `SwitchField`.
- Bordered label/value lists (custom fields) are a `PropertyGrid`.
- Saving: one Save in the header or a footer, disabled until something changed. Or save each
  control on change and confirm with a `Snackbar`; do not mix the two on one screen.

## Permissions groups

Grants and toggles in collapsible groups (Figma "Roles & permissions").

```tsx
<SettingsSection
  title="Inventory"
  selection={{ checked: granted.length, total: all.length, onChange: (all) => setAll(all) }}
>
  {all.map((p) => (
    <SettingsRow key={p.id} variant="list" label={p.label} info={p.help}
      control={<Checkbox checked={granted.includes(p.id)} onChange={() => toggle(p.id)} />} />
  ))}
</SettingsSection>
```

- Each row is 36px, the whole row toggles, dividers run full width.
- The group header shows "Select all" / "Deselect all" automatically.

## Form dialog

Creating or editing something with a few fields (Figma "Modal").

```tsx
<Modal
  open={open}
  onClose={close}
  title="New bin"
  footer={{
    onCancel: close,
    primary: <Button variant="contained" color="primary" startIcon={<Save />} loading={saving}
      loadingPosition="start" disabled={!valid} onClick={save}>Save</Button>,
  }}
>
  <Section>
    <FormField horizontal label="Name" required>
      <TextField hiddenLabel fullWidth placeholder="Enter value" />
    </FormField>
    <FormField horizontal label="Location">
      <Select variant="standard" fullWidth displayEmpty>…</Select>
    </FormField>
  </Section>
</Modal>
```

- One primary action, on the right. Cancel is the text button on the left; the X also cancels.
- Several groups of fields: one titled `Section` per group.
- Choices that need a sentence each: `RadioGroup` with `RadioCard`s, stacked.
- Tabs that switch the whole content go in `Modal tabs`.
- Validation errors go on the field (`FormField error`), and `AppErrorAlert` above the footer for
  a failed save.

## Confirm and blocked

**Any destructive action asks first.** Delete, remove, cancel an order, or anything that cannot be
undone opens a `ConfirmDialog destructive` before it runs. Renaming or saving a draft does not.

```tsx
<ConfirmDialog
  open={confirmOpen}
  title="Delete role?"
  description="Users with this role lose access. This cannot be undone."
  confirmLabel="Delete"
  cancelLabel="Keep role"
  destructive
  busy={deleting}
  error={deleteError}
  onConfirm={handleDelete}
  onCancel={() => setConfirmOpen(false)}
/>
```

- The title is a question; the description is one line on the consequence.
- The confirm label is the verb, never "OK". Close the dialog only when the action succeeds;
  on failure pass the error to `error` and keep it open.

**Blocked**: when an action cannot happen because something else must change first, use the same
dialog with a `TriangleAlert` icon and a non-destructive label that takes the user to the fix.

```tsx
<ConfirmDialog
  open={blockedOpen}
  icon={<TriangleAlert />}
  title="Can't delete role"
  description="This role is assigned to 4 users. Reassign them first."
  confirmLabel="Go to users"
  onConfirm={goToUsers}
  onCancel={() => setBlockedOpen(false)}
/>
```

Navigating to another part of the platform goes through the host: check `getHostCapabilities`
for a navigation method before you write one.

## Dashboard

A home or overview (Figma "Home"): tiles and lists on the grey page.

```tsx
<PageHeader title="Production" />
<Box sx={{ bgcolor: "background.secondary", minHeight: "100%" }}>
  <PageBody>
    <Section title="Open jobs" subtitle="4 active"
      actions={<Button variant="text" endIcon={<ArrowUpRight />}>View all</Button>}>
      <Box sx={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(240px, 1fr))", columnGap: 1, rowGap: 2 }}>
        {jobs.map((job) => (
          <Card key={job.id} sx={{ p: 2 }}>
            <Typography variant="overline" color="text.secondary">{job.number}</Typography>
            <Typography variant="body2" sx={{ mt: 1 }}>{job.name}</Typography>
            <ProgressBar value={job.done} total={job.qty} unit="pcs" sx={{ mt: 2 }} />
          </Card>
        ))}
      </Box>
    </Section>
  </PageBody>
</Box>
```

- Tiles are `Card` (outlined, radius 16, padding `p: 2`), in a grid with an 8px column gap and a
  16px row gap. Never wrap the whole dashboard in a card.
- Task lists: one `Card` holding a `List` of `ListItemButton` rows: a 20px icon, a `body2` title, a
  `body1` secondary description and an `overline` tertiary time on the right.
- KPI tiles: `Card sx={{ p: 2 }}` with a `Section` heading, the number in `h2`, and a `Chip`
  (`color="green"` / `"red"`, `size="small"`) for the delta or a breach. Missing values show
  "None"; loading shows `Skeleton`.
- Charts (`@mui/x-charts`) sit in a `Card` with a `Section` heading; series colours come from
  palette accents (`theme.palette.blue.main`, `emerald`, `violet`, `amber`), axis labels stay
  the chart's defaults, and an empty chart shows an `EmptyState` in the card.

## Status and action

A screen about one thing's current state with one main action (clock in / out, start / stop a
machine) and the history of that action.

```tsx
<PageHeader
  title="Time card"
  actions={
    <Button variant="contained" color="primary" loading={busy} loadingPosition="start"
      startIcon={clockedIn ? <LogOut /> : <LogIn />} onClick={toggle}>
      {clockedIn ? "Clock out" : "Clock in"}
    </Button>
  }
/>
<PageBody maxWidth={720}>
  <Section title="Today">
    <PropertyGrid rows={[
      { label: "Status", value: clockedIn
        ? <StatusPill status="active" label="Clocked in" />
        : <StatusPill status="closed" label="Clocked out" /> },
      { label: "Worked today", value: formatDuration(total) },
    ]} />
  </Section>
  <Section title="Recent shifts">
    <DataTable inset="none" … />
  </Section>
</PageBody>
```

- The one action is the header's primary button; its label and icon follow the state. It is
  not red, even when it stops something.
- The current state is a `StatusPill` in a `PropertyGrid`; the history is a `DataTable` (or an
  `EmptyState` when there is none).

## Empty, loading and error

Every screen has all three; design them, do not leave a blank area.

- **Empty**: `EmptyState` where the content would be, inside `PageBody`, with at most one
  button in `actions`. A filtered list with no matches says so and offers to clear the filter.
- **Loading**: `DataTable loading`, `Button loading` (spinner in place of the start icon),
  `Skeleton` blocks the shape of the content for cards. Never a full-page spinner, never
  "Loading..." text on its own.
- **Error**: `AppErrorAlert error={error} onRetry={refetch}` at the top of the content, inside
  `PageBody`. Field errors on the field. A failed confirm shows in the `ConfirmDialog` (`error`).

## Row and record actions

- The record's main action is a visible button (header or panel footer). Everything else goes
  behind one `IconButton aria-label="More actions"` with `Ellipsis`, opening a `Menu`.
- Menu items are sentence case with a 16px `ListItemIcon`. The destructive item goes last and
  opens `ConfirmDialog destructive`.
- In a `DataTable`, the overflow menu is the last column (`width: 48`, `sortable: false`),
  rendering the `IconButton`. Never put buttons, and never a red button, in a row.
- Export is the table's `onDownload` (it calls `AppHost.invoke("download", …)`); print is
  `AppHost.invoke("print", …)`.

## Checklist with progress

An ordered list of tasks grouped into sections, each with a completion measure.

- Each section is a `Section` heading with a `ProgressBar` ("value / total unit", never a
  percentage).
- Each row has a completion marker (`CircleCheck` or a `Checkbox`), a `body2` title, a `body1`
  secondary description and a `StatusPill` on the right.
- Done rows stay where they are; do not reorder the list when a task completes.
- A blocked task shows the reason and who is fixing it, in one line under the row.

## Chat composer

A conversation column with a composer pinned to the bottom.

- The composer is a multiline grey `TextField hiddenLabel` with a placeholder, an optional attach
  `IconButton` (`Paperclip`) and a send `IconButton color="primary"` (`Send`).
- Assistant messages sit in `Card`s; the user's messages are right-aligned `Card`s on
  `bgcolor: "background.secondary"`.
- Keep the composer pinned; never inside a panel that scrolls away.
