# Related data root lists totals and groups

A related-data plan whose root has no `ids` lists the records of one entity instead of starting from records you already know. That root can be filtered, sorted, paged, totalled and grouped, which makes it the way to answer "how many orders per month" or "the ten newest users with their carts" in one call.

Read this when you need a list, a count or a breakdown over a whole entity. How edges, nodes and filters work is in the main related-data document.

→ `mcp/docs/api/related-data#building-one-node-of-a-plan` · `mcp/docs/api/related-data#plan-limits-differ-between-instances`

## Listing records without ids

Leave `ids` out of `root` and the root becomes a list. It answers `{ total, items, truncated }`, and `total` counts every record that passes the conditions regardless of `limit` and `offset` — it arrives even when the page is empty.

| Key | Meaning |
|---|---|
| `filter`, `filterMatch` | The conditions you chose, as on any node |
| `where` | Extra conditions, always joined with "and" to each other and to `filter` |
| `sort` | `{ field, dir }` on a `sortable` field |
| `limit` | 0 to `maxRootLimit`, default `defaultRootLimit`; with `include`, no more than the `rootIdsByDepth` value for the plan's depth |
| `offset` | 0 or more |

A root with `ids` takes none of these and answers `400` saying `root.ids: cannot be combined with` the key you sent. With `ids` the answer has no `total`.

## Paging a root list without losing records

Records with equal sort values, and every list without `sort`, are ordered by `id` descending. That order is stable, so stepping `offset` by `limit` visits every record once.

Use `where` for the conditions of the page you are on — a date range picked by a person, say — and `filter` for the ones that define the list. Both narrow the same set.

## Narrowing a root list to an interface module

`module` narrows the root list as well as the edges. The root entity must be the entity of the module's base — `orders` for an orders module — or the plan answers `400` naming both.

→ `mcp/docs/api/related-data#narrowing-a-plan-to-an-interface-module`

## Grouping conditions with any and all

On the root, an item of `filter` or `where` may be a condition or a group: `{ "any": [ … ] }` means "or", `{ "all": [ … ] }` means "and". Groups nest at most two levels deep, and each of the two keys holds at most `maxRootConditions` conditions counted across the whole tree.

```json
{ "root": { "entity": "orders", "where": [
  { "field": "createdDate", "op": "between", "value": ["2026-01-01", "2026-02-01"] },
  { "any": [ { "field": "statusIdentifier", "regex": "^paid$" },
             { "field": "totalSum", "op": "gt", "value": 100 } ] } ] },
  "fields": ["id", "totalSum"] }
```

Groups exist only on the root. On a node or a path step they answer `400` saying condition groups are allowed only in `root.filter` and `root.where`.

## Totals over the whole root list

`root.aggregates` takes the same functions as a node — `count`, `sum`, `avg`, `min`, `max`, `values` — and computes them over every record that passes the conditions, not over the page. Send `limit: 0` to get the totals with no records.

The items, `total`, the totals and the groups are always computed over the same set of records.

## Grouping root records

`root.groups` is `{ by, aggregates, tz }`:

- `by` lists one or two fields, each `{ field }` or `{ field, bucket }`. A field may be a string, a list, a link or a date.
- A date needs `bucket` set to `day`, `week` or `month`. Weeks start on Monday and are keyed by that Monday as `YYYY-MM-DD`.
- `tz` is an IANA zone name such as `Europe/Berlin` used to bucket dates with a time; without it buckets are UTC.
- `aggregates` takes the node functions except `values`, which answers `400` in groups.

A record whose list field holds several items counts in the group of each item.

## What a grouped answer looks like

```json
{ "total": 270, "groupsTotal": 13, "items": [],
  "groups": [{ "key": ["2026-06-15", "Soups"], "title": [null, null],
               "count": 10, "aggregates": { "avg:price": 11.6 } }] }
```

With one `by` field `key` and `title` are single values; with two they are pairs in `by` order. Groups come in ascending key order, with records that have no value in a last group keyed `null`. At most `maxGroups` groups are returned; `groupsTotal` says how many there were.

## Common mistakes

- **Sending `limit` or `offset` next to `ids`.** A root has ids or is a list, never both.
- **Reading `total` as the page size.** It counts every matching record.
- **Grouping by a date without `bucket`.** The plan answers `400`.
- **Using `any` or `all` on a node.** Only the root takes groups.
- **Summing `count` across groups.** A record with several list items counts more than once.

→ `mcp/docs/api/related-data#why-a-plan-answers-400` · `mcp/docs/api/orders#listing-filtering-and-paging-orders`
