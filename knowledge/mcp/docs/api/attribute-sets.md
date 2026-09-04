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
  "type": "string",
  "identifier": "first_name",
  "localizeInfos": { "en_US": { "title": "First name" } },
  "validators": { "en_US": {
    "requiredValidator": { "strict": true },
    "stringInspectionValidator": { "stringMin": "3", "stringMax": "20" } } },
  "additionalFields": { "en_US": [
    { "marker": "placeholder", "type": "string", "value": "Your name" } ] }
}
```

The bounds are **strings**, not numbers, and an unset one is `""` rather than absent. `listTitles` and `additionalFields` are locale-first too — `additionalFields` a list per locale, each entry carrying its own `marker`. What sits **inside** a list option is not: an option's extra value is flat.

→ `mcp/docs/api/list-options-and-extra-values`

Written one level flat — `"validators": { "requiredValidator": { "strict": true } }` — creating or updating the set answers `400`, and the message names the attribute and the field. The same check covers `localizeInfos`, `listTitles` and `additionalFields`.

One route does not run that check: replacing the schema of an existing set. There a flat map is accepted, and the rule is stored where nothing reads it — the attribute then behaves as one with no validators, and a field you meant to make required is not enforced when the value is submitted.

So after replacing a schema, read the set back and confirm each localized field sits under a locale code — through the right read, which is the next section.

→ `mcp/docs/api/silent-no-ops`

## The validator name comes from a fixed list

A validator name is not free text. Creating a set, updating it, or replacing its schema with a name outside the list answers `400`, and the message names the attribute, the locale and the name it did not recognise, then lists the ones it accepts. That response is where to read the list — no read operation returns it.

```text
requiredValidator         trimValidator             stringInspectionValidator
urlInspectionValidator    emailInspectionValidator  checkForNumberValidator
roundingNumberValidator   booleanValidator          comparisonValidator
regExpValidator           fieldMaskValidator        checkForAGivenList
defaultValueValidator     sizeInPixelsValidator     checkingFilesValidator
minMaxDateValidator       valueValidator            checkForLettersValidator
```

Two are easy to get wrong by analogy: it is `emailInspectionValidator`, not `emailValidator`, and `checkForAGivenList` carries no `Validator` suffix.

Accepted is not the same as enforced. `sizeInPixelsValidator`, `checkingFilesValidator`, `minMaxDateValidator`, `valueValidator` and `checkForLettersValidator` are stored for the admin panel to read, and no submission is rejected because of them. The rest are checked when a value is submitted. So setting one of those five and reporting the field as validated is wrong — confirm by submitting a value that should fail.

The name check follows the locale key. A `validators` map written flat, with no locale level, is not name-checked at all, so on the schema-replace route — which accepts a flat map — an unrecognised name is stored there and no `400` is raised. Key the map by locale and both checks apply.

A set that already holds an unrecognised name keeps it, and nothing re-checks it until the set is written again. That write is where it surfaces: it answers `400` until the name is corrected, and the field is enforced by nothing up to that point.

## Two reads two answers

An attribute set can be read two ways, and they do not show the same thing:

| read | shows |
|---|---|
| the set **by id** | the raw stored schema — exactly what you sent, including a flat map |
| the set's attributes **by marker**, and a form **by marker** | the projection a site and an SDK receive, resolved for one locale |

The projection resolves each localized field for the requested locale. A flat `validators` map has nothing under that locale, so it comes back as `{}` — while `type`, `position` and `identifier`, which are not locale-scoped, come through unchanged. An attribute that looks fully populated in the raw read can therefore arrive at a site with no validators at all.

**That asymmetry is the trap.** Reading the set by id after writing shows your own flat map back to you and looks like success. It is the projection that decides whether the site sees anything.

So verify a validator through the read the consumer uses — the form by marker, or the set's attributes by marker — and check that `validators` is non-empty there. If it is empty while the raw read is full, the shape is flat: rewrite it under the locale key rather than concluding that validators are unsupported.

→ `mcp/docs/server/payload-conventions#verify-with-the-read-your-consumer-uses` · `mcp/docs/api/silent-no-ops`

## Values outlive the attribute you removed

Dropping an attribute from a set clears the values entities held under its key, but not at the instant the call returns: the cleanup runs behind the write. For a short window a read can still show the removed key.

That window is what to plan for:

- do not build an update from a read taken immediately after removing an attribute — a read-modify-write can send the stale key straight back;
- when you build an update from any read, compare the keys you are about to send against the current set and drop the ones it no longer defines. This is worth doing regardless: it also protects against a set edited by someone else between your read and your write.

→ `mcp/docs/api/bulk-content-migration#read-back-every-object-you-wrote`

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
