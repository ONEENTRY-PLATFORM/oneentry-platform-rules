# Searching inside attached documents

Admin-side search over the **text of files** attached to records as attribute values — a price list, a warranty, a specification. A separate surface from the entity search: its own operations, own permissions, own unit of result.

Read this when a human asks to find a document by something written inside it, or when a search that should match a document returns nothing.

→ `mcp/docs/api/global-search#what-global-search-never-finds` · `mcp/docs/api/files-and-uploads#referencing-a-file-from-an-attribute` · `mcp/docs/api/settings#changing-any-setting`

## Nothing is indexed until someone turns it on

Document processing is **off on a new instance and stays off after an upgrade**. Until it is enabled, no file is ever read, and the search answers an empty list.

Enable it in general settings, under a `fileContentSearch` section. Send the section whole: a partial body replaces the keys you send and keeps the rest, so read before you write.

Enabling it does **not** pick up files that were already attached before that moment. Coverage stays at zero, and the search stays empty, until something reprocesses them. After enabling, call `AdminFileContentController_rebuild` with `{ "scope": "missing" }` once. It finds the files already held in attribute values of the enabled owner sections and accepts them for processing. Files attached or changed later are picked up without it.

On a large instance that one call accepts the whole existing corpus at once. Warn the user before calling it there, since processing a large corpus takes a long time.

```jsonc
// PUT general settings, body
{
  "fileContentSearch": {
    "enabled": true,
    "formats": ["txt", "md", "html", "pdf", "docx", "odt", "epub", "rtf"],
    "maxCharsPerFile": 500000,
    "ownerTables": ["products", "pages", "blocks", "slides", "templates", "discounts", "forms", "events"],
    "ocr": { "enabled": false, "maxPages": 50 },
    "language": { "mode": "auto", "fixed": "english", "allowManualOverride": true },
    "search": {
      "queryLanguageMode": "auto",
      "enableInfixStage": true,
      "enableSemanticStage": false,
      "enableFuzzyStage": false
    }
  }
}
```

Turning it **off** later does not delete anything. Existing documents stay searchable; only new work stops.

## Which sections can have their files searched

`ownerTables` decides which sections are read. It accepts every section whose records carry an attribute set, because that is where a file attribute can live:

`products`, `pages`, `blocks`, `slides`, `templates`, `discounts`, `forms`, `events`, `orders`, `form_data`, `user_groups`, `users`, `admins`

The first ten are on by default. `users`, `user_groups` and `admins` are **accepted but off**, and stay off through an upgrade: the documents there belong to customers and staff, and reading them into a search index is a decision somebody makes rather than something an upgrade does. Add them only when a human asked for exactly that, and say what it means first — every admin holding `files.contentSearch` and that section can then read the text of those documents.

Files attached to orders and to form submissions are searchable on the admin side, as the `orders` and `form_data` sections. They are never searchable publicly, whatever `ownerTables` says: a visitor's own order attachments are not separated from anyone else's there, so the public search leaves both sections out. Existing attachments of those two sections are picked up the next time the order or the submission is saved, or by a `missing` rebuild.

A section left out of `ownerTables` is not read at all: no index entry appears for its files, so an absent entry is the expected answer rather than a sign that processing failed. Turning the section on and then saving the record again is what creates the entry.

Each section is also filtered per admin: a result names its owner record, and both the result and that name are limited to the sections the admin can reach. Two admins can get different result counts for the same query, and neither is wrong.

## Two permissions nobody holds yet

`AdminFileContentController_search` and the index listings need `files.contentSearch`. Rebuilding and editing index entries need `files.contentIndex.manage`.

The first admin of the instance holds both. **No other admin holds either until someone grants them**, and a `403` here is almost always that rather than a misconfigured feature.

`AdminFileContentController_getStatus` needs **neither** key and answers for any admin. Call it first: it tells you whether this instance can search file content at all, so a `403` from the search or the listings means the grant is missing rather than the feature being unavailable here. Report those two as different things — one is a grant a human can make, the other is not.

They are deliberately separate from `files.create` and `files.delete`: being allowed to upload a file is not being allowed to read the text of every document on the instance. Granting the upload permissions does nothing for this search.

