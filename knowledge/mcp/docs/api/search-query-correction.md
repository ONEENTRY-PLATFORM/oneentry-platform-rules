# Search query correction

The instance keeps its own list of the words used in its content and corrects typos in public search queries against it: `fish ladr` becomes `fish ladder`, and the search then runs on the corrected words. A site no longer needs its own copy of the word list, and a newly published document becomes a correction target without any action from the site.

Read it when building a "Showing results for …, search as written instead" line, when a public search answers with a corrected query, or when a correction is wrong.

→ `mcp/docs/api/file-content-search#the-public-search-is-a-narrower-surface-with-the-same-shape` · `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Which words can be a correction target

Words come from what a visitor may read, per language:

- titles of visible pages, products and blocks. A page contributes its menu title and its page text as well — plain, HTML and markdown; a product and a block contribute the title only.
- values of their `string`, `text` and `textWithHeader` attributes, with markup removed;
- text of attached documents that are searchable, attached to a visible page, product or block. `fileContentSearch.ownerTables` does not gate this: a document's words stay correction targets even while its owner's section is out of that list.

Never a target: anything from user profiles, hidden records and their documents, documents excluded from search or marked low confidence, and attributes of any other type — numbers, dates, codes, files, lists.

One attribute is left out when it carries `isInSearchVocabulary: false` or `isVisible: false` in the set schema, or is marked as SKU, price, currency or tax rate. A validator does not exclude anything: a `string` attribute validated as an email or a URL is read like any other.

Only words of letters, 4 to 24 characters long by default. A word that occurs only in documents counts when it appears in at least 2 documents or at least 3 times, which keeps scanning noise out.

## Correct a query directly

`GET /api/content/search/correct` with `x-app-token`.

```http
GET /api/content/search/correct?q=fish%20ladr&langCode=en_US&scope=pages,files

{ "query": "fish ladr", "corrected": "fish ladder",
  "tokens": [ { "from": "fish", "to": null },
              { "from": "ladr", "to": "ladder", "distance": 2, "docFreq": 1, "reason": "cooccurrence" } ] }
```

- `q` is required and 1 to 200 characters. Missing or empty answers `400`.
- `langCode` defaults to `en_US`; `en_US` and `en` are one language of the list. A value that is not a locale code answers `400`. A language with no words answers `200` with `"corrected": null`, never an error.
- `scope` is a comma-separated subset of `pages`, `products`, `blocks`, `files`, and limits targets to words that exist in at least one of them. Pass the section you are about to search, so a product search is never corrected to a word that exists only in a document. Any other value answers `400` naming the four.
- `"corrected": null` means there is nothing to correct. Every word of the query is listed in `tokens`; `to: null` means kept as written.
- `distance`, `docFreq` and `reason` are present only on a token that was corrected, and a correction taken from the project's own pairs carries `reason` alone.
- `reason` is `pair`, `cooccurrence`, `prefix` or `docFreq`.

The route needs a permission record linked to the caller's group like any public read; a new instance links it to the guest group.

## Which words are never corrected

- A word that exists in the list, as written or as another form of the same word: `paying` is kept when any form of `pay` is present.
- Words shorter than 4 letters, stop words of the language, and question and function words such as `how`, `what`, `who`, `are`, `you`, `get`, `do`.
- Words on the project's `doNotCorrect` list.
- Nonsense: a candidate has to be within two edits. Words of up to 4 letters accept one edit, and a second only when the candidate appears in one document together with the rest of the query.

Among candidates, one that appears together with the other words of the query wins, then the closer one, then a longer shared beginning, then the more frequent.

## Correction inside the public searches

Three of them correct: `GET /api/content/files/search`, `/pages/quick/search` and `/products/quick/search`. Each **corrects first and then searches the corrected words**, and each corrects only against its own section's words — so a typo of a word that exists only in a document corrects in the document search and nowhere else. The results you get back are the correction's results, not the typo's, so never present them as results for what the visitor typed. `/blocks/quick/search` takes no `correct` parameter and never corrects.

`files/search` reports it in the body. When a correction exists the answer carries `query`, `correctedQuery` and the warning `corrected` — including when the corrected search finds nothing, so a `total` of zero still tells you which words were searched, and `fuzzy_fallback` does not run in that case. With no correction, `correctedQuery` is absent and `fuzzy_fallback` can still appear as the last resort. `query` is always present.

The two quick searches keep a plain array as the body and report the correction in response headers instead: `X-Search-Query` carries the query as written, `X-Corrected-Query` the correction, and `X-Search-Warnings` the value `corrected`. The two query headers are URI-encoded — a space arrives as `%20` — and all three are named in `Access-Control-Expose-Headers`, so a browser can read them. A query that needed no correction carries none of the three. The last word of the query is kept when it is the beginning of a known word, so a half-typed word still matches.

`correct=false` on any of the three searches as written: no `correctedQuery`, no headers. That is what a "search as written instead" link should send.

## Tuning correction for a project

A `vocabulary` block inside the `fileContentSearch` section of the general settings, sent through the general settings update like any other section:

```json
{ "fileContentSearch": {
    "vocabulary": {
      "enabled": true,
      "minWordLength": 4, "maxWordLength": 24,
      "fileOnlyMinFiles": 2, "fileOnlyMinOccurrences": 3,
      "doNotCorrect": ["milltown"],
      "corrections": [ { "from": "watr", "to": "water" } ] } } }
