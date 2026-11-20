# Admin scope by action and by record type

A record scope can say more than which records an admin reaches. For areas with a flat list of records it can also say what the admin may do with each one: see it, change it, delete it. Discounts and blocks can additionally be confined by record type, so new records of an allowed type are covered without editing the list.

Read this when an admin should see records it may not change, or manage only bonus discounts or only slider blocks, and when a change answers `403 SCOPE_DENIED` although the record is listed.

→ `mcp/docs/api/admin-record-scope-and-field-access#which-records-an-admin-may-touch` · `mcp/docs/api/admin-permission-catalog#a-403-names-the-missing-permission`

## How to give separate lists for view change and delete

A flat scope key accepts either a list of ids, as before, or an object with one list per action.

```json
{
  "orders.storage.scope": { "read": [12, 14], "update": [12], "delete": [] }
}
```

- A plain list gives the same records for every action.
- `read` is what the admin sees. A record can be changed or deleted only if it is also in `read`.
- A missing action list adds no limit of its own: the action follows `read`. An empty list means none.
- The object needs at least one of `read`, `update`, `delete`, each a list of whole positive ids. Any other key, or an empty object, answers `400`.
- `pages.scope` and `catalog.scope` cover a tree and take a plain list only. An object there answers `400`.

The area's own permissions still apply: a record in `update` cannot be changed without the area's update permission.

## Which action a request counts as

- A read is checked against `read`, a delete against `delete`, every other change against `update`.
- List, search, export and preview calls count as reads, including the ones sent with `POST`.
- Removing an item that belongs to a record, such as a coupon of a discount or an order of a storage, counts as deleting from that record.
- Creating a record in an area whose `update` is limited by ids is refused, because the new record cannot be in the list.

## Which areas can be scoped

On top of the keys in `mcp/docs/api/admin-record-scope-and-field-access#which-records-an-admin-may-touch`:

| Key | Records |
|---|---|
| `payments.accounts.scope` | payment accounts: the list, one account, its update, delete and payment provider connection |
| `settings.attributesSets.scope` | attribute sets: the list, the list of all attributes, one set and every operation on it |

Checking whether a marker is free is not a read of a record and is never refused by a scope.

## How to confine discounts or blocks to some types

| Key | Values |
|---|---|
| `discounts.types.scope` | `DISCOUNT`, `BONUS`, `PERSONAL_DISCOUNT` |
| `blocks.types.scope` | block types such as `common_block`, `product_block`, `slider_block` |

The value is a list of types, or the same per-action object with types instead of ids. The full list of types for each key is in `scopes[].values` of `AdminsController_getPermissionsCatalog`.

- A type scope and the id scope of the same area are both applied: a record must pass both.
- Lists and per-type counts include only the allowed types.
- A new record can be created only with a type in `update`. Changing a record's type to one outside `update` is refused.

## What the refusals look like

| Trigger | Answer |
|---|---|
| Reading, changing or deleting a record outside the action's list | `403`, `code: SCOPE_DENIED`, `permission` is the area's id scope key, for example `discounts.scope` |
| Creating a record of a type outside `update`, or changing to one | `403`, `code: SCOPE_DENIED`, `permission` is the type scope key, for example `blocks.types.scope` |
| An object on a tree scope, an unknown action or type | `400` |

## How the catalog describes scopes

`AdminsController_getPermissionsCatalog` lists every scope in `scopes`. An entry with `isTree: false` accepts the per-action object. A type scope entry also carries `values` and `idScope`, the id scope it is applied with. In `AdminsController_getPermissionsValueTypes` id scopes are `number[]` and type scopes are `enum[]`.

## Who can grant a per-action scope

The rule in `mcp/docs/api/admin-record-scope-and-field-access#who-can-grant-a-scope-or-a-field-rule` applies to every action on its own. An author who may see records 12 and 14 but change only 12 cannot grant a plain list `[12, 14]`, because that list also grants changing 14.

## Common mistakes

- **Sending `update` with ids outside `read`.** They are accepted but give nothing: the record is not visible.
- **Sending `"delete": []` to mean no limit.** An empty list blocks every delete. Leave `delete` out to follow `read`.
- **Retrying a create with another type.** Check `update` of the type scope first.

→ `mcp/docs/api/admin-record-scope-and-field-access#what-the-refusal-codes-tell-you` · `mcp/docs/api/admins-and-permissions#never-replace-an-admins-permission-map`
