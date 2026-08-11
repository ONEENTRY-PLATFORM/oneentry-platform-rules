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

## Pinned values

Pinning keeps a value visible in a facet regardless of the current data. It is how a shop stops a category disappearing from the sidebar because it is temporarily out of stock.

Two things to know: pinning affects presentation, not matching — a pinned value with no products behind it returns nothing when clicked; and new values arriving in the data do not automatically become pinned.

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
- **Expecting a pinned value to have products behind it.** Pinning is presentation only.

→ `mcp/docs/api/products`
