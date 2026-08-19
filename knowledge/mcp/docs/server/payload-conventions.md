# Payload conventions of the Admin API

The wire-format rules that decide whether a body is accepted, silently ignored, or rejected. These apply across entities, so they live here once and every entity document points at them.

If you read one document before your first write, make it `mcp/operating-rules`; this is the same material with the detail restored.

→ `mcp/operating-rules` · `mcp/docs/api/attribute-sets`

## localizeInfos is keyed by locale code

Human-readable content sits under `localizeInfos`, keyed by locale:

```json
{ "localizeInfos": { "en_US": { "title": "Summer sale", "content": "…" } } }
```

Required when creating a product, and effectively required on pages — an entity with no localized title is unusable in the admin panel even where the API accepts it.

Write every locale the instance has active. Content written for only one locale appears blank to anyone using another, and there is no fallback.

## Get the active locale codes first

Never hardcode `en_US`. Read the active codes with `AdminLocalesController_findAllActive` and build the object from that list.

An instance carries every locale code in existence but activates only a few, so an instance that looks English-only today may have three active locales tomorrow, and content written against a hardcoded key silently stops being complete.

→ `mcp/docs/api/locales`

## attributesSets is two levels deep

Attribute values are locale-keyed **and** attribute-keyed:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1", "integer_id43": 7 } } }
```

The outer key is the locale code. The inner key is `<attribute type>_id<attribute id>` — the field type and the numeric id of the attribute, taken from the attribute set the entity belongs to.

You cannot guess the inner keys. Read the attribute set first, and build them from what it returns.

## The flat map that silently does nothing

A single-level map is the most expensive mistake in this API:

```json
{ "attributesSets": { "string_id42": "SKU-1" } }
```

This is accepted. The call answers 201, the entity is created, and its attributes are **empty**. Nothing warns you.

The defence is a habit, not a check: after creating or updating an entity with attributes, read it back by id and confirm the values are there. If they are not, the nesting was wrong.

## attributeSetId and the audience type id

Two different numbers, easy to confuse:

- `attributeSetId` — which attribute set this entity uses. Read it from an existing entity of the same kind, or list the sets.
- the attribute set **type** id — which audience the set belongs to, for instance products or pages. It is instance data, not a constant, and there is no lookup you can hardcode against.

Resolve both against the instance. An id copied from another instance points somewhere else or nowhere.

→ `mcp/docs/api/attribute-sets`

## One structure that is not locale keyed

The locale-first rule has one exception worth memorising, because it is inside a structure that follows the rule everywhere else. The **extra value of a list option** is flat:

```json
{ "title": "Cherry", "value": "cherry", "position": 1,
  "extended": { "type": "string", "value": "#d11241" } }
```

`listTitles` is keyed by locale. The option inside it is not, and `extended` carries `type` and `value` directly. Adding a locale level by analogy is accepted, reads back exactly as written, and shows as an empty field in the admin panel.

The general lesson is the one to carry: a field that appears in neither the operation schema nor this corpus is not safely inferred from its neighbour. Find it documented, or verify it by looking at the result.

→ `mcp/docs/api/list-options-and-extra-values`

## position is a lexorank string or a number

Ordering comes in two forms, and which one you get depends on the endpoint:

| Where | Form |
|---|---|
| Parent-scoped Admin operations | a **lexorank string** such as `0|i0000n:` |
| Flat Admin lists and the Content API | a **number** |

Two rules follow. Never sort a lexorank value numerically — it sorts as text, and `0|i0000n:` versus `0|hzzzzz:` is meaningful only as a string comparison. And never reorder by patching the field: each entity that supports ordering has a dedicated position operation, and using it is the only way the surrounding items get renumbered correctly.

Sending the read form straight back is safe on a page update: the string is accepted and ignored, so read-modify-write does not have to strip a field it never asked for. It still does not reorder anything — that is the position operation's job.

## marker is portable id is not

Blocks, forms, menus, templates, general types and modules carry a `marker` or `identifier` that means the same thing on every instance. A numeric `id` does not.

Prefer the marker-based operation whenever one exists. A `404 Not found` on an id somebody handed you is, more often than not, an id from a different instance.

## x loose fields where the example is the contract

`cms_api_describe` marks some body fields `"x-loose": true`, with the original declared type under `x-source-type`. Those types are not JSON Schema types and could not be normalised.

For those fields, copy the shape from the schema's `example`. Do not infer structure from the type name, and do not expect the server to validate it for you — client-side body validation here is advisory, so a wrong shape reaches the instance untouched.

`localizeInfos` and `attributesSets` are usually both in this list, which is exactly why the two rules above exist.

## Writes are eventually consistent

Reading an entity **by id** reflects your write immediately. Lists and search responses may not, for a few seconds.

```text
POST  /products …            → 201, id 512
POST  /products/all …        → 512 is not in the list yet
GET   /products/512          → there it is
```

That is normal, not a failure.

## An omitted field can mean clear it

Most updates merge: a field you leave out is left as it was. **Some do not**, and on those an omitted field is applied as "set it to nothing". Nothing in the schema distinguishes the two, and both answer `200`.

The known cases, all of them relationship fields:

| entity | field | what omitting it does |
|---|---|---|
| menu item | parent reference | flattens the item to the top level |
| form | `formModuleConfigs` | deletes every binding and the submissions made against them |
| user | `formData`, `notificationData` | replaces them with empty values |

Everything else merges — products, pages, blocks, `generalTypeId`, `attributeSetId`. The rule is not "PUT always replaces", and treating it that way makes you resend fields you have not read and get wrong.

Where a field merges, an **explicitly empty** value still means what it says: `blockPages: []` detaches a block from every page, and a `parentId` of `null` moves a page to the root. Absent and empty are two different instructions.

The habit that covers all of it: **read the entity, change what you meant to change, send it back whole**. Then read it again and check the fields that were *not* in your body. A menu that reorganised itself and a form whose submissions disappeared are the same one cause.

→ `mcp/docs/api/menus` · `mcp/docs/api/forms-and-form-data` · `mcp/docs/api/silent-no-ops`

## Verify with the read your consumer uses

An entity can often be read more than one way, and the ways do not always agree. The raw record shows what you wrote. The projection a site or the admin panel reads shows what it will actually get, after locale resolution and shaping.

Checking your write through the endpoint you wrote to therefore proves very little: you see your own input echoed back, including input that is the wrong shape and that no consumer will ever read.

So when a document names a *consumer* read — `/forms/marker/{marker}` for form fields, the listing operation for a product status — verify through that one. `cms_api_describe` names it under `verifyWith` where the pairing is known.

→ `mcp/docs/api/silent-no-ops` · `mcp/docs/api/verification-recipes`

## Read the entity back after creating it

One habit that covers three separate traps: the silent flat-attribute map, the eventual-consistency gap, and any field the instance normalised differently from what you sent.

Create, then `GET` by the id from the response, then check the fields you cared about. If something is missing, fix the payload and **update** the entity — do not create it again. A repeated create is how duplicates get made, and duplicates consume the instance's record quota.

## Narrowing a list instead of asking for everything

Responses are capped at 24 KB by default and come back with a `_truncated` envelope when they exceed it. Retrying does not raise the cap.

Use the operation's own parameters: `limit` and `offset` to page, and whatever filters it declares to reduce the set. `cms_api_describe` lists them. Asking for 200 items and receiving 30 is a signal to ask a better question, not to ask again.

→ `mcp/docs/server/response-shaping`
