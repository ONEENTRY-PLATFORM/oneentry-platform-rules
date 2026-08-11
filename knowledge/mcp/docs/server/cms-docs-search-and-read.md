# Searching and reading this documentation

The two tools that give you these documents at runtime, how to phrase a query for this particular index, and how to page through a long document without wasting context.

With operation-level documentation hints removed, search is the navigation path. `mcp/docs/server/doc-map` is the index; this document is how to use it.

→ `mcp/docs/server/doc-map` · `mcp/docs/server/knowledge-subsystem`

## cms docs search

| Argument | Type | Meaning |
|---|---|---|
| `query` | string, at least 2 characters | What you are looking for |
| `limit` | 1–20 | Maximum hits, default 8 |

```json
{ "hits": [
    { "docId": "mcp/docs/api/baseline-data", "anchor": "user-groups-and-the-guest-group",
      "title": "User groups and the guest group", "doc": "Baseline data every instance already has",
      "score": 8.42, "snippet": "…a user group with identifier `guest` is provisioned…" } ],
  "next": "Read a hit with cms_docs_read { docId, anchor }." }
```

A hit is a **section**, not a document. The `anchor` is what you pass to `cms_docs_read` to get that section's full text.

## Phrasing a query for this index

The index matches terms with OR, with prefix and fuzzy matching, and weights a section's heading four times more than its body. Two consequences:

- **Query the way a heading is written.** `confirm token` finds the section called "Confirm tokens". `how do I approve a delete` matches weakly, on the least useful words.
- **Short queries beat long ones.** Every extra word adds weak OR matches that dilute the ranking. Two or three content words is the sweet spot.

Words of two characters or fewer are dropped entirely, so never let a short acronym carry the query.

The corpus is English. A question in another language will not find an English document — the search does not translate.

## When nothing matches

```json
{ "hits": [],
  "hint": "Nothing matched. Try a module name (menus, orders, blocks, attributes) or an entity field name." }
```

The usual fix is to search for the entity rather than the problem: `attribute sets` rather than `why are my attributes empty`.

## cms docs read

| Argument | Type | Meaning |
|---|---|---|
| `docId` | string, required | For example `mcp/docs/api/orders` |
| `anchor` | string | A section anchor; omit for the preamble |

Omitting the anchor is the normal way to open an unfamiliar document:

```json
{ "docId": "mcp/docs/api/orders" }
```

```json
{ "docId": "mcp/docs/api/orders", "anchor": "", "title": "…", "doc": "…",
  "sourcePath": "mcp:knowledge/mcp/docs/api/orders.md",
  "text": "…the preamble…",
  "sections": [ { "anchor": "what-an-order-is-made-of", "title": "What an order is made of", "bytes": 812 } ] }
```

You get the preamble plus the full list of sections with their sizes, which is enough to decide what to read next without pulling the whole document into context.

## Paging through a document

Read the preamble, pick from `sections`, then read those anchors one at a time. The `bytes` figure tells you what each will cost.

A section is capped at 12 KB; anything longer ends with `…[section truncated]`. Documents here are written to stay well under that, so seeing the marker means the document needs fixing — report it rather than working around it.

## When a docId or anchor is wrong

```json
{ "error": "Unknown docId \"mcp/docs/api/order\".",
  "candidates": ["mcp/docs/api/orders", "mcp/docs/api/order-statuses"],
  "hint": "Use cms_docs_search to get a valid docId." }
```

```json
{ "error": "Document \"mcp/docs/api/orders\" has no section \"statuses\".",
  "anchors": ["what-an-order-is-made-of", "order-statuses", "…"] }
```

Both errors carry the correct alternatives, so neither needs a second search to recover from.

## The two resources

Alongside the tools, the server publishes two MCP resources that a client can attach directly:

- `oneentry://knowledge/mcp/operating-rules` — the operating rules as one markdown document. Useful to pin into context at the start of a session.
- `oneentry://knowledge/index` — every document with its title and section count, as JSON.

## Reporting a gap

If you needed something that is not here, say so to the human in one line: what you were doing, what you searched for, and what you had to work out by trial. That sentence is what turns into the next document.

→ `mcp/docs/server/authoring-knowledge`
