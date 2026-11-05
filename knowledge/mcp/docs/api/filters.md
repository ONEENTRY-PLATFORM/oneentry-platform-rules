# Filters

Two related things: the **filter payloads** you send to narrow a listing, and the **filter configuration** an operator builds so a site can offer facets to visitors.

Both rest on the same foundation — only indexed attributes can be filtered on — so read that section first if a filter is silently returning nothing.

→ `mcp/docs/api/index-attributes` · `mcp/docs/server/response-shaping`

## Filtering a listing

List operations accept filter parameters over attribute values, and the product listing takes them in the request body because they are too large for a query string.

A filter names an attribute, an operator and a value. The available operators cover equality and inequality, greater and less than, membership in a set and absence from it, and existence checks.

`cms_api_describe` shows the exact shape the operation expects. Copy it from the example rather than inferring, since filter payloads are usually loose fields.

## Only indexed attributes can be filtered

An attribute must be marked as indexed to be usable in a filter. A filter over a non-indexed attribute does not error — it **matches nothing**, which reads as "there are no such products".

That single behaviour accounts for most "the filter is broken" reports. Before debugging a filter, confirm the attribute is indexed.

→ `mcp/docs/api/index-attributes#what-an-index-attribute-is`

## Comparisons follow the attribute type

The operator has to make sense for the type. Membership on a number either fails or returns nothing, and a comparison the type cannot make — a greater-than whose value is not a number, a date filter whose value is not a date — matches no entities rather than answering an error. Greater and less than are numeric comparisons wherever both sides are numbers, including a text attribute whose value happens to be one.

Watch these cases in particular:

- **Numbers can be `null`.** An existence check is not the same as comparing to zero.
- **A choice is matched by the option's value, not by its title.** An option titled `Red` whose value is `red` is found by `red` and not by `Red`. The whole value is compared and the comparison is case-sensitive, so membership behaves like equality here and `Re` finds nothing. An entity with several options selected matches when any one of them does.
- **Dates compare at the granularity of the type.** A date compares by day, so a value stored as `2026-05-07T10:00:00.000Z` is equal to `2026-05-07`; a date-and-time and a time compare to the minute. Send ISO 8601 — anything else is not read as a date and matches nothing — and remember that a date-time with no zone is read as UTC.
- **Negation keeps everything else that carries the attribute.** On a choice, `neq` and `nin` keep entities without that option selected, including those with an empty selection; on text, `neq` keeps every other value. Entities that do not define the attribute at all are a separate case — see the next section. Text is matched by part of its value with `in` and `nin`, decimal and negative numbers are accepted, and `0` is never matched by a condition value that is not a number.
- **An empty value counts as no value.** An empty choice list, an empty media slot, a cleared text: an existence check answers false for them, and its negation answers true.

→ `mcp/docs/api/attribute-types`

## Why a negation leaves out entities without the attribute

A negated condition — `neq`, `nin`, or the negated existence check — returns only
entities that **carry the attribute being filtered on**. An entity built on a
different attribute set, one that does not define that attribute at all, is left
out, even though it plainly does not hold the value you excluded.

This is the opposite of what "everything except" suggests, and it is the usual
reason a negation returns far fewer products than the catalog size:

```text
# attribute `color` belongs to one attribute set out of several
{"filter":[{"attributeMarker":"color","conditionMarker":"nexs"}]}
# → only products whose set defines `color` and whose value is empty,
#   not every product without a colour
```

An empty value is **not** the same as a missing attribute. A product whose set
defines `color` and leaves it blank carries the attribute, so a negation keeps
it and an existence check rejects it. A product whose set has no `color` at all
is outside the filter either way: neither `eq` nor `neq` returns it.

Two consequences worth planning for:

- To ask "everything in the catalog except X", filter nothing and exclude the
  `eq X` result yourself. One negated condition will not express it across
  mixed attribute sets.
- The public product listing and the admin product listing answer the same way
  here, so a count taken from one is comparable with a page taken from the other.

→ `mcp/docs/api/attribute-sets`

## Filter configuration for a site

Separately from ad-hoc filtering, an operator can configure the facets a site shows: which attributes appear, in what order, and which of their values are surfaced.

```text
cms_api_search { "query": "filters", "mutating": false }
```

A configured filter carries the attributes it exposes and their arrangement. Some values can be pinned so they always appear, whether or not any current product carries them.

## Pinning is set on an attribute not on a value

Pinning is a property of an **attribute inside a filter**, not of the individual values you want kept. There is no operation that pins one value. The record is written through the filter's config:

```text
PUT /filters/{id}/config/attribute
  { "attributeSetId": 12, "attributeIdentifier": "colour", "isPinned": true,
    "pinAnchorId": 34, "pinAnchorType": "filter-item", "pinPlacement": "before" }
```

`pinAnchorType` is `filter-item` or `filter-custom-item` and is required whenever `pinAnchorId` is set; leaving the anchor out means the root. `pinPlacement` is `before` or `after`. Omitting `isPinned` registers the attribute without changing the flag, so a config call is not automatically a pin.

The answer carries the record back, `knownValues` among it — the values of that attribute the filter has already seen.

## A pinned attribute picks up new values on its own

That `knownValues` list is the behaviour to plan around. When a new value appears for a pinned attribute, the filter **adds an item for it automatically**, positioned against the anchor, and extends `knownValues` to match. No filter call is involved — adding the value to the attribute is enough.

So a pinned attribute is a standing instruction to surface whatever turns up, not a frozen set. Pin one and the facet grows with the data; a value you did not want shown has to be removed after the fact, and will not come back unless it is new again.

Pinning still affects presentation, not matching: an item with no products behind it returns nothing when clicked.

## Ordering within a filter

Filters and their items are ordered, and as everywhere in this API that ordering is set through a dedicated position operation rather than by patching a field.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## Reading available values

There is an operation returning the distinct values an indexed attribute currently holds. It is what a facet is built from, and it is also the fastest way to check whether a filter value you are about to send exists at all.

Use it before reporting that a filter returns nothing — often the value simply is not present in the data.

## Common mistakes

- **Filtering on a non-indexed attribute.** Silent empty result.
- **Filtering a choice attribute by its label.** Filter by the option's value.
- **Expecting a negation to cover entities without the attribute.** It does not.
- **Assuming a truncated result means "no more matches".** The response was capped; page it.
- **Patching a position field on a filter item.** Use the position operation.
- **Expecting a pinned item to have products behind it.** Pinning is presentation only.
- **Looking for a way to pin one value.** Pinning is set on the attribute.
- **Assuming a pinned facet is a fixed set.** New values are added to it automatically.

→ `mcp/docs/api/products`
