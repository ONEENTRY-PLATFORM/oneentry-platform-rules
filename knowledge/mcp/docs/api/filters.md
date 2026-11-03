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

The operator has to make sense for the type. Greater-than on a string, or membership on a number, either fails or returns nothing.

Watch two cases in particular:

- **Numbers can be `null`.** An existence check is not the same as comparing to zero.
- **Choice attributes store option ids.** Filter on the id, not on the label a human sees.

→ `mcp/docs/api/attribute-types`

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
- **Filtering a choice attribute by its label.** Store and filter by option id.
- **Assuming a truncated result means "no more matches".** The response was capped; page it.
- **Patching a position field on a filter item.** Use the position operation.
- **Expecting a pinned item to have products behind it.** Pinning is presentation only.
- **Looking for a way to pin one value.** Pinning is set on the attribute.
- **Assuming a pinned facet is a fixed set.** New values are added to it automatically.

→ `mcp/docs/api/products`
