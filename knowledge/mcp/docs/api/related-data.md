# Related data

Related data answers a question that crosses sections in one call: the orders of a user, what sits in a cart, who bought a product. You describe the records you want as a plan, and the instance follows the links between entities for you.

Read this before building a plan or when a plan answers `400`. Listing, totalling and grouping a whole entity without ids has its own document.

→ `mcp/docs/api/related-data-root-lists#listing-records-without-ids` · `mcp/docs/api/interface-modules#the-screen-base-holds-a-custom-screen`

## The four calls and the order to make them

| Call | What it gives you |
|---|---|
| `GET /api/admin/related-data/graph` | The entities a plan can start from |
| `GET /api/admin/related-data/graph/{entity}?langCode=` | That entity's fields and its edges to other entities |
| `GET /api/admin/related-data/limits` | The ceilings your plans must stay under |
| `POST /api/admin/related-data/query?langCode=` | The records the plan describes |

Walk the graph one entity at a time and build the plan from what it returns; nothing in the plan can be guessed. The query is a read even though it is a `POST`: it changes nothing.

The graph only offers what your permissions let you read. An entity that answers `404` on the graph step, or `unknown entity` in a plan, is either misspelled or out of your reach — check your permissions before concluding it does not exist.

## Edge keys are copied from the graph unchanged

Every edge in a graph step has a `key`, a `to` entity and a `many` flag. Put the `key` into the plan exactly as the graph wrote it, punctuation included; never build one by hand or translate it.

- `many: false` — a link to one record, such as the user of an order.
- `many: true` — a link to a list, such as the orders of a user.
- A list edge that carries its own `fields` holds values that belong to the link rather than to either record — the quantity of a product in a cart. Ask for them with `edgeFields` on the node and they come back under `edge` on each item; sort and total them as `edge.<field>`.

## Building one node of a plan

A plan is `root`, the root `fields`, and an `include` list of nodes. A node has:

- `edge` (or `path`, below) and `as`, the key its answer appears under. `as` must be unique among its siblings and must not repeat a field name at the same level; `__proto__` is refused.
- `fields` — the fields of the target entity. `[]` makes the node a pass-through that returns only `id`.
- `limit` 0 to 10, default 10. `0` returns only `total` and the totals.
- `sort` as `{ "field": "createdDate", "dir": "desc" }`, `filter`, `filterMatch`, `aggregates`, and a nested `include`.

A link to one record answers an object or `null`, and refuses `limit`, `sort`, `filter` and `aggregates`.

```json
{ "root": { "entity": "users", "ids": [41] }, "fields": ["id", "identifier"],
  "include": [{ "edge": "<orders.user_id", "as": "orders", "fields": ["id", "totalSum"],
    "limit": 5, "sort": { "field": "createdDate", "dir": "desc" },
    "aggregates": [{ "fn": "count" }, { "fn": "sum", "field": "totalSum" }] }] }
```

## What a list node answers

A list node answers `{ total, aggregates, items }`; `aggregates` appears only when you asked for totals. `total` counts every linked record that passes the filter, not just the `limit` you received.

The answer as a whole stops growing at `maxRecords` records. When that happens it carries `truncated: true` and some lists are incomplete — narrow the plan rather than trusting the counts of the items you got.

## Which fields can be sorted filtered or totalled

Each field in a graph step carries flags, and a plan that ignores one answers `400`:

- `sortable` — usable in `sort`.
- `filterable` — usable in a pattern condition.
- `comparable` — usable in a comparison condition.
- `aggregates` — the totals allowed on it; a field without the list cannot be totalled.
- `money` — the value is an amount in the record's currency.

Attribute fields also carry `attributeType` and `sets`, the attribute sets that define them.

## Filtering a node by pattern or by comparison

`filter` takes up to five conditions, either kind mixed. `filterMatch` is `"all"` by default or `"any"`.