→ `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## The public search is a narrower surface with the same shape

`GET /api/content/files/search` searches the same document text for a visitor. It answers the same envelope — `total`, `offset`, `limit`, `queryLanguage`, `warnings`, `items` — with the same snippet convention and the same pagination ceiling, so everything below about languages, warnings and snippets applies unchanged.

Four differences decide whether a call works:

- **`q` must be at least three characters.** Two are accepted on the admin side and answer `400` here.
- **Only records a visitor may read contribute their files.** A record hidden from the site contributes nothing, and neither does one its groups may not read. Sections the instance holds for staff and customers are narrowed to the caller: the administrator section is never searched publicly at all, and the user and user-group sections return only the caller's own records, so a guest gets nothing from them.
- **A result carries fewer fields.** `id`, `storageKey`, `extension`, `title`, `langCode`, `pageCount`, `rank`, `snippet`, `owners`, `ownersTotal` — and nothing describing the state of the index. Do not expect `status`, `isExcluded`, `truncated`, `lowConfidence`, `langSource`, `langConfidence` or `tsConfig` here; read those from the admin index listing.
- **`ownerTables` in settings still decides.** What the instance does not process is not searchable publicly either, and the same request may be filtered further by the permission record's own section restrictions.

The route needs a permission record for its path linked to the caller's group, like every public read. Without it the call answers `403` naming the route rather than an empty list — grant it the same way as any other public route. A new instance links it to the guest group.

**Admin and public results differ legitimately and neither verifies the other.** A document found as an administrator and missing for a visitor usually means its record is hidden or closed to that group, not that the document fell out of the index. Check the record before touching anything about processing.

→ `mcp/docs/api/content-api-permission-rules#give-a-group-a-route-it-does-not-have-yet`

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
- `semantic_unavailable` — meaning-based matching is switched on but could not answer for this call, so only word matching ran. The result is narrower than it should be, not wrong; say so instead of reporting an empty list as final.
- `fuzzy_fallback` — nothing matched the query as written and these results come from near-spellings of it. Offer them as "did you mean", and expect the snippet to show a word the human did not type.

## Why a document you attached is not found

First check that the document has an entry at all. **No entry** means no reader ever looked at the file, and the usual cause is the owner section being absent from `ownerTables` — see the section above. An entry that exists tells a different story.

Check its entry in `GET /files/content-index` before concluding the search is at fault. Each entry carries `owners` and `ownersTotal` in the same shape as a search result, so you can open the referencing record and check the attribute value itself. Each entry also carries a `status`, and several of them mean the text was never obtained:

- `pending` — not processed yet, or the format is not enabled in settings.
- `done` — searchable. With `truncated: true` only the first part of a long document is.
- `unsupported` — the format is not enabled for this instance.
- `no_extractor` — this instance has no reader for that format.
- `too_large`, `encrypted`, `corrupt`, `type_mismatch`, `archive_rejected` — properties of the document itself. Retrying changes nothing; these are answers.
- `no_text_layer` — pages, but no text in them: a scan. Retrying changes nothing, recognition can.
- `failed` — worth one rebuild.

A `tsConfig` of `simple` means the document's language has no word-form matching here: only the exact word will match, not its other forms.

`lowConfidence: true` on an entry means the text was obtained but came out questionable — garbled encoding, run-together words. The document is searchable and ranks below clean ones. List only those entries with `lowConfidence=true`. Say so when you offer it; do not quote it as if it read cleanly.

## Making a scan readable one document or a whole slice

A document sitting at `no_text_layer` is almost always a scan: it has pages and no text in them. `AdminFileContentController_patch` takes `ocrRequested: true` for a single index entry and queues that one document for character recognition.

One document at a time is the point. Recognition costs on the order of a second per page, so turning it on for a whole corpus is days of work, while one document a human actually needs is seconds. Read `capability.ocrAvailable` first — where recognition is unavailable the call answers `400` rather than accepting work it cannot do.

`capability.ocrAvailable` only says recognition runs here. Check `capability.ocrLanguages` as well: it lists the language codes this instance can recognise, and a document written in a language outside that list comes back as text that reads as nonsense rather than as a failure. Such an entry is usually marked `lowConfidence: true` and is worse than no text at all, because it is searchable and wrong. Where the language is not covered, say so instead of queueing the document.

Recognition is much slower than ordinary reading and runs apart from it, so the rest of the corpus keeps processing meanwhile. Re-read the entry to see the outcome instead of sending the call again.

When every scan on the instance has to be read — after recognition first becomes available, say — `AdminFileContentController_rebuild` with `{ "scope": "ocr" }` asks for the whole slice at once, and queues each entry in it for recognition the way the single-entry call queues one. It takes only the documents sitting at `no_text_layer` and skips the ones excluded from search. Its answer counts the entries it actually queued, and only those entries carry the recognition request; where it queues nothing the slice stays as it was. It answers `400` in the same case the single-entry call does, when `capability.ocrAvailable` is not true. Prefer the single entry while a human is waiting on one document: the slice is charged a second per page over every scan in it.

Sending `{ "scope": "ocr" }` a second time does not ask for the same scans twice while the first request is still outstanding. A scan leaves the slice for as long as recognition has been asked for it and has not finished, so a repeat sent during a run accepts nothing it accepted before. This matters because the slice keeps a scan at `no_text_layer` until recognition finishes and finds text, and because `processing.rebuild.remaining` reaches `0` long before recognition has worked through the queue: the run looks finished when it is not. Wait and re-read the entries rather than sending the call again.

