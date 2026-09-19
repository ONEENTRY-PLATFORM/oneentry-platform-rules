# Searching inside attached documents

Admin-side search over the **text of files** attached to records as attribute values — a price list, a warranty, a specification. A separate surface from the entity search: its own operations, own permissions, own unit of result.

Read this when a human asks to find a document by something written inside it, or when a search that should match a document returns nothing.

→ `mcp/docs/api/global-search#what-global-search-never-finds` · `mcp/docs/api/files-and-uploads#referencing-a-file-from-an-attribute` · `mcp/docs/api/settings#changing-any-setting`

## Nothing is indexed until someone turns it on

Document processing is **off on a new instance and stays off after an upgrade**. Until it is enabled, no file is ever read, and the search answers an empty list.

Enable it in general settings, under a `fileContentSearch` section. Send the section whole: a partial body replaces the keys you send and keeps the rest, so read before you write.

```jsonc
// PUT general settings, body
{
  "fileContentSearch": {
    "enabled": true,
    "formats": ["txt", "md", "html", "pdf", "docx", "odt", "epub", "rtf"],
    "maxCharsPerFile": 500000,
    "ownerTables": ["products", "pages", "blocks", "slides", "templates", "discounts"],
    "ocr": { "enabled": false, "maxPages": 50 },
    "language": { "mode": "auto", "fixed": "english", "allowManualOverride": true },
    "search": { "queryLanguageMode": "auto", "enableInfixStage": true }
  }
}
```

Turning it **off** later does not delete anything. Existing documents stay searchable; only new work stops.

## Two permissions nobody holds yet

`AdminFileContentController_search` and the index listings need `files.contentSearch`. Rebuilding and editing index entries need `files.contentIndex.manage`.

Both are **new keys that no admin holds until someone grants them**, including the first admin of the instance. A `403` here is almost always that, not a misconfigured feature.

`AdminFileContentController_getStatus` needs **neither** key and answers for any admin. Call it first: it tells you whether this instance can search file content at all, so a `403` from the search or the listings means the grant is missing rather than the feature being unavailable here. Report those two as different things — one is a grant a human can make, the other is not.

They are deliberately separate from `files.create` and `files.delete`: being allowed to upload a file is not being allowed to read the text of every document on the instance. Granting the upload permissions does nothing for this search.

