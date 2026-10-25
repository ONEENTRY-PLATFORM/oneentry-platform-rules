# Search query correction

The instance keeps its own list of the words used in its content and corrects typos in public search queries against it: `fish ladr` becomes `fish ladder`, and the search then runs on the corrected words. A site no longer needs its own copy of the word list, and a newly published document becomes a correction target without any action from the site.

Read it when building a "Showing results for …, search as written instead" line, when a public search answers with a corrected query, or when a correction is wrong.

→ `mcp/docs/api/file-content-search#the-public-search-is-a-narrower-surface-with-the-same-shape` · `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Which words can be a correction target

Words come from what a visitor may read, per language:

- titles of visible pages, products and blocks;
- values of their `string`, `text` and `textWithHeader` attributes, with markup removed;
- text of attached documents that are searchable, attached to a visible page, product or block, in a section enabled in `fileContentSearch.ownerTables`.

Never a target: anything from user profiles, hidden records and their documents, documents excluded from search or marked low confidence, and attributes of any other type — numbers, dates, codes, files, lists. Attributes marked as login, password, SKU, price or currency, and string attributes validated as an email or URL, are left out too.

Only words of letters, 4 to 24 characters long by default. A word that occurs only in documents counts when it appears in at least 2 documents or at least 3 times, which keeps scanning noise out.

## Correct a query directly

`GET /api/content/search/correct` with `x-app-token`.

```http
GET /api/content/search/correct?q=fish%20ladr&langCode=en_US&scope=pages,files

{ "query": "fish ladr", "corrected": "fish ladder",
  "tokens": [ { "from": "fish", "to": null },
              { "from": "ladr", "to": "ladder", "distance": 2, "docFreq": 8, "reason": "cooccurrence" } ] }
```

- `q` is required; a missing one answers `400`.
- `langCode` picks the language. A language with no words answers `200` with `"corrected": null`, never an error.
- `scope` limits targets to words that exist in at least one of `pages`, `products`, `blocks`, `files`. Pass the section you are about to search, so a product search is never corrected to a word that exists only in a document. Any other value answers `400`.
- `"corrected": null` means there is nothing to correct. Every word of the query is listed in `tokens`; `to: null` means kept as written.
- `reason` is `cooccurrence`, `distance`, `prefix`, `frequency` or `manual`.

The route needs a permission record linked to the caller's group like any public read; a new instance links it to the guest group.

## Which words are never corrected

- A word that exists in the list, as written or as another form of the same word: `paying` is kept when any form of `pay` is present.
- Words shorter than 4 letters, stop words of the language, and question and function words such as `how`, `what`, `who`, `are`, `you`, `get`, `do`.
- Words from the project's never-correct list.
- Nonsense: a candidate has to be within two edits. Words of up to 4 letters accept one edit, and a second only when the candidate appears in one document together with the rest of the query.

Among candidates, one that appears together with the other words of the query wins, then the closer one, then a longer shared beginning, then the more frequent.

## Correction inside the public searches

`GET /api/content/files/search` corrects first. When a correction exists and the search on it finds anything, the answer carries `query`, `correctedQuery` and the warning `corrected`, and the results are those of the corrected words. Otherwise the search runs as written and `correctedQuery` is absent from the answer altogether; `fuzzy_fallback` can still appear then as the last resort.

`GET /api/content/pages/quick/search` and `GET /api/content/products/quick/search` search as written first, since a half-typed word must keep its matches. Only an empty result is retried with the corrected query. The body stays a plain array; the correction is reported in the response headers `X-Corrected-Query` (URI-encoded) and `X-Search-Warnings: corrected`, both readable from a browser.

`correct=false` on any of the three searches as written, which is what a "search as written instead" link should send.

## Tuning correction for a project

A `searchVocabulary` section of the general settings, sent through the general settings update like any other section:

```json
{ "searchVocabulary": {
    "neverCorrect": ["milltown"],
    "corrections": [ { "from": "watr", "to": "water" } ],
    "minWordLength": 4, "maxWordLength": 24,
    "fileOnlyMinFiles": 2, "fileOnlyMinOccurrences": 3 } }
```

`corrections` are applied before any automatic choice; `from` is one word of letters, `to` is words of letters separated by single spaces. The length limits apply from the next rebuild of the list; everything else applies at once.

To keep one attribute out of the list, set `inSearchVocabulary: false` on it in the attribute set schema. It only matters for `string`, `text` and `textWithHeader` attributes.

## Is the word list up to date

`AdminFileContentController_getStatus` answers a `vocabulary` section: `words`, `builtAt` (last full rebuild, `null` before the first), `pendingUpdates` (changed records not yet reflected), `bySource` (words present in pages, products, blocks and documents) and `languages`.

A change to a record reaches the list within seconds. `pendingUpdates` above zero for minutes means the list is behind. `AdminFileContentController_rebuild` with `{ "scope": "vocabulary" }` rebuilds it in the background and answers `enqueued: 1`; it needs `files.contentIndex.manage` but not the document-search plan, and `builtAt` moves when it is done. A full rebuild also runs every night.

## Common mistakes

- **Showing a correction the search did not use.** Show `correctedQuery` or the header only; a separate call to the correct route may be ahead of the search you displayed.
- **Expecting a body field from quick search.** It reports the correction in headers only.
- **Correcting without `scope`.** The search may then be corrected to a word its own section does not contain.
- **Adding a real word to `corrections` to stop a correction.** Use `neverCorrect`.
