# Attribute value search

`GET /api/admin/search/attribute-values` is a mode of the admin global search that answers "which attribute holds this value, and in which kind of record" instead of listing the records themselves. It is the step before a related-data filter or a value write, when you know a value but not the attribute it lives in.

Read this when a human gives you a value — a category name, a phone number — and you need the attribute marker to filter or write by.

→ `mcp/docs/api/global-search#the-query-parameter-is-called-q` · `mcp/docs/api/related-data#filtering-a-node-by-pattern-or-by-comparison`

## What one row of the answer means

The call takes `q`, `langCode` and optionally `types`, with the same names and kinds as the global search. It answers one row per pair of record kind and attribute where the value was found:

```json
{ "query": "soups", "items": [{ "type": "products",
  "attribute": { "identifier": "category", "title": "Category" },
  "attributeSetIds": [65], "records": 10, "sample": "Soups" }] }
```

`attributeSetIds` are the sets that define the attribute, `records` how many records of that kind hold a match, and `sample` one matching value, shortened when long. Only attribute values are searched — titles and markers of records never produce a row. The list of pairs is capped, and the usual search permissions decide which kinds appear.

## How the query is matched

- From three characters on, `q` matches anywhere inside a value, ignoring case: `anna` finds both `Anna` and `Panna Cotta`.
- With one or two characters it is not a substring search: only a whole value, or a whole item of a list value, equal to `q` ignoring case is found. `so` does not find `Soups`.

An empty `items` therefore does not prove that no value contains a short query; it proves that none equals it.

## Which user attributes are searched

For users only the attributes of their sign-up forms are searched. A value a user holds in any other attribute never produces a row.

## From a row to the records

The row names the attribute, not the records. To list them, put the attribute into a related-data plan as `attributes.<identifier>` with a pattern or a comparison, rooted at the entity named by `type`.

→ `mcp/docs/api/related-data-root-lists#listing-records-without-ids`

## Common mistakes

- **Reading a row as a record.** It is an attribute; `records` is a count.
- **Searching two characters as a substring.** Short queries match whole values only.
- **Expecting a record title to match.** Only attribute values are searched here.

→ `mcp/docs/api/global-search#what-global-search-never-finds` · `mcp/docs/api/related-data`
