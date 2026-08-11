# Glossary

Terms that appear in payloads, tool output and error messages, with the one thing about each that catches people out. If a word in a response is unfamiliar, look here first.

Entries are alphabetical within each group.

→ `mcp/docs/server/payload-conventions` · `mcp/docs/api/baseline-data`

## Terms about this server

**allow level** — the server's write policy: `read`, `write` or `destructive`. Set by the operator at startup; no tool argument can raise it.

**audit log** — a JSONL file recording every non-`GET` call: who, what operation, which outcome. Arguments are hashed, not stored.

**catalog** — the list of operations this server can call, built at runtime from the instance's own API document. If it is empty, nothing can be called.

**confirm token** — a 24-character single-use token, valid five minutes, bound to the exact call. Required for deletes and for the permanently gated paths.

**dry run** — `dryRun: true`. Returns the resolved request, the policy decision and the current state of the target, without sending anything.

**instance** — one deployment of the OneEntry platform. Ids are not portable between instances; markers are.

**loose field** — a body field whose declared type is not a JSON Schema type, marked `"x-loose": true`. Its `example` is the contract.

**operation id** — the identifier you pass to `cms_api_call`, for example `AdminPagesController_findAllRoot`. Comes from `cms_api_search` and is never constructed by hand.

**operator** — the person who runs the server and holds the credentials. Changing the allow level or granting a permission is their action, not yours.

**risk** — an operation's classification, derived from its HTTP method: `read`, `write` or `destructive`.

**target** — in a dry run or confirmation response, the current state of the object about to change, fetched with the sibling `GET`.

**truncated** — the server clipped a response to fit the size cap. A successful call, not an error.

## Terms about the API

**attribute set** — the schema attached to an entity: which attributes it has, of which types. Read it before building an attribute payload.

**attribute set type** — which audience a set belongs to, such as products or pages. Provisioned with the instance; never create one.

**attributesSets** — the payload field holding attribute **values**, keyed by locale and then by `<type>_id<attribute id>`. Two levels, always.

**attributeValues** — the same values as they come back in a response, resolved for reading.

**baseline data** — everything provisioned with an instance: the guest group and its permissions, modules, general types, attribute set types, field types, locales, block types, default templates, settings records. List first, create second.

**block** — a reusable content unit attached to pages. Addressed by marker.

**Content API** — the public read API a website uses. Not exposed through this server.

**Developer API** — a separate surface, not exposed through this server. Accounts flagged as developer accounts are rejected by the Admin API.

**general type** — the classification of a page, block, product, form, order or discount. Provisioned, resolved by name, and unique.

**guest group** — the user group with identifier `guest`, normally id 1, that every Content API permission is attached to. Unauthenticated access is decided by its rules.

**identifier** — a stable string name on templates, menus, order statuses, modules and groups. Interchangeable in practice with *marker*, which is the word used on blocks, forms and general types.

**langCode / locale code** — a code such as `en_US`. Read the active ones from the instance; never hardcode.

**lexorank** — the string ordering scheme used by parent-scoped position fields. Sorts as text; never sort it numerically or patch it directly.

**localizeInfos** — the payload field holding titles and descriptive content, keyed by locale code.

**marker** — a stable string identifier that means the same thing on every instance. Prefer it to a numeric id.

**module** — an admin panel section such as `catalog` or `orders`. Eighteen are provisioned; mutations are permanently confirm-gated.

**permission** — a dotted string such as `menu.update` that an operation requires and an admin either holds or does not. Checked locally before the request is sent.

**position** — an ordering value: a lexorank string on parent-scoped Admin operations, a number on flat lists and the Content API.

**template** — a rendering definition referenced by blocks and pages. Identifiers are unique; the dynamic block types each ship one named `<block_type>_default`.

## Two pairs worth keeping straight

**attributesSets vs attributeValues** — the first is what you send, the second is what you read. Different shapes, same underlying data.

**id vs marker** — an id is a number that is meaningful on exactly one instance. A marker is a string that is meaningful on all of them. A 404 on an id someone gave you is usually this distinction.
