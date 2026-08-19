# Global search

A cross-entity search that answers "where is this thing" across pages, products, blocks, forms, orders and the rest, rather than searching one listing at a time.

It is a lookup tool, not a reporting tool. For anything you need exhaustively, use the entity's own listing with filters.

→ `mcp/docs/api/filters` · `mcp/docs/api/index-attributes`

## What it is for

Given a word a human used — a product name, a page title, an order reference — global search finds the entities matching it and tells you what kind each one is. From there you fetch the entity properly through its own operation.

That is the intended shape of the workflow: search to locate, then read to work.

## Finding the operation

```text
cms_api_search { "query": "search", "method": "get" }
```

There is an admin-side search and a public one; this server exposes the admin side.

## Parameters worth setting

Expect to be able to control:

- the **query** itself;
- the **locale**, since content is language-keyed and a match in one language is not a match in another;
- which **entity kinds** to include;
- how many results **per kind**, and paging over the whole set.

Restricting the kinds is the single most useful habit. A query for a common word across every kind produces a large, mostly irrelevant response that this server will then truncate.

## Results tell you what matched

A result identifies the entity, its kind, and which field the match came from. Use that: a product matched on its title is a different signal from one matched on a description fragment, and the field tells you whether the match is the one the human meant.

## What it will not do

- **It is not exhaustive.** Results are limited per kind by design.
- **It does not replace filtering.** "All products under £10" is a filtered listing, not a search.
- **It does not translate.** A query in one language finds content written in that language.
- **It does not see unindexed content.** What is searchable follows the same indexing rules as everything else.

→ `mcp/docs/api/index-attributes#what-an-index-attribute-is`

## Semantic search over one kind of entity

Separately from global search, most entity kinds carry a `POST /{kind}/vector/search` route that matches by meaning rather than by word — products, pages, orders, users, discounts, admins and form data each have one. It answers the question global search cannot: "something like this", where the human's wording and the stored wording share no words.

Three things to know before reaching for it:

- It is per kind. There is no cross-entity variant, so pick the kind first.
- It depends on a capability an instance may not have available. When it is not, the call answers `503`. That is a temporary state, not a malformed request: retry later, or fall back to the entity's own listing with a filter. Do not rewrite the body and try again, and do not report it as a defect without saying it was a 503.
- On form data the working route is the public one; the admin variant answers `405`.

Match quality follows what has been indexed, so a kind whose content was loaded recently can return less than you expect while the rest of the instance behaves normally.

## Permissions on the public side

The equivalent Content API route needs its permission granted to the relevant group before a site can call it — and on some instances that grant is not in place by default. A site search that returns a permission error is that, not a missing feature.

Grant the existing permission to the group; do not create one.

→ `mcp/docs/api/users-and-groups`

## Using it well

1. Ask the human for the exact wording they saw, not a paraphrase.
2. Restrict to the kinds that make sense.
3. Pass the locale explicitly.
4. Take the identifiers from the results and read the entities properly.
5. If nothing matches, try the entity's own listing with a filter before concluding the thing does not exist.

## Common mistakes

- **Searching every kind for a common word.** Large, truncated, unhelpful.
- **Treating results as a complete set.** They are capped per kind.
- **Reporting "not found" from search alone.** Confirm with the entity's listing.
- **Omitting the locale** and then concluding a translation is missing.
