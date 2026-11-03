# Page listing filters

The paginated page listing reads one level of the page tree at a time. Its body can pick chosen pages wherever they sit, or narrow a level by page type and by attribute conditions. This document covers those body keys and the `400` answers they produce.

Read it before listing pages by id, or when a narrowed listing comes back empty.

→ `mcp/docs/api/pages#the-page-tree` · `mcp/docs/api/pages#why-a-catalog-page-filter-returns-plain-pages`

## A listing without parentId reads the top level only

`POST /pages/paginated` with no `parentId` in the query reads the root level, and `ids` narrows that level rather than searching the tree. Asking for the id of a nested page this way answers `200` with no rows, which looks like a missing page and is not.

## Reading chosen pages wherever they sit

Send `ignoreParent: true` with a non-empty `ids` and no `parentId`:

```json
{ "ignoreParent": true, "ids": [245, 312] }
```

Each refusal answers `400` naming the rule:

| Body | Answer |
|---|---|
| `ignoreParent: true` with no `ids`, or `ids: []` | `ignoreParent requires non-empty ids` |
| `ignoreParent: true` with `parentId` in the query | `ignoreParent cannot be combined with parentId` |

## Narrowing one level by type or condition

`types` is a list of page types. `filter` is a list of conditions, all of which must pass, each `{ attributeMarker, conditionMarker, conditionValue }` or `{ isVisible }`. With either key, `childrenCount` and `total` count only matching pages.

With `parentId` and `types`, the parent must itself be of a listed type, otherwise the level is empty. A common page that groups catalogue pages answers no rows for `types: ["catalog_page"]`; list both types to read it.

→ `mcp/docs/api/filters#comparisons-follow-the-attribute-type`

## Body combinations that answer 400

| Body | Answer |
|---|---|
| `type` and `types` together | `400 type cannot be combined with types` |
| `filter: []` | `400 filter must be a non-empty array of conditions or null` |

Leave `filter` out for "no conditions". The single `type` key keeps its older meaning, under which a catalogue filter also returns plain pages that have catalogue pages below them.

## Common mistakes

- **Looking up a nested page by `ids` alone.** It reads the root level; add `ignoreParent: true`.
- **Narrowing a mixed branch by one type.** The parent of the other type empties the level.
- **Sending `filter: []`.** Leave the key out.

→ `mcp/docs/api/interface-module-data#reading-what-a-module-shows` · `mcp/docs/api/pages`
