# Operating rules for the OneEntry Admin API

Read this before your first write. Every rule here has broken a real payload; each links to the document explaining it in full. When one applies to what you are about to do, follow the pointer before you build the body.

→ `mcp/docs/server/doc-map` · `mcp/docs/server/payload-conventions`

## The loop you must follow

1. `cms_guide` once, at the start.
2. `cms_docs_search` for the entity you are about to touch — **before** the payload, not after a 400.
3. `cms_api_search` for the operation, then `cms_api_describe` for its shape.
4. `cms_api_call` with `dryRun: true` for anything that mutates, then again with the confirm token if one was issued.

Never invent a path, an operation id or a body key. `cms_api_search` is the only authority on what exists.

## Trust the example not the type

The API document carries field types that are not JSON Schema types. `cms_api_describe` normalises what it can and marks the rest `"x-loose": true`.

For those, **the `example` is the contract** — copy its shape. Validation here is advisory; the instance is the real validator, so a call is never blocked over a field we could not check. A field flagged `x-example-mismatch` contradicts its own type, and the example wins there too.

→ `mcp/docs/server/cms-api-describe#loose-fields`

## Content is locale keyed

Titles and descriptive content live under `localizeInfos`, keyed by locale code:

```json
{ "localizeInfos": { "en_US": { "title": "Summer sale" } } }
```

Required on a product, effectively required on a page. Do not hardcode `en_US`: read the active locales with `AdminLocalesController_findAllActive` and write every one the content is meant to appear in.

→ `mcp/docs/api/locales`

## Attribute values are two level and locale keyed

`attributesSets` is **two levels deep** — locale, then attribute key:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1" } } }
```

The inner key is `<attribute type>_id<attribute id>`, read from the entity's attribute set. A flat one-level map is accepted, answers 201 and stores nothing — so read the entity back by id after creating it.

→ `mcp/docs/api/attribute-sets`

## Positions are lexorank or numeric depending on the endpoint

Ordering is a lexorank **string** on parent-scoped Admin operations and a **number** on flat lists and the Content API. Never sort a lexorank numerically, and never reorder by patching the field — use that entity's position operation.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## A read straight after a write can lag

Reading an entity **by id** shows your write immediately. Lists and searches may not, for a few seconds.

If a list does not show what you just created, re-read by id. **Never repeat the write** — you get a duplicate that consumes quota and has to be cleaned up by hand. And never swallow a failed read into an empty result: "empty" and "malformed" then look alike, and the next run recreates everything.

## A 200 means accepted not applied

Several endpoints take the body as one opaque value, so a wrong **shape** is stored as happily as a right one and the answer is still `200`.

Confirm a write by its effect, and **through the read its consumer uses**. The raw record echoes your input back, wrong shape included, while the projection a site receives shows nothing — verifying through the endpoint you wrote to proves little.

→ `mcp/docs/api/silent-no-ops`

## An omitted field can mean clear it

Most updates merge. A few apply an omitted field as **"set it to nothing"**, and still answer `200`: a page without `parentId` moves to the root, a block without `blockPages` detaches from every page, a menu item without its parent reference flattens, a user without `formData` loses it.

Products, `generalTypeId` and `attributeSetId` merge, so "PUT always replaces" is the wrong lesson. Read, change what you meant to, send it back whole — then check the fields that were **not** in your body.

→ `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it`

## Prefer marker over id

Blocks, forms, menus, templates, general types and modules are addressed by a `marker` or `identifier` stable across instances. A numeric `id` is not, and a `404` on an id you were given is usually that. Where an operation accepts either, use the marker.

## Baseline data already exists do not recreate it

Every instance arrives populated: user groups, modules, general types, attribute set and field types, locales, block types with their default templates, the singleton settings.

**List first, create second.** Some duplicates fail loudly, which is harmless. The dangerous ones — user groups, modules, attribute set types, settings — **succeed silently**.

Two lists are the exception and start **empty**: product statuses and template previews. Nothing seeds them, nothing reports their absence, and without them no product is sellable and no upload gets a preview. There, create.

→ `mcp/docs/api/baseline-data`

## Never touch these without a human saying so

Mutations on the instance's own configuration — admins, modules, backups, settings — are permanently confirm-gated at every allow level. `cms_guide` prints the exact list.

The gate is not a suggestion. State what you intend to change, show the human the dry run's `target`, and wait for a yes here.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Permissions are checked before the request is sent

Each operation declares the permission it needs, and this server refuses locally when the admin does not hold it. Nothing is sent, so nothing changed. A refusal means **ask for the grant** and stop — no retry and no sibling operation gets past it.

→ `mcp/docs/api/admins-and-permissions`

## Truncated responses are deliberate

A large response comes back with a `_truncated` envelope reporting what was shown and what the total was. That is this server capping what it hands you, not the API. Do not retry hoping for more — narrow the request with the operation's own `limit`, `offset` and filters.

→ `mcp/docs/server/response-shaping`

## Operations with a single supported path

One route works and the obvious alternative does not. Use it directly.

- **Create a form** — wrap in `newForm`, with `type` (`data` for a contact form) though the schema omits it.
- **Replace an attribute set schema** — send the schema object itself. Wrapped as `{ "schema": … }` it answers 200 and destroys it.
- **Update a product** — include `blocks` (`[]` if nothing to set), never `forms`.
- **Create a menu** — with `pagesIds: []`, attaching pages later. Non-empty on create answers 500.
- **Set a product status** — `statusId` in the product update. Bulk `set-status` takes it in a field named `id`, and given `statusId` it nulls the status and answers `201 true`.

## List products and other calls whose input is split

`POST /products/all` is the only way to list products, and its input is split: **paging and `langCode` go in the query, the body is an array of filters** — `[]` for none. Sent in the body they are ignored, and the 400 blames `langCode` for a value you never sent.

Copy `example` from `cms_api_describe` whole: separate `params` and `body` schemas do not assemble into an obvious call, and `example` is already one.

A 5xx outside these two lists means stop and report it, with the operation id and the request.

## Where to look next

`mcp/docs/server/doc-map` lists every document with a reason to read it. The corpus is **English** — search it in English whatever language you answer in.