- A pattern: `{ "field": "identifier", "regex": "^vip", "not": false, "flags": "i" }` — a regular expression, on `filterable` fields only.
- A comparison: `{ "field": "totalSum", "op": "gt", "value": 100 }` with `op` one of `gt`, `gte`, `lt`, `lte`, `between`. `between` takes `[from, to]` and includes both ends. Only `comparable` fields.

Dates are ISO 8601; one without an offset is read as UTC, and a date copied from a response finds its own record. Amounts are plain numbers in the record's currency. An empty value never satisfies a condition. `total`, the totals and `limit` all apply after the filter.

## Totals and the values breakdown

`aggregates` takes `count`, and `sum`, `avg`, `min`, `max` with a `field`. Each answers under `fn:field`, such as `sum:totalSum`.

`{ "fn": "values", "field": "statusIdentifier", "limit": 3 }` returns the most frequent values of a field, `limit` 1 to 10 and 3 by default, under `values:<field>` as `{ items: [{ value, title, count }], total }`. `title` appears where the value has a display name.

Amount totals come back as strings, such as `"10.00"`. Next to them `currency` names the currency; when the records mix currencies it is `null` and `currencies` lists them.

## Counting through a chain of edges with path

`path` replaces `edge` with two or more steps, each `{ edge, filter, filterMatch }`. It answers about the records at the end of the chain — for a product, the users who bought it, through its orders.

- `as` and `aggregates` are required. Without `fields` the node answers totals only.
- `{ "fn": "count", "distinct": true }` counts different end records once each, under `count:distinct`.
- With `fields` it also returns the distinct end records. `limit` and `sort` then use fields of the end entity, and `limit` without `fields` answers `400` saying it requires fields.

## Narrowing a plan to an interface module

`module` takes the `base` and `scope` from an interface module's `config` and narrows the plan to what that module shows. Every edge or path step that reaches the base's entity keeps only records inside the scope, and a root list must start from that entity or the plan answers `400`.

It is a view, not an access boundary: you can read the same records without it. A `screen` module has no scope and is refused here.

## Attribute values in a plan

Attributes are fields named `attributes.<marker>`, and each answers `{ type, value }` under `attributes`. Password and other sensitive fields are never offered; sensitive values of form submissions come back as `null`.

Orders, users and form submissions expose `attributeSetIdentifier`, the attribute set their form values belong to. It is what a value write needs, and the user list returns it as well.

→ `mcp/docs/api/attribute-value-writes#writing-form-values-of-orders-users-and-submissions`

## Plan limits differ between instances

`GET /limits` returns `maxDepth`, `rootIdsByDepth`, `maxInclude`, `maxAggregates`, `maxLimit`, `defaultLimit`, `maxRecords`, `maxRootLimit`, `defaultRootLimit`, `maxGroups` and `maxRootConditions`. The numbers depend on the instance; read them and never hardcode them.

`rootIdsByDepth` caps `root.ids` by how deep the plan goes, and each id is a whole number from 1 to 2147483647. A plan over a ceiling answers `400` naming it, for example `depth exceeds 4` or `include: at most 30 items`.

## Why a plan answers 400

Every problem in the plan is reported at once. `message` lists them as `path: problem`, joined in one string, so you can fix them in one pass:

```text
include[0].limit: not allowed on a to-one edge; include[1].as: must not be "__proto__"
```

The path points at the node: `include[2].filter[1].op` is the second condition of the third node. Fix every item named, then send the plan again.

## Common mistakes

- **Writing an edge key by hand.** Copy it from the graph step.
- **Sending `limit` or `sort` to a link to one record.** It answers an object, not a list.
- **Taking a truncated answer as complete.** Check `truncated` before trusting counts.
- **Summing amounts as numbers.** They arrive as strings with a `currency`.
- **Treating `module` as a permission.** It only narrows what the plan returns.

→ `mcp/docs/api/related-data-root-lists#grouping-root-records` · `mcp/docs/api/global-search`
