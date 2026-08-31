# Global search

A cross-entity search that answers "where is this thing" across pages, products, blocks, forms, orders and the rest, rather than searching one listing at a time. There is an admin-side address and a public one, and this document covers both.

It is a lookup tool, not a reporting tool. For anything you need exhaustively, use the entity's own listing with filters — and know its limits before you promise a customer a site search built on it.

→ `mcp/docs/api/filters` · `mcp/docs/api/index-attributes`

## What it is for

Given a word a human used — a product name, a page title, an order reference — global search finds the entities matching it and tells you what kind each one is. From there you fetch the entity properly through its own operation.

That is the intended shape of the workflow: search to locate, then read to work.

```text
cms_api_search { "query": "search", "method": "get" }
```

This server exposes the admin side. The public address is `GET /api/content/search`, and a site calls it directly.

## The query parameter is called q

`q` carries the search text, and no other name works: `query`, `search` and `name` all answer `400`. That is worth knowing before concluding the address is wrong, because `name` **is** the parameter of a different route — the per-kind quick search below.

- `q` — one to two hundred characters, control characters rejected. The public address additionally refuses anything shorter than three characters with `Search query must be at least 3 characters`; the admin side accepts one.
- `langCode` — content is language-keyed, so a match in one language is not a match in another. Pass it explicitly.
- `types` — comma-separated kinds. Restricting them is the single most useful habit.
- `visibility` — admin side only; the public address always searches visible records and strips the flag from the answer.

## Results come back grouped in fives

The answer is `{ "query": …, "groups": [ { "type": …, "items": […], "hasMore": … } ] }` — one group per kind, each holding **five** items by default and its own `hasMore`. A group with no matches is left out entirely.

`limitPerType` raises that to twenty, and no further. So five results for a whole project is the default shape of the response, not evidence that the project holds five matching records — check `hasMore` before reading anything into the count.

`hasMore` says only that the candidates it collected did not fit on the page. It is not a count, and no total comes back, so never quote a number of matches from this response.

## Paging one kind needs limit and exactly one type

`limit` switches the call into paging a single kind, and it then demands one:

```text
GET /api/content/search?q=beneficiary&limit=50
→ 400 Drilldown mode (limit/offset) requires exactly one accessible type in `types`
```

Send `types` with exactly one kind you may actually search, and `limit` with `offset` page it. Two constraints bound the window:

- `limit` is at most 50;
- `offset + limit` may not exceed 200, and answers `400` when it does.

Inside that mode an empty group is kept rather than dropped, so the shape of the answer no longer depends on whether anything matched. On the last reachable page `hasMore` is `false` whether or not more records exist beyond the two hundredth.

## What the public search covers

The public address searches four kinds only — **products, pages, blocks and discounts** — and only records that are visible. A kind outside that set is dropped silently rather than refused, so asking for one and nothing else returns an empty answer, or the drilldown `400` above when `limit` is set.

Its permission record must be linked to the reading group. It is provisioned linked to the guest group, so a site usually has it, but an instance that predates the route may not.

→ `mcp/docs/api/content-api-permission-rules#the-five-rules-and-what-each-one-opens`

## What global search never finds

- **Files and documents.** Nothing is searched by file name or file content, and the attribute types that hold files, images, groups of images, JSON and captcha values are excluded from matching. A document attached to a page as an attribute value cannot be found this way at all.
- **Fields marked as passwords.** Excluded from every mode.
- **Content in another language.** It does not translate.
- **Anything unindexed.** What is searchable follows the same indexing rules as everything else.

A site that must find documents needs its own index built from the texts that carry them. Decide that early: it is a real piece of work, and the search will not grow into it.

→ `mcp/docs/api/index-attributes#what-an-index-attribute-is`

## Results tell you what matched

A result identifies the entity, its kind, and which field the match came from. Use that: a product matched on its title is a different signal from one matched on a description fragment, and the field tells you whether the match is the one the human meant.

## Searching one kind by title alone

Each of pages, blocks and products also carries a quick search — `GET /api/content/pages/quick/search?name=…&langCode=…` — which matches the title in the requested language and returns identifiers and titles, nothing else. It takes `name`, not `q`, and on pages an optional `url` narrows it to one branch.

It is the cheap answer to "find the page called roughly this" and it does not cap its answer in groups of five.

## Semantic search over one kind of entity

Separately, most kinds carry a `POST /{kind}/vector/search` route that matches by meaning rather than by word — products, pages, orders, users, discounts, admins and form data each have one. It answers the question the word search cannot: "something like this", where the human's wording and the stored wording share no words.

The public form works and is granted to the guest group by default:

```bash
curl -X POST "https://your-instance.example/api/content/pages/vector/search?langCode=en_US" \
  -H "x-app-token: <application token>" -H "Content-Type: application/json" \
  -d '{"queryText": "how do I buy past service"}'
```

**The address has a slash: `/vector/search`.** `/vector-search` with a hyphen is not an address at all and answers `404`. Read that `404` as a typo, never as "this instance cannot search by meaning" — it is the one mistake that turns a working capability into a feature you tell a customer they cannot have.

## Reading a semantic search answer

The answer is `{ "items": [...], "total": n }`, and a `POST` that worked answers `201`.

- `queryText` is required, up to 512 characters. `limit` defaults to 30 and `offset` pages, both as query parameters alongside `langCode`.
- `total` counts the candidates found before paging, and is itself capped by `maxHits` — 100 by default, 500 at most.
- `vectorDistanceThreshold` tightens or loosens the match, from 0 to 2, lower being stricter. Send `debug: true` and each item carries its `distance`, which is how you choose a threshold rather than guessing at one.
- The search covers **the whole instance**. There is no site, branch or parent parameter, so a project holding several sites gets all of them back and must narrow the results itself.
- Pages come back whether or not they are visible. Each item carries `isVisible`; a site that shows results unfiltered will publish pages that were deliberately hidden.

Two more things to know. It depends on a capability an instance may not have available, and when it is missing the call answers `503` — a temporary state, not a malformed request, so retry later or fall back to a filtered listing rather than rewriting the body. And on form data the working route is the public one; the admin variant answers `405`.

Match quality follows what has been indexed, so a kind whose content was loaded recently can return less than you expect while the rest of the instance behaves normally.

→ `mcp/docs/api/index-attributes#why-a-search-by-meaning-finds-nothing-for-a-kind`

## Using it well

1. Ask the human for the exact wording they saw, not a paraphrase.
2. Restrict to the kinds that make sense, and pass the locale explicitly.
3. Read `hasMore` before you report a count.
4. Take the identifiers from the results and read the entities properly.
5. If nothing matches, try the entity's own listing with a filter, or the search by meaning, before concluding the thing does not exist.

## Common mistakes

- **Calling the parameter `query` or `name`** and reading the `400` as a wrong address.
- **Reporting "the search returns five results"** as a defect. It is the default group size.
- **Sending `limit` without a single `types` value.** That is the drilldown `400`.
- **Expecting documents in the results.** File-bearing attribute values are never matched.
- **Typing `/vector-search`** and concluding from the `404` that search by meaning does not exist.
- **Publishing semantic results unfiltered.** Hidden pages are among them.
- **Treating either search as a complete set.** Both are capped by design.
