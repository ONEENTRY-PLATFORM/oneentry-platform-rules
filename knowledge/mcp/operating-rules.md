# Operating rules for the OneEntry Admin API

Read this before your first write. Every rule here has broken a real payload, and each links to the document explaining it. When one applies, follow the pointer before you build the body.

→ `mcp/docs/server/doc-map` · `mcp/docs/server/payload-conventions`

## The loop you must follow

1. `cms_guide` once, at the start.
2. `cms_docs_search` for the entity you are about to touch — **before** the payload, not after a 400.
3. `cms_api_search` for the operation, then `cms_api_describe` for its shape.
4. `cms_api_call` with `dryRun: true` for anything that mutates, then again with the confirm token. Files go through `cms_upload_file` or `cms_import_file_from_url`.

Never invent a path, an operation id or a body key. `cms_api_search` is the only authority on what exists.

## Trust the example not the type

The API document carries field types that are not JSON Schema types. `cms_api_describe` normalises what it can and marks the rest `"x-loose": true`, and for those **the `example` is the contract**. A field flagged `x-example-mismatch` contradicts its own type, and the example wins there too; a `curatedBody` beats both.

→ `mcp/docs/server/cms-api-describe#loose-fields`

## Content is locale keyed

Titles and descriptive content live under `localizeInfos`, keyed by locale code:

```json
{ "localizeInfos": { "en_US": { "title": "Summer sale" } } }
```

Required on a product, effectively required on a page. Never hardcode `en_US`: read the active locales and write every one the content is meant to appear in. The one structure that is **not** locale keyed is an option's extra value.

→ `mcp/docs/api/locales`

## Attribute values are two level and locale keyed

`attributesSets` is **two levels deep** — locale, then attribute key:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1" } } }
```

The inner key is `<attribute type>_id<attribute id>`, from the entity's attribute set. A flat one-level map is accepted, answers 201 and stores nothing — read the entity back by id.

→ `mcp/docs/api/attribute-sets`

## Positions are lexorank or numeric depending on the endpoint

Ordering is a lexorank **string** on parent-scoped Admin operations and a **number** on flat lists and public reads. Never sort a lexorank numerically, never reorder by patching the field, and never send a string one back to an update.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## A read straight after a write can lag

An **admin** read by id shows your write immediately; admin lists and **every public read**, one entity included, lag by seconds. So re-read after a pause and **never repeat the write** — that makes a duplicate somebody cleans up by hand. Never swallow a failed read into an empty result either: the next run then recreates everything.

## A 200 means accepted not applied

Several endpoints take the body as one opaque value, so a wrong **shape** is stored as happily as a right one and the answer is still `200`. Confirm a write by its effect, **through the read its consumer uses** — the raw record echoes your input back, wrong shape included.

→ `mcp/docs/api/silent-no-ops`

## An omitted field can mean clear it

Most updates merge. Two still apply an omitted field as **"set it to nothing"** and answer `200`: a menu item flattens to the top level, a form loses its bindings and their submissions. An explicitly empty value — `[]` or `null` — always means what it says.

Read, change what you meant to, send it back whole — then check the fields that were **not** in your body.

→ `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it`

## Prefer marker over id

Blocks, forms, menus, templates, general types and modules carry a `marker` or `identifier` stable across instances. A numeric `id` is not, and a `404` on an id you were handed is usually that. Where both are accepted, use the marker.

## Baseline data already exists do not recreate it

Every instance arrives populated: user groups, modules, general types, attribute set and field types, locales, block types, the singleton settings. **List first, create second** — the dangerous duplicates (user groups, modules, attribute set types, settings) succeed silently.

Two lists start **empty**: product statuses and template previews. Nothing reports their absence, yet without them no product is sellable and no upload gets a preview. There, create.

→ `mcp/docs/api/baseline-data`

## Never touch these without a human saying so

Mutations on the instance's own configuration — admins, modules, backups, settings — are confirm-gated at every allow level, and `cms_guide` prints the list. State what you intend to change, show the dry run's `target`, wait for a yes.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Permissions are checked before the request is sent

Most operations declare the permission they need, and this server refuses locally when the admin lacks it. Some declare none — admin accounts among them — and there the `403` comes from the instance. **Ask for the grant**; retrying cannot work.

→ `mcp/docs/api/admins-and-permissions`

## Truncated responses are deliberate

A large response comes back with a `_truncated` envelope saying what was shown and what the total was — this server capping the answer, not the API. Do not retry for more; narrow the request with the operation's `limit`, `offset` and filters.

→ `mcp/docs/server/response-shaping`

## Operations with a single supported path

One route works and the obvious alternative does not.

- **Create a form** — wrapped in `newForm`, with `type`; nothing else reports it missing.
- **Replace an attribute set schema** — the schema object itself. Wrapped as `{ "schema": … }` it answers 200 and destroys it.
- **Update a product** — never send `forms`; the schema takes it, the save does not.
- **Create a menu** — `pagesIds` is a flat set; nesting and labels come later.
- **Set a product status** — `statusId`, in the product update or in bulk `set-status`.
- **Upload a file** — `cms_upload_file` or `cms_import_file_from_url`; `cms_api_call` sends JSON only.

## List products and other calls whose input is split

`POST /products/all` is the only way to list products, and its input is split: **paging and `langCode` in the query, the body an array of filters** — `[]` for none. Sent in the body they are ignored, and the 400 blames `langCode` for a value you never sent. Copy `example` from `cms_api_describe` whole, and prefer `curatedBody` where it appears.

A 5xx outside these two lists means stop and report it, with the operation id and the request.

## Reading it back is not always verifying

Two cases where the habit is not enough:

- **A batch write** can miss one entity while every response reports success. Re-read **all** of them — for products, by ids in one call — and retry the mismatches. Calculated values such as ratings arrive after a delay: wait, then check again.
- **A field that exists for the admin panel** — an option's extra value, a field setting — comes back from every read exactly as sent, while the panel still shows it empty. Get a human to look, or report the check as incomplete and say what is unverified.

→ `mcp/docs/api/bulk-content-migration#panel-facing-fields-cannot-be-verified-by-reading`

## Where to look next

`mcp/docs/server/doc-map` lists every document with a reason to read it, and `mcp/docs/api/content-modelling` covers where content should go. The corpus is **English** — search it in English whatever language you answer in.
