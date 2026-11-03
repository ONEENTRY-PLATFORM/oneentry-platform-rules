# Reading the data of an interface module

The admin panel reads what an interface module shows by sending the module's `scope` values to ordinary listing operations, which accept the saved conditions as they are. An agent can do the same to see exactly what a module's admin sees, or to export it.

Read this when answering "what does this module show", when exporting a module's records, or when one of these listings answers `400`.

→ `mcp/docs/api/interface-module-scopes#what-the-scope-of-each-base-holds` · `mcp/docs/api/interface-modules`

## Reading what a module shows

| Base | Operation | Body |
|---|---|---|
| catalogue | `POST /products/page/{id}` | the module's `scope.filter` |
| catalogue | `POST /products/counts-by-pages` | `{ pageIds, filter }`; answers product counts keyed by page id |
| catalogue | `POST /products/quick/search` | `{ pageIds, isNested, filter, id }` |
| pages | `POST /pages/paginated` | `{ types, filter }` with `parentId` in the query |
| pages | `POST /pages/quick/search` | `{ pageIds, isNested, types, filter, id }` |
| orders | `POST /orders-storage/orders/paginated` | `{ storageIds, filter }` |
| orders | `POST /orders/refund-requests/paginated` | `{ storageIds, filter }`; pending refund requests |
| payments | `POST /payments/sessions/paginated` | `{ paymentAccountIds, storageIds, filter }` |
| users | `POST /users/paginated` | `{ groupIds, authProviderIds, filter, search }` |

Paging and `langCode` stay in the query.

→ `mcp/docs/api/page-listing-filters#narrowing-one-level-by-type-or-condition`

## An empty filter answers 400

`filter: []` answers `400 filter must be a non-empty array of conditions or null` on the pages, orders, payments and users operations. The product operations read `[]` as no conditions. To say "no conditions" everywhere, leave the `filter` key out.

## An id that no longer exists returns nothing

On the orders, payments and users operations, a storage, account, group or provider id that no longer exists narrows to nothing rather than answering `400`. An empty result after a module was edited is worth checking against the current ids before concluding there is no data.

The payment session listing needs at least one of `paymentAccountIds` and `storageIds`; an empty body answers `400`.

## What the paginated listings return

- **Payment sessions** — the same rows and order as `GET /payments/sessions`.
- **Users** — the same rows and order as `POST /users/all`; a user in two listed groups is one row. `search` is a case-insensitive part of the login. The conditions read the user form in the request's `langCode`, so a user with no values in that language passes only `nexs`.
- **Orders** — the order list of the listed storages.

## Exporting the records a module shows

| Operation | Permission | Answer |
|---|---|---|
| `POST /export/orders` | `orders.export` | a zip with one file per storage |
| `POST /export/payments` | `payments.export` | a zip with one file per payment account that has matching sessions, plus `_no-payment-account` for sessions whose account was deleted |
| `POST /export/users` | `users.export` | a zip with one file per auth provider that has matching users |

Each takes the same body as its listing and a required `format` query parameter, `csv` or `xml`; without it the call answers `400`. An export with nothing to match answers an empty zip, not an error.

## Creating a user inside a users module

Send `groupIds` to `POST /users-auth-providers/{id}/users/sign-up`. The user and the groups are saved together, on top of the auth provider's own group. An unknown group answers `400` and no user is created, so a failed call needs no clean-up.

## Common mistakes

- **Sending `filter: []` to a pages, orders, payments or users listing.** Leave the key out instead.
- **Reading an empty list as "no data" after a module changed.** Check that its ids still exist.
- **Exporting without `format`.** It answers `400`; send `csv` or `xml`.
- **Expecting a user listing in another language to match.** Conditions read the form in the request's `langCode`.

→ `mcp/docs/api/form-submission-scope-filters#keys-that-narrow-submissions-to-a-module` · `mcp/docs/api/users-and-groups`
