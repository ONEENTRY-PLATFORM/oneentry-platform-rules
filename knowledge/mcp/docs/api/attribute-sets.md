# Attribute sets

An attribute set is the schema attached to an entity: which custom fields it has, of which types, with which markers. You cannot write attribute values correctly without reading the set first.

This is where most failed payloads on this API come from, so read it before your first product or page write.

→ `mcp/docs/server/payload-conventions#attributessets-is-two-levels-deep` · `mcp/docs/api/attribute-types`

## What a set is

A named collection of attributes, belonging to an **attribute set type** that says what kind of entity it can be attached to — products, pages, blocks, forms, orders, users, groups, events, admins, discounts.

An entity references a set by `attributeSetId`. Every attribute in that set is a field the entity can carry.

## Read the set before building a payload

```text
cms_api_search { "query": "attributes sets", "mutating": false }
```

From the set you need three things per attribute: its **type** (`string`, `integer`, `image`, `list`, …), its numeric **id**, and its **marker**. The first two build the payload key; the marker is how the attribute is identified in the admin panel and in Content API responses.

The fastest way to see a correct shape is to read an existing entity of the same kind and copy the structure of its attribute object.

## The value key is type and id

Inside `attributesSets` the key is the attribute's type followed by `_id` and its numeric id:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1", "integer_id43": 7 } } }
```

`string_id42` is the attribute of type `string` with id 42. You cannot construct these from markers, and you cannot guess the ids — they come from the set.

## Two levels always

The outer key is a locale code, the inner key is the attribute key. Both levels are required, on every write, for every attribute — including ones whose value is not really language-dependent.

A single-level map is the trap worth memorising: it is **accepted**, the call answers 201, and the attributes are stored empty. Nothing warns you. Read the entity back by id after every write that touches attributes.

→ `mcp/operating-rules#attribute-values-are-two-level-and-locale-keyed`

## attributeSetId and the set type id

Two different numbers:

- **`attributeSetId`** — which set the entity uses. Read it from an existing entity of the same kind, or from the sets listing.
- **the set type id** — which audience the set belongs to. It is instance data with no fixed value, and there is no lookup you can hardcode against.

Both differ between instances. Resolve them against the instance you are connected to, every time.

## Attribute set types are provisioned

The set types themselves — for products, pages, blocks, forms, users, groups, orders, events, admins, discounts, and a system one — come with the instance. Their names are **not unique**, so creating a duplicate succeeds silently and leaves an ambiguous type that resolution by name will then pick arbitrarily.

Resolve by name and use what is there. Never create a set type.

→ `mcp/docs/api/baseline-data#attribute-set-types-and-field-types`

## Create a set or an attribute

Creating a *set* is a normal operation: it is customer data, not baseline data. Creating an *attribute inside a set* is normal too, and it is how new fields appear on an entity kind.

Before you do either:

- Check whether a suitable set already exists. Entities of one kind usually share one set, and a second set fragments the content model in a way that is awkward to undo.
- Choose the attribute type from the closed list; there is no way to add a type.
- Pick a marker that reads well in the admin panel — humans see it.

Dry run first, and read the set back to confirm the attribute's id before writing any values against it.

## An attribute definition is locale keyed too

The locale level is not only for values. Inside a set, an attribute's own localized fields are maps keyed by locale code first:

```json
{
  "type": "text",
  "identifier": "question",
  "localizeInfos": { "en_US": { "title": "Your question" } },
  "validators": { "en_US": { "requiredValidator": { "strict": true } } }
}
```

`listTitles` and `additionalFields` follow the same shape.

Written one level flat — `"validators": { "requiredValidator": { "strict": true } }` — the attribute is accepted and the call succeeds, but the rule is stored where nothing reads it. The attribute then behaves as one with no validators: the validator map comes back empty for every locale, and a field you meant to make required is not enforced when the value is submitted.

So after writing an attribute, read the set back and confirm each localized field sits under a locale code.

## Changing an attribute afterwards

Changing an attribute's **type** after values exist is the one change to treat as dangerous. Existing values were stored in the shape the old type implied, and there is no automatic conversion.

If a human asks for it, say what will happen to the existing values and let them decide. Adding a new attribute and migrating deliberately is almost always the better route.

## Checklist before writing values

1. Read the set and note each attribute's type, id and marker.
2. Read the active locale codes.
3. Build `attributesSets` as locale → `<type>_id<id>` → value.
4. Dry run.
5. Send, then **read the entity back by id** and confirm the values are present.

→ `mcp/docs/api/products` · `mcp/docs/api/pages`
