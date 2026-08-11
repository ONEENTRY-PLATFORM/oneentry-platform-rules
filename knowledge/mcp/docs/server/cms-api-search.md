# The cms api search tool

Finding Admin API operations by keyword, tag, method or whether they mutate. This is the only authority on which endpoints exist on the instance you are connected to.

Operation ids come from here and from nowhere else. An id you construct by hand is refused, and an id you remember from another instance may not exist on this one.

→ `mcp/docs/server/cms-api-describe` · `mcp/docs/server/api-catalog`

## Arguments

| Argument | Type | Meaning |
|---|---|---|
| `query` | string, required | Keywords: an entity, a path fragment, or an action |
| `tag` | string | Restrict to one tag, for example `Orders` |
| `method` | `get` `post` `put` `patch` `delete` | Restrict to one HTTP method |
| `mutating` | boolean | `true` for writes and deletes only, `false` for reads only |
| `limit` | 1–50 | Maximum hits, default 15 |

## What a hit looks like

```json
{ "hits": [
    { "opId": "AdminMenusController_updatePosition", "method": "PUT",
      "path": "/menus/{id}/position", "tag": "Menus",
      "summary": "Update menu position", "permission": "menu.update",
      "risk": "write", "score": 7 } ],
  "next": "Get the payload shape with cms_api_describe { opId }." }
```

`permission` is the admin permission the operation declares, when it declares one. `risk` is `read`, `write` or `destructive`, derived from the method.

## How results are ranked

The query is lowercased and split on spaces, slashes and dots; terms of one character are ignored. A term scores 3 when it appears in the path, 2 in the tag and 1 anywhere in the operation's searchable text. Ties break towards the shorter path, which tends to surface the general operation before its narrower variants.

Practical consequence: **path fragments are the strongest signal**. Searching `menus position` outranks searching `how do I reorder a menu`.

## Phrasing a query

- Start with the bare entity name: `menus`, `orders`, `attributes sets`. This is the reliable first move when you do not yet know the vocabulary.
- Add the action as a separate word: `orders refund`, `menu position`, `product import`.
- Use a path fragment when you know one: `users/groups`.
- Do not phrase it as a sentence. There is no natural-language understanding here — extra words only add weak matches.

## Filtering

```json
{ "query": "products", "mutating": false, "limit": 30 }
```

`mutating: false` is the safest way to explore an unfamiliar area: it lists everything you can call without any policy question arising.

```json
{ "query": "delete", "tag": "Menus", "method": "delete" }
```

`tag` matches exactly, case-insensitively. If you are not sure of the tag, run a query with no tag and read the tags off the hits — or trigger the empty-result case below.

## When nothing matches

```json
{ "hits": [], "tags": ["AI Gateway", "Admins", "Attributes Sets", "…"],
  "hint": "Nothing matched. Try a bare entity name, or filter by one of the tags listed here." }
```

The full tag list comes back with the empty result, which makes a deliberately bad query a fast way to see the whole API map. `cms_guide` shows the same map with operation counts.

## When the catalog is empty

If every search returns nothing and the tag list is empty too, the catalog itself is empty — the instance did not serve its API document. Check `cms_whoami`: it reports the operation count and any catalog warnings. Nothing can be searched or called until the instance is reachable.

→ `mcp/docs/server/errors-startup#the-catalog-is-empty`

## Never construct an operation id

Ids follow a visible pattern, which makes guessing tempting and wrong. A hand-made id is refused with a "did you mean" list:

```text
Unknown opId "AdminProductsController_getAll". No request was sent.
```

There is no `GET /products` list operation, the real one is `POST /products/all`, and no amount of pattern-matching gets you there. Search, then describe, then call.
