# Operating rules for the OneEntry Admin API

Read this before your first write. Every rule here has broken a real payload, and each one links to the document that explains it in full.

These rules are short on purpose. When one of them applies to what you are about to do, follow the pointer before you build the body.

→ `mcp/docs/server/doc-map` · `mcp/docs/server/payload-conventions`

## The loop you must follow

1. `cms_guide` once, at the start.
2. `cms_docs_search` for the entity you are about to touch — **before** you write a payload, not after a 400.
3. `cms_api_search` to find the operation, then `cms_api_describe` for its exact shape.
4. `cms_api_call` with `dryRun: true` for anything that mutates, then again with the confirm token if one was issued.

Never invent a path, an operation id or a body key. `cms_api_search` is the only authority on what exists; the documents are the authority on what the values mean.

## Trust the example not the type

The OpenAPI document this catalog is built from contains field types that are not JSON Schema types. `cms_api_describe` normalises what it can and marks the rest `"x-loose": true`, with the original under `x-source-type`.

For a loose field, **the `example` is the contract**. Copy its shape. Client-side validation of your body is advisory only — the instance is the real validator, so a call is never blocked because a loose field could not be checked.

→ `mcp/docs/server/cms-api-describe#loose-fields`

## Content is locale keyed

Titles and descriptive content live under `localizeInfos`, keyed by locale code:

```json
{ "localizeInfos": { "en_US": { "title": "Summer sale" } } }
```

Required when creating a product, and effectively required on pages. Do not hardcode `en_US`: read the active locale codes from the instance first, and write every locale the instance has active if the content is meant to be visible in all of them.

→ `mcp/docs/api/locales`

## Attribute values are two level and locale keyed

`attributesSets` is **two levels deep** — locale, then attribute key:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1" } } }
```

The inner key is `<attribute type>_id<attribute id>`, taken from the attribute set the entity belongs to. Read the set before you build the body.

A flat single-level map is accepted and stored empty. The call answers 201 and the attributes are silently missing, so always read the entity back by id after creating it.

→ `mcp/docs/api/attribute-sets`

## Positions are lexorank or numeric depending on the endpoint

Ordering is a lexorank **string** on parent-scoped Admin operations, and a **number** on flat lists and the Content API. Never sort a lexorank value numerically, and never reorder by patching the field directly — use the dedicated position operations of that entity.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## A read straight after a write can lag

Reading an entity **by id** shows your write immediately. List and search responses may not, for a few seconds.

If a list does not yet show what you just created, re-read by id to confirm it exists. **Never repeat the write** — you will create a duplicate that consumes instance quota and has to be cleaned up by hand.

## A 200 means accepted not applied

Several endpoints take the whole body as one opaque value. A body of the wrong **shape** is then stored as happily as a right one, and the response is still `200` — there is no error to react to.

So confirm a write by its effect: read the entity back **by id** and look at the field you set. If it is missing, your shape was wrong. Do not resend the same body, and do not conclude that the field is unsupported.

The known case: `PUT /attributes-sets/{id}/schema` takes the schema object **itself** as the body. Wrapping it as `{ "schema": … }` answers `200 true` and then stores the wrapper — the set's attributes are replaced by a single key called `schema` and the previous schema is gone. Only a read-back reveals it.

## Prefer marker over id

Blocks, forms, menus, templates, general types and modules are addressed by a `marker` or `identifier` that is stable across instances. A numeric `id` is not: an id taken from one instance is meaningless on another, and `404 Not found` on an id you were given is usually that mistake.

Where an operation accepts either, use the marker.

## Baseline data already exists do not recreate it

Every instance arrives populated: a guest user group with its content-API permissions, the admin modules, the general types, the attribute set types and field types, the locales, the dynamic block types and their default templates, and the singleton settings records.

**List first, create second.** Some duplicates fail loudly, which is harmless. The dangerous ones — user groups, modules, attribute set types, settings records — **succeed silently** and leave a shadow record that nothing references.

→ `mcp/docs/api/baseline-data`

## Never touch these without a human saying so

Mutations under `immutable-settings`, `admins`, `backups`, `modules`, `payments/webhook`, `settings-general`, `system/captcha-keys` and `auth/logout/all-users` are permanently confirm-gated by this server, at every allow level.

The gate is not a suggestion. State what you intend to change, show the human the `target` the dry run returned, and wait for them to say yes in this conversation.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Permissions are checked before the request is sent

Each operation declares the permission it requires, and this server refuses locally when the authenticated admin does not hold it. Nothing is sent, so nothing changed.

A permission refusal means **ask for the grant** and stop. Retrying cannot succeed, and neither can a different operation that needs the same permission.

→ `mcp/docs/api/admins-and-permissions`

## Truncated responses are deliberate

Large responses come back with a `_truncated` envelope reporting what was shown and what the total was. That is this server capping what it hands you, not the API limiting itself.

Do not retry hoping for more. Narrow the request with the operation's own `limit`, `offset` and filter parameters.

→ `mcp/docs/server/response-shaping`

## Operations with a single supported path

For the calls below, one route works and the obvious alternative does not. Use the supported one directly — a different body or a retry on the other route will not help.

- **Create a form** — send the payload wrapped in `newForm`. The OpenAPI document shows the fields unwrapped; the wrapped form is the one the endpoint accepts.
- **Replace an attribute set schema** — send the schema object itself, never wrapped in `{ "schema": … }`. The wrapped form answers 200 and destroys the existing schema.
- **Update a product** — always include `blocks` (send `blocks: []` when you have nothing to set), and never include `forms`.
- **Read locale codes** — use `AdminLocalesController_findAllActive`.
- **List products** — the list operation is `POST /products/all`; there is no `GET /products`.
- **Export operations** — `orders.export`, `payments.export` and `users.export` are not grantable on current instances. Treat them as unavailable and tell the human rather than retrying.

If a call outside this list answers 5xx, stop and report it with the operation id and the request you sent. Do not retry it with a modified body.

## Where to look next

With the operation hints removed from `cms_api_describe`, `cms_docs_search` and the map below are how you find anything.

- `mcp/docs/server/doc-map` — every document, with when to read it
- `mcp/docs/server/payload-conventions` — the rules above, in full
- `mcp/docs/api/baseline-data` — what already exists on your instance
- `mcp/docs/server/errors-and-refusals` — what a specific error means and what to do next