→ `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## A result is a whole file and not a fragment

`GET /files/search` takes `q` (two characters or more) and answers files. One file appears once however many times the term occurs inside it.

```jsonc
{
  "total": 1,
  "offset": 0,
  "limit": 10,
  "queryLanguage": { "resolved": ["english"], "source": "explicit", "candidates": ["english"] },
  "warnings": [],
  "items": [
    {
      "id": 138,
      "storageKey": "files/project/product/123/docs/9f2c.txt",
      "extension": "txt",
      "title": null,
      "langCode": "en",
      "tsConfig": "english",
      "langSource": "detected",
      "langConfidence": 0.95,
      "pageCount": 1,
      "truncated": false,
      "isExcluded": false,
      "status": "done",
      "rank": 0.07,
      "snippet": { "text": "…", "pageFrom": 1 },
      "owners": [{ "tableName": "products", "dataId": 123, "title": "Price list", "langCode": "en_US" }],
      "ownersTotal": 1
    }
  ]
}
```

`owners` is the first few records that reference the file and `ownersTotal` is how many there are in all — one document is routinely attached to many records. `title` is the document's own title where the format carries one and `null` where it does not; the original file name is not recoverable, so label a row by its owner.

Filters `extensions`, `ownerTables`, `localeCodes` and `onlyTruncated` are comma-separated. `offset + limit` may not exceed 200; beyond that the call answers `400`.

## The snippet marks matches with control characters

`snippet.text` wraps each match in `U+0002` … `U+0003`. It is **not** markup, and that is deliberate: the text comes from a document nobody on the instance wrote, so returning markup would put document content into whatever renders it.

Split the string on those two characters and emphasise the parts between them. Never hand `snippet.text` to anything that interprets markup, and never assume the matched span equals what the human typed — it will not, because matching is by word form.

## Always tell the human which language was searched

`queryLanguage` comes back on every response and is the difference between a working search and one that looks broken. A single word does not reveal its language, so the instance resolves it from the writing system of the query and from the languages the documents here are actually in.

- `source: "explicit"` — the caller passed `langCode` and it was used.
- `source: "script"` or `"facet"` — resolved from the query's writing system, then narrowed by what exists here.
- `source: "configured"` — taken from the instance setting.
- `source: "fallback"` — nothing resolved; matching is literal, with no word-form matching.

Pass `langCode` when the human tells you the language. An unknown value is refused with `400` and the message lists what this instance accepts, so read that list rather than guessing.

## An empty result that has a reason says so

`warnings` is not decoration. An empty list because processing is off looks exactly like an empty list because nothing matched, and only this field tells them apart.

- `processing_disabled` — the index is not being kept up to date. Counts may be stale.
- `query_too_common` — the query was made only of words too common to match on. Ask for a more specific term; do not retry.
- `query_language_capped` — more candidate languages than the instance will search at once. Pass `langCode` to pick one.

## Why a document you attached is not found

Check its entry in `GET /files/content-index` before concluding the search is at fault. Each entry carries a `status`, and several of them mean the text was never obtained:

- `pending` — not processed yet, or the format is not enabled in settings.
- `done` — searchable. With `truncated: true` only the first part of a long document is.
- `unsupported` — the format is not enabled for this instance.
- `no_extractor` — this instance has no reader for that format.
- `too_large`, `encrypted`, `corrupt`, `type_mismatch`, `archive_rejected` — properties of the document itself. Retrying changes nothing; these are answers.
- `no_text_layer` — pages, but no text in them: a scan. Retrying changes nothing, recognition can.
- `failed` — worth one rebuild.

A `tsConfig` of `simple` means the document's language has no word-form matching here: only the exact word will match, not its other forms.

`lowConfidence: true` on an entry means the text was obtained but came out questionable — garbled encoding, run-together words. The document is searchable and ranks below clean ones. Say so when you offer it; do not quote it as if it read cleanly.

## Making a scan readable one document at a time

A document sitting at `no_text_layer` is almost always a scan: it has pages and no text in them. `AdminFileContentController_patch` takes `ocrRequested: true` for a single index entry and queues that one document for character recognition.

One document at a time is the point. Recognition costs on the order of a second per page, so turning it on for a whole corpus is days of work, while one document a human actually needs is seconds. Read `capability.ocrAvailable` first — where recognition is unavailable the call answers `400` rather than accepting work it cannot do.

Recognition is much slower than ordinary reading and runs apart from it, so the rest of the corpus keeps processing meanwhile. Re-read the entry to see the outcome instead of sending the call again.

## Attaching a document does not index it instantly

A file becomes searchable a while after the attribute value referencing it is saved — longer on a large instance. Re-read the index entry to confirm rather than attaching the file a second time; a second attachment creates a second reference, not a second attempt.

Clearing the attribute value removes the reference, and a file nothing references stops being searchable.

## Fixing a wrong language without reprocessing

`AdminFileContentController_patch` takes `langCode`, `isExcluded` and `ocrRequested` for one index entry.

Changing `langCode` sets `langSource` to `manual` and rebuilds the matching data from the text already held — the document is not read again, and a later reprocessing will not overwrite the correction. Use it when a detected language is wrong, which happens most on short documents and on documents mixing two languages.

`isExcluded: true` takes one document out of the search, leaving the file and its references alone. Use it for a document that should not be searchable by everyone who can search.

Both need `files.contentIndex.manage`. If `language.allowManualOverride` is off, the language change answers `403`.

## Reading coverage before promising a search works

`AdminFileContentController_getStatus` answers what the instance can do and how much of the corpus is actually searchable.

```jsonc
{
  "capability": { "tariffAllows": true, "extractorAvailable": true, "ocrAvailable": true, "embeddingAvailable": false },
  "processing": { "enabled": true, "queuePaused": false, "pending": 0 },
  "coverage": { "files": 2, "done": 1, "failed": 0, "truncated": 0, "lowConfidence": 0, "noExtractor": 0, "unsupported": 1, "corrupt": 0, "excluded": 0 },
  "storage": { "indexBytes": 311296, "budgetBytes": 536870912, "overBudget": false },
  "languages": [{ "code": "en", "tsConfig": "english", "files": 1 }]
}
```

`tariffAllows` and `extractorAvailable` are separate on purpose and must be reported separately to a human: not on your plan and no reader available on this instance need different answers. `languages` is what the documents here are actually written in — use it to offer a language choice that cannot be empty.

Two fields here explain a corpus that has quietly stopped growing while nothing reports an error:

- `storage.overBudget: true` — the index has filled the space allowed it. New documents are no longer accepted and stay at `pending`; everything already indexed stays searchable and nothing is deleted. No amount of reprocessing moves a document out of this. Space has to be freed, or the allowance raised.
- `processing.queuePaused: true` — processing is held on the instance itself. This is **not** the tenant's `enabled` setting and toggling that setting will not release it. Report it and stop; turning `enabled` off and on again only changes the tenant's own switch.

## Reprocessing a slice of the corpus

`AdminFileContentController_rebuild` takes `{ "scope": … }` and answers how many documents it accepted for reprocessing.

- `missing` — never processed, and anything that failed.
- `failed` — only failures. Document properties such as encrypted or unsupported are not retried.
- `stale` — processed by an older reader than this instance now has.
- `all` — everything.
- `one` — a single document; `storageKey` is then required, and omitting it answers `400`.

The answer is an acceptance, not a result. Read coverage again later rather than assuming it finished.

## Common mistakes

- **Expecting results before enabling processing.** Off by default, and off after an upgrade.
- **Granting `files.create` to fix a `403`.** Uploading and reading text are separate permissions.
- **Rendering `snippet.text` as markup.** It is document text with control-character marks.
- **Hiding `queryLanguage`.** A silent language choice is how a working search gets called broken.
- **Reading an empty list without reading `warnings`.**
- **Retrying an encrypted, corrupt or unsupported document.** The status is the answer.
- **Rebuilding to clear `overBudget`.** Nothing reprocesses out of a full index.
- **Turning recognition on for the whole instance** to read one scan. Ask for the one entry.
- **Treating `truncated` as success.** Only the first part of a long document matches.
- **Re-attaching a file that has not appeared yet.** That is a second reference, not a retry.
- **Paging past `offset + limit` of 200.** Narrow the query instead.

→ `mcp/docs/api/files-and-uploads#referencing-a-file-from-an-attribute` · `mcp/docs/api/index-attributes#what-an-index-attribute-is`