```

`enabled: false` stops queries being corrected at all. Each limit has a range the write enforces — `minWordLength` 2 to 10, `maxWordLength` 10 to 64, `fileOnlyMinFiles` 1 to 100, `fileOnlyMinOccurrences` 1 to 1000. A value outside its range answers `400` naming the key and stores **nothing** from that call, including the keys that were in range.

Either list takes at most 1000 entries of 1 to 64 characters, and every entry is trimmed and lower-cased on the way in — `"Milltown"` comes back as `milltown`.

`doNotCorrect` protects a word a visitor typed: that word is kept as written instead of being corrected to something else. It does not stop the word being a correction target for a typo of it.

`corrections` are applied before any automatic choice and win over it; `to` may be several words and need not exist in the list at all. Both lists and `enabled` take effect at once; the length limits apply from the next rebuild.

There is no `searchVocabulary` section. A write under that name answers `200` and is kept nowhere, so the read afterwards looks exactly as it did and nothing you set has any effect — one of the silent no-ops in `mcp/docs/api/silent-no-ops`.

To keep one attribute out of the list, set `isInSearchVocabulary: false` on it in the attribute set schema, beside its `type` and `position`. It only matters for `string`, `text` and `textWithHeader` attributes, and the set write re-reads the records of that set, so the change shows up a few seconds later.

## Is the word list up to date

`AdminFileContentController_getStatus` answers a `vocabulary` section: `words`, `builtAt` (last full rebuild, `null` before the first), `pendingUpdates` (changed records not yet reflected), `bySource` (counts under `page`, `product`, `block` and `file`) and `languages`.

A change to a record usually reaches the list within seconds, a delete included, but `pendingUpdates` is the only thing that tells you it has: treat `pendingUpdates: 0` as the evidence that what you just wrote is correctable, and poll it rather than sleeping on a guess.

`AdminFileContentController_rebuildVocabulary` rebuilds the whole list in the background. It takes **no body** at all and answers `{ "enqueued": true }`; it needs `files.contentIndex.manage` but not the document-search plan, and `builtAt` moves once the run has finished and `pendingUpdates` is back at zero.

It is a different operation from `AdminFileContentController_rebuild`, which reprocesses documents. That one's `scope` does not accept `vocabulary` — sending it answers `400` listing the scopes it does accept, and rebuilds nothing.

## Common mistakes

- **Presenting corrected results as results for what was typed.** Each of the three corrects first. Show `correctedQuery` or `X-Corrected-Query` beside the results.
- **Expecting a body field from quick search.** It reports the correction in headers only.
- **Correcting without `scope`.** The search may then be corrected to a word its own section does not contain.
- **Using `doNotCorrect` to stop a word being a correction target.** It only holds a typed word as written.
- **Tuning correction under `searchVocabulary`.** The write answers `200` and changes nothing. The section is `fileContentSearch.vocabulary`.
- **Rebuilding the list with a `scope`.** It is its own operation and takes no body.
