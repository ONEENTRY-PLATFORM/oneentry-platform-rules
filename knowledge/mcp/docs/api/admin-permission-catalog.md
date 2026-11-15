# Admin permission catalog and refusal details

The admin permission catalog describes every admin permission key: its area, the kind of action it governs, whether it is sensitive, and the shape of its value. A refusal for a missing key now names that key in the response body.

Read it before you build a permission map for a new admin, and whenever an operation answers `403` and you need to know exactly which grant to ask for.

→ `mcp/docs/api/admins-and-permissions#permissions-are-dotted-keys` · `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## What the permission catalog returns

`AdminsController_getPermissionsCatalog` — `GET /admins/permissions/catalog` — answers any signed-in administrator, with no key required. Without a token it answers `401`.

```json
{
  "permissions": [
    { "key": "blocks.delete", "section": "blocks", "kind": "delete",
      "sensitive": false, "valueType": "boolean", "gated": true }
  ],
  "scopes": [{ "key": "forms.scope", "section": "forms", "isTree": false }],
  "fields": { "discounts": ["startDate", "endDate", "gifts"] }
}
```

`permissions` has one entry for every key that `AdminsController_getAllAvailablePermissionsKeys` lists, and no others. `scopes` lists the keys that confine an account to chosen records, and `fields` lists, per area, the field names a field rule may name.

→ `mcp/docs/api/admin-record-scope-and-field-access#which-records-an-admin-may-touch`

- `section` — the area the key belongs to. It is the first dotted segment, except for `settings.*` keys, where it is the first two (`settings.locales`).
- `kind` — one of `read`, `create`, `update`, `delete`, `switch`, `status`, `positions`, `export`, `other`. Use it to grant or withdraw one kind of action across every area at once, for example every `delete` key or every `export` key.
- `valueType` — `boolean`, `string[]`, `number[]` or `object`, the same shapes the value-types listing reports. Only `boolean` keys are granted as `true`.
- `sensitive` and `gated` — see the two sections below.

## Which permissions count as sensitive

An entry with `sensitive: true` controls something destructive, or something that reaches beyond the account itself: another admin's permissions or credentials, deleting the journal, reading sensitive form fields, managing data environments, payment refunds.

When you provision an admin with "everything", leave the sensitive keys out and grant them only when a person explicitly asked for that key. A full map is every `boolean` entry with `sensitive: false` that you are allowed to grant. Add `admins.modules` too, or the account has no sections to work in. You can only grant what your own account holds, so build the map from your own grants, not from the catalog alone.

## A key the platform does not check

`gated: false` marks a key that no operation requires. The admin panel still reads it to decide what to show, so grant it as usual when a person asks for it. Do not expect withdrawing it to block anything through the API.

## A 403 names the missing permission

A refusal for a missing key carries the key in its body:

```json
{
  "statusCode": 403,
  "message": "Forbidden resource",
  "code": "PERMISSION_DENIED",
  "permission": "system.maintenance"
}
```

`message` is unchanged, so code that reads only `message` keeps working. When `code` is `PERMISSION_DENIED`, ask for exactly the key in `permission` — not a neighbouring key, and not a wider set.

Two other codes mean the key is not the problem:

| `code` | Meaning |
|---|---|
| `SCOPE_DENIED` | The record is outside the records the account may touch; `permission` names the scope key, `ids` the records |
| `FIELD_DENIED` | The request changes a field the account may only read; `fields` names them |

Some `403` responses carry no `code`: rules about the project owner, and grants beyond your own. Read those by their `message` as before. The key is not the cause there, so asking for it changes nothing.

## Keys on block visibility and position maintenance

Two operations that any signed-in administrator could call now each require their own key:

| Operation | Key |
|---|---|
| `AdminBlocksController_setVisibility` — show or hide one product in a similar-products block | `blocks.switching` |
| `AdminSystemController_positionsMaintenance` | `system.maintenance` |

Without the key they answer `403` with `code: PERMISSION_DENIED`, and nothing changes. Administrators that already held other keys hold these too. An account created with an empty map does not.

## Common mistakes

- **Granting every catalog key.** Sensitive keys are refused unless your own account holds them, and they should not be granted without an explicit request anyway.
- **Granting a `number[]` or `string[]` key as `true`.** That is a `400`; send its value.
- **Answering a `PERMISSION_DENIED` by requesting a wider set.** Ask for the one key named in `permission`.
- **Reading a `403` with no `code` as a missing key.** Check ownership and your own grants first.
- **Answering `SCOPE_DENIED` or `FIELD_DENIED` by asking for the key in `permission`.** The account already holds it; the record or the field is what is restricted.

→ `mcp/docs/api/admins-and-permissions#a-recognised-key-with-the-wrong-value-is-a-different-400` · `mcp/docs/api/admin-page-scope`
