# Interface module scopes

The `config.scope` of an interface module says which part of its base section the module shows: catalogue categories, page branches, order storages, payment accounts, or user groups, narrowed by a saved filter. This document lists what each base accepts and how its conditions match.

Read it when building or changing a module's `config`, or when a module shows fewer items than expected.

→ `mcp/docs/api/interface-modules#an-update-replaces-config-whole` · `mcp/docs/api/interface-module-data#reading-what-a-module-shows`

## What the scope of each base holds

Every base has `scope`; `scope.filter` is `null` or a non-empty list of up to 20 conditions, all of which must pass. `filter: []` answers `400` on every base — send `null` for no conditions.

| Base | Required in `scope` |
|---|---|
| `catalog` | `categoryIds`: distinct catalogue page ids, at least one |
| `pages` | `pageIds`: distinct ids of existing pages of any type, the roots of the module's branches |
| `orders` | `storageIds`: distinct ids of existing order storages |
| `payments` | at least one of `paymentAccountIds` and `storageIds` |
| `users` | at least one of `groupIds` and `authProviderIds` |

An id that is not of the right kind answers `400` naming it — a common page in `categoryIds`, for instance.

## Catalogue scope

`scope.sort` is `null` or `{ key, order }` with the `sortKey` and `sortOrder` values of the product listing. `scope.filter` holds product-filter conditions in the body shape of the category product listing.

A `pat` condition in a catalogue filter is read as a pattern in which `*` is any one character and `_` a whitespace, and every other character keeps its pattern meaning. A mask that is not a valid pattern, such as `(`, answers `400` naming the condition when the module is saved.

→ `mcp/docs/api/products#filtering-a-product-list`

## Pages scope

`scope.pageTypes` is `null` for all types or a non-empty list of page types. `scope.filter` conditions are `{ attributeMarker, conditionMarker, conditionValue }` or `{ isVisible }`. In a `pat` mask only `*` (any character) and `_` (a space) are special.

A page that fails the types or the filter hides its whole branch. Types match exactly: with `pageTypes: ["catalog_page"]`, a common page inside a catalogue branch hides the catalogue pages below it, and a common root gives an empty module. List both types to keep such a branch.

## Orders scope

`scope.filter` conditions are `{ statusIdentifiers }` — statuses of the module's storages — or `{ paymentAccountIds }` — payment accounts linked to at least one of them.

A status condition restricts an order only on the axes of its own storage that have a listed status. On such an axis the order's status must be one of them, and an order with no status there fails. A storage none of whose statuses are listed is not restricted. In `basic` mode a storage has one status per order, the order status.

→ `mcp/docs/api/order-statuses`

## Payments scope

When both `paymentAccountIds` and `storageIds` are set, a payment session must pass both, and a session with no order never passes `storageIds`. Each `scope.filter` condition is exactly one of:

| Condition | Values |
|---|---|
| `{ statuses }` | `waiting`, `completed`, `canceled`, `expired` |
| `{ dateField, lastDays }` | `dateField` is `createdDate` or `updatedDate`; `lastDays` from 1 to 36500, counted back from the request |
| `{ dateField, from, to }` | ISO 8601 moments, at least one; a date without a time is midnight UTC |
| `{ amountFrom, amountTo }` | numbers from 0, at least one |

Bounds are inclusive. A session with no amount or no date fails the matching condition.

## Users scope

A group counts together with all its subgroups at any depth. `scope.filter` conditions are `{ attributeMarker, conditionMarker, conditionValue }` on the attributes of the user form, with the `conditionMarker` values of the user listing; `conditionValue` may be omitted.

Inside a module three markers mean something different from `POST /users/all`:

| Marker | Inside a module | In `POST /users/all` |
|---|---|---|
| `same` | the value matches at least one other user within the module, archived users included; for a choice, the set of selected values regardless of order | the value is set |
| `same_part` | the same, comparing the value without its last parts; `conditionValue` is `<separator>\|<count>`, default `-\|2` | the attribute is present |
| `pat` | only `*` and `_` are special | a pattern, `*` any one character, `_` a whitespace |

An empty value (`""` or `[]`) counts as not filled for `exs`, `nexs` and `same` on every attribute type.

→ `mcp/docs/api/filters#comparisons-follow-the-attribute-type`

## Panel settings of each base

None of these restrict an operation; they are what the panel offers inside the module.

- **Catalogue.** `productCategories.mode` is `module` (the default), `all` or `list`, with `categoryIds` only for `list`. `productDefaults` is `null` or any of `statusId`, `templateId` (a product template), `attributeSetId` (a `forProducts` set) and `isVisible`. `productStatusIds` is `null` or distinct product status ids, and must contain `productDefaults.statusId` when both are set.
- **Pages.** `pageDefaults` is `null` or `parentId` (a root or a page below one), `pageType` (one of `pageCreateTypes` when that is set), `isVisible`, a page `templateId` and a `forPages` `attributeSetId`. `pageCreateTypes` is `null` for all types or distinct types; it is independent of `scope.pageTypes`, so a page created in a module may not appear in it. `pageDeleteFlags` is `null` for all delete options or a list of them, `[]` for none.
- **Orders.** `orderStatusIdentifiers` is `null` for all, `[]` for none, or statuses of the module's storages that the panel offers to set.
- **Payments.** `orderLinkModuleId` is `null` or a positive integer naming the orders module the panel links to; whether it exists is not checked.
- **Users.** `newUserGroupIds` is `null` or distinct existing group ids the panel adds new users to.

## Common mistakes

- **Sending `filter: []` for no conditions.** It answers `400`; send `null`.
- **Narrowing pages by one type when branches mix types.** The unlisted type hides everything below it.
- **Copying a `pat` mask between bases.** Catalogue masks are patterns; pages and users treat only `*` and `_` as special.
- **Expecting a session with no order to pass `storageIds`.** It never does.

→ `mcp/docs/api/interface-modules#settings-that-only-change-what-the-panel-offers` · `mcp/docs/api/users-and-groups`