Once recognition has finished on a scan and still found no text, that scan is back in the slice. This is what makes the scope usable after the instance gains a language it could not read before, or after its reader is upgraded: send `{ "scope": "ocr" }` again and every such scan is asked for once more, without a call per document. Do not send it while the previous run is still working — you would only be waiting on the same answer. To ask again for one document on its own, send the single-entry `ocrRequested: true` for the entry you want.

A zero in the answer means nothing was taken, and nothing in the slice was changed. Recognition also needs processing enabled and space left in the index, so `{ "scope": "ocr" }` answers `201` with zero accepted while `processing.enabled` is false or `storage.overBudget` is true. Read both fields from the status before reading a zero as "there was nothing left to recognise": clear the condition and the same call takes the slice.

The single-entry call is subject to the same condition, and says as little about it: while `processing.enabled` is false it answers `200` with `ocrRequested: true` on the entry, and no recognition is queued. Read `processing.enabled` from the status before reading that answer as work accepted. Nothing is lost by it — the entry stays in the `ocr` slice, so once processing is enabled `{ "scope": "ocr" }` takes it along with the rest and there is no need to repeat the single-entry call document by document.

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
  "capability": { "tariffAllows": true, "extractorAvailable": true, "extractorReason": "ok", "ocrAvailable": true, "ocrLanguages": ["en", "ru"], "embeddingAvailable": true, "semanticStageEnabled": false, "fuzzyStageEnabled": false, "formats": ["txt", "md", "html", "pdf", "docx", "odt", "epub", "rtf"] },
  "processing": { "enabled": true, "queuePaused": false, "pending": 0, "rebuild": null },
  "coverage": { "files": 2, "done": 1, "failed": 0, "truncated": 0, "lowConfidence": 0, "noExtractor": 0, "unsupported": 1, "corrupt": 0, "excluded": 0 },
  "storage": { "indexBytes": 311296, "budgetBytes": 536870912, "overBudget": false },
  "languages": [{ "code": "en", "tsConfig": "english", "files": 1 }]
}
```

`coverage` counts only files that a record in a section **this admin can reach** references — exactly the rows `GET /files/content-index` lists for the same filter. Two admins can see different coverage, and neither is wrong. `done` means searchable now, so it leaves out manually excluded documents; its listing is `status=done&excluded=false`, where `excluded=false` is a filter and not the absence of one.

`embeddingAvailable` says whether meaning-based matching can answer on this instance at the moment of the call. It is not the same as it being used: `semanticStageEnabled` and `fuzzyStageEnabled` say whether the two optional matching stages are switched on in settings, and both are off until someone turns them on. Report the pair, never one of them — available and unused is the state a human most often mistakes for broken.

`formats` is what this instance can actually read, which is not the same list as the `formats` setting — that one is what the instance is allowed to read. A format present in the setting and absent here is why a document sits at `no_extractor`. An empty list means nothing answered about reading at all; read `extractorReason` then.

`tariffAllows` and `extractorAvailable` are separate on purpose and must be reported separately to a human: not on your plan and no reader available on this instance need different answers.

`extractorReason` says why reading is unavailable, and the three causes need three different answers — `extractorAvailable: false` alone sends every one of them to the same wrong place:

- `ok` — reading works.
- `not_configured` — this instance was never told where the reader is. A human with access to the instance configuration fixes it.
- `unauthorized` — the reader answered and refused the call. A shared credential does not match on the two sides; also configuration, but a different field.
- `unreachable` — nothing answered, or the reader failed. Operational, and nothing a tenant can do.

Report the reason you were given rather than "unavailable". An unfamiliar value means a newer instance than you know about: say it verbatim instead of guessing. `languages` is what the documents here are actually written in — use it to offer a language choice that cannot be empty.

`ocrLanguages` is the separate answer for character recognition: the language codes this instance can recognise in a scan. It is not the same list as `languages`, which is what the already-indexed documents are written in, and it is not implied by `ocrAvailable`. A locale the instance serves may still be absent from it.

Two fields here explain a corpus that has quietly stopped growing while nothing reports an error:

- `storage.overBudget: true` — the index has filled the space allowed it. New documents are no longer accepted and stay at `pending`; everything already indexed stays searchable and nothing is deleted. No amount of reprocessing moves a document out of this. Space has to be freed, or the allowance raised.
- `processing.queuePaused: true` — processing is held on the instance itself. This is **not** the tenant's `enabled` setting and toggling that setting will not release it. Report it and stop; turning `enabled` off and on again only changes the tenant's own switch.

## Reprocessing a slice of the corpus

`AdminFileContentController_rebuild` takes `{ "scope": … }` and answers how many documents it accepted for reprocessing.

- `missing` — never processed, including files attached before processing was enabled, plus every
  document a retry can still finish: failures, and documents whose format had no reader when they
  were first read. Use it after a reader becomes available on the instance — it is the bulk path
  back for that slice, and `all` is not needed for it. A document waiting for recognition is not
  in this slice — `ocr` is the scope for those, and no other scope recognises one.
- `failed` — only failures. Document properties such as encrypted or unsupported are not retried.
- `stale` — processed by an older reader than this instance now has.
- `all` — everything.
- `ocr` — every scan not currently waiting on a recognition request, excluded documents aside:
  scans never asked for, and scans whose last recognition finished and still found no text. A
  scan with a request outstanding is left alone, so a repeat sent mid-run accepts nothing it
  accepted before. The only scope that recognises anything: the others read a scan again and
  leave it exactly where it was. Needs `capability.ocrAvailable`, and answers `400` without it.
- `vectors` — no document is read again. It fills in meaning-based matching for text already held, which is what the corpus indexed before that matching existed lacks. Its answer counts fragments completed rather than documents accepted, and it works in batches: call it again while the number keeps coming back above zero. Nothing else recovers that slice, and until it is done a meaning-only query finds nothing however the setting is switched.
- `one` — a single document; `storageKey` is then required, and omitting it answers `400`.

The answer is an acceptance, not a result. Follow it in `processing.rebuild` of the status: `total` is how many documents the last rebuild accepted, `remaining` how many still wait, and `startedAt` when it began. `remaining: 0` means that run is finished. `remaining` can include other documents waiting at the same time, so it never exceeds `total`. The field is `null` when no rebuild ran in the last day. A rebuild that accepts nothing does not replace a run whose documents are still waiting, paused processing included: the status keeps that run's `total` and `startedAt`. Once nothing is waiting, the same call shows `total: 0`. Poll every few seconds, not in a tight loop, and do not start another rebuild while `remaining` is above zero.

## Matching by meaning and past a typo

Two optional stages sit beside word matching, both off until settings turn them on, and both reported in `capability`.

**Meaning-based matching** finds a document that carries what was asked about while sharing no word with the query — "return conditions" reaching a document that says "how to exchange an item". Switch it on with `search.enableSemanticStage`, but only after `AdminFileContentController_rebuild` with `{ "scope": "vectors" }` has finished for the existing corpus: until then the stage has nothing to match against and the search behaves exactly as before, which reads as the setting doing nothing. Check `capability.embeddingAvailable` first — where it is false the stage is switched on and silently idle, and every answer carries `semantic_unavailable`.

The two stages are merged, not chosen between: a document found either way appears once, and the unit of a result stays the whole file. `rank` then orders results by how well each did across both stages, so compare it only inside one answer — never between two answers, and never as a score.

**Typo tolerance** (`search.enableFuzzyStage`) runs only when nothing matched at all, and marks its answer with the `fuzzy_fallback` warning. It cannot widen a query that already found something, so switching it on never changes a working search.

Both stages obey everything else on this page unchanged: the same sections, the same per-admin filtering, the same pagination ceiling, the same snippet convention.

## Common mistakes

- **Expecting results before enabling processing.** Off by default, and off after an upgrade.
- **Expecting existing attachments to appear on their own after enabling.** Run one `missing` rebuild.
- **Granting `files.create` to fix a `403`.** Uploading and reading text are separate permissions.
- **Rendering `snippet.text` as markup.** It is document text with control-character marks.
- **Hiding `queryLanguage`.** A silent language choice is how a working search gets called broken.
- **Reading an empty list without reading `warnings`.**
- **Retrying an encrypted, corrupt or unsupported document.** The status is the answer.
- **Reaching for `all` to recover documents that had no reader.** One `missing` rebuild takes them.
- **Rebuilding a scan with any scope but `ocr`.** It comes back exactly as it went in.
- **Rebuilding to clear `overBudget`.** Nothing reprocesses out of a full index.
- **Turning recognition on for the whole instance** to read one scan. Ask for the one entry.
- **Treating `truncated` as success.** Only the first part of a long document matches.
- **Re-attaching a file that has not appeared yet.** That is a second reference, not a retry.
- **Turning meaning-based matching on without the `vectors` rebuild.** The setting alone changes nothing.
- **Reading an empty answer carrying `semantic_unavailable` as final.** Only word matching ran.
- **Paging past `offset + limit` of 200.** Narrow the query instead.

→ `mcp/docs/api/files-and-uploads#referencing-a-file-from-an-attribute` · `mcp/docs/api/index-attributes#what-an-index-attribute-is`
