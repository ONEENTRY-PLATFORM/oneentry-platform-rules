# Version history of an entity

The admin panel keeps a short history of snapshots for many kinds of record and can roll a record back to one of them. This document covers which records have a history, which permission each call needs, and what a snapshot and a rollback leave out.

Read it before taking a snapshot or rolling a record back, and when one of these calls answers `400` or `403`.

→ `mcp/docs/api/admins-and-permissions#reading-admin-data-needs-a-permission-too` · `mcp/docs/api/admin-page-scope#what-a-refusal-tells-you`

## The four version history operations

| Operation | Call | Does |
|---|---|---|
| `AdminEntityVersionsController_createVersion` | `POST /entity-versions` with `{"entityName": "products", "entityId": 12}` | stores the record as it is now |
| `AdminEntityVersionsController_getVersions` | `GET /entity-versions/{entityName}/{entityId}` | lists the snapshots, newest first |
| `AdminEntityVersionsController_getVersion` | `GET /entity-versions/{entityName}/{entityId}/{version}` | one snapshot; `404` when that version does not exist |
| `AdminEntityVersionsController_restoreVersion` | `POST /entity-versions/{entityName}/{entityId}/{version}/restore` | overwrites the record with the snapshot |

Only the most recent snapshots are kept per record; older ones drop off as new ones are taken. A rollback overwrites the record in place and is not itself undoable, so take a fresh snapshot first when the current state may be wanted later.

## Which records have a version history

`entityName` takes exactly these values:

`admins` · `attributes_sets` · `blocks` · `discounts` · `forms` · `menus` · `order_statuses` · `orders_storage` · `pages` · `payment_accounts` · `product_statuses` · `products` · `template_previews` · `templates` · `user_groups` · `users` · `users_auth_providers`

Any other value answers `400` on all four operations, for every account. There is no other kind of record to snapshot; do not guess a name.

## Which permission each call needs

Taking a snapshot and rolling back need the **update** permission of that kind of record, the same key as an ordinary update of it. Reading the history needs the read permission only where reading that kind of record is itself gated; elsewhere any signed-in admin may read it.

| `entityName` | Read history | Snapshot and rollback |
|---|---|---|
| `admins` | `admins.get` | `admins.update` |
| `attributes_sets` | `settings.attributesSets.get` | `settings.attributesSets.update` |
| `menus` | `menu.get` | `menu.update` |
| `pages` | `pages.get` | `pages.update` |
| `users` | `users.get` | `users.update` |
| `blocks` | open | `blocks.update` |
| `discounts` | open | `discounts.update` |
| `forms` | open | `forms.update` |
| `order_statuses` | open | `orders.storage.order-status.update` |
| `orders_storage` | open | `orders.storage.update` |
| `payment_accounts` | open | `payments.accounts.update` |
| `product_statuses` | open | `catalog.status.update` |
| `products` | open | `catalog.products.update` |
| `template_previews` | open | `settings.templatePreview.update` |
| `templates` | open | `settings.templates.update` |
| `user_groups` | open | `userGroups.update` |
| `users_auth_providers` | open | `usersAuthProvider.update` |

A read permission never allows a rollback. Without the key the call answers `403` and nothing changes.

## Rolling back a page outside your page scope

An account holding `pages.scope` may snapshot and roll back only the pages inside its branches, exactly as with an ordinary page update. A page outside them answers `403` naming the page id, even though the account holds `pages.update`.

## What a snapshot does not show

The `data` of a snapshot is the record as it was, with secrets removed:

- for `admins`, the password hash and the pending password-reset and e-mail confirmation codes are absent;
- for `users`, the password hash is absent;
- for `payment_accounts` and `users_auth_providers`, stored credentials read as `********`, the same as in an ordinary read of those records.

## What a rollback leaves alone

A rollback restores the content of the record but never its credentials or privileges. For `admins` it keeps the current password, permission map, owner and developer flags, a pending e-mail change and the certificate binding. For `users` it keeps the current password.

To change those, use their own operations — see `mcp/docs/api/admin-account-ownership#who-may-change-permissions`. Rolling back an admin to undo a permission change does nothing to the permissions.
