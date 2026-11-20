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

Taking a snapshot and rolling back need the **update** permission of that kind of record, the same key as an ordinary update of it. Reading the history always needs a key as well: the read permission of that kind of record, or, where that kind has no read permission of its own, its update permission. No history is readable by a signed-in admin without one.

| `entityName` | Read history | Snapshot and rollback |
|---|---|---|
| `admins` | `admins.get` | `admins.update` |
| `attributes_sets` | `settings.attributesSets.get` | `settings.attributesSets.update` |
| `blocks` | `blocks.get` | `blocks.update` |
| `discounts` | `discounts.get` | `discounts.update` |
| `forms` | `forms.get` | `forms.update` |
| `menus` | `menu.get` | `menu.update` |
| `order_statuses` | `orders.storage.get` | `orders.storage.order-status.update` |
| `orders_storage` | `orders.storage.get` | `orders.storage.update` |
| `pages` | `pages.get` | `pages.update` |
| `payment_accounts` | `payments.accounts.get` | `payments.accounts.update` |
| `product_statuses` | `catalog.status.update` | `catalog.status.update` |
| `products` | `catalog.products.get` | `catalog.products.update` |
| `template_previews` | `settings.templatePreview.update` | `settings.templatePreview.update` |
| `templates` | `settings.templates.update` | `settings.templates.update` |
| `user_groups` | `userGroups.get` | `userGroups.update` |
| `users` | `users.get` | `users.update` |
| `users_auth_providers` | `usersAuthProvider.get` | `usersAuthProvider.update` |

A read permission never allows a rollback. Without the key the call answers `403` and nothing changes.

## Rolling back a page outside your page scope

An account holding `pages.scope` may snapshot and roll back only the pages inside its branches, exactly as with an ordinary page update. A page outside them answers `403` naming the page id, even though the account holds `pages.update`.

## Rolling back a product or order status

A rollback of `product_statuses` or `order_statuses` follows the same rules as an ordinary update of that status:

- **The default flag is not rolled back.** `isDefault` keeps its current value, because only one status may be the default of a catalog or of an order storage axis. To change the default, update the status itself.
- **A marker held by another status is refused.** When the snapshot's `identifier` now belongs to a different status, the rollback answers `400` with the same message as an update onto that marker (`Product's status with this identifier already exists` / `Order's status with this identifier already exists`), and nothing changes. Rename or delete the other status first.
- **Status maps follow the restored marker.** When rolling back an order status brings back an older `identifier`, every trigger of that storage's status map that pointed at the current marker is moved to the restored one, exactly as an update that renames the status does.

→ `mcp/docs/api/order-status-triggers#where-the-status-maps-live` · `mcp/docs/api/product-statuses#what-making-a-status-the-default-changes`

## Rolling back an order storage

A snapshot of `orders_storage` also records which payment accounts the storage accepts, and a rollback brings that set back together with the storage's own settings. After the rollback, `paymentAccountIds` in a read of the storage matches the snapshot, exactly as if the storage had been updated with that list.

- A payment account deleted after the snapshot was taken is skipped; the others are linked again.
- Some older snapshots carry no account list. Rolling back to one of them leaves the current accounts in place, so read the storage after the rollback.

→ `mcp/docs/api/version-history#what-a-rollback-leaves-alone`

## What a snapshot does not show

The `data` of a snapshot is the record as it was, with secrets removed:

- for `admins`, the password hash and the pending password-reset and e-mail confirmation codes are absent;
- for `users`, the password hash is absent;
- for `payment_accounts` and `users_auth_providers`, stored credentials read as `********`, the same as in an ordinary read of those records.

## What a rollback leaves alone

A rollback restores the content of the record but never its credentials or privileges. For `admins` it keeps the current password, login, e-mail address, permission map, owner and developer flags, a pending e-mail change and the certificate binding. For `users` it keeps the current password.

Stored secret keys of an integration are merged, not overwritten. For `payment_accounts` (`settings` and `testSettings`) and `users_auth_providers` (`config`), a secret the snapshot does not hold keeps its current value, and a secret the snapshot does hold comes back with it. Every other field of those settings returns to its snapshot value. Rolling one of these records back to a snapshot taken before a key was entered therefore keeps the key, while older values such as a shop id or a return URL do come back, so read the settings after the rollback.

The project owner's own record can be rolled back only by the owner. Any other account gets `403` on `AdminEntityVersionsController_restoreVersion` for it, even with `admins.update`, the same as an ordinary update of the owner's record.

To change those, use their own operations — see `mcp/docs/api/admin-account-ownership#who-may-change-permissions`. Rolling back an admin to undo a permission change does nothing to the permissions.
