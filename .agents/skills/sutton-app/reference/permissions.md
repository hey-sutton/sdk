# Permissions

## Model

1. `manifest.json` `permissions` declare the app's **ceiling**
2. At grant/refresh, Sutton intersects that list with the viewing user's **live** permissions
3. The scoped token can never exceed either set

Empty `permissions: []` means the app cannot call Sutton GraphQL successfully for protected
fields — fine for UI-only apps.

## Looking up permissions (required)

Sutton MCP is required. Do **not** invent permission strings.

1. Call MCP tool **`appPermissions`**
2. Choose the permissions your GraphQL operations need (use descriptions to pick)
3. Put each permission’s **`key`** into `manifest.permissions` exactly as returned

Unknown strings cause publish to fail with:

```text
manifest.json "permissions" contains unknown permissions: …
```

## Tips

- Look up fields with `graphql-schema://…` first, then declare only the permissions those
  operations need
- After adding a new Sutton API call, update `permissions` in the same change
- Before pack/publish, check `manifest.permissions` against MCP **`appPermissions`** and
  confirm every Sutton API operation the app calls is covered
- Do not copy admin-only permissions into an app unless the product explicitly needs them
  and the users who open the app hold them
