# Moving a lot of content in one run

What changes when a task stops being one write and becomes hundreds: idempotency, verification of every object rather than a sample, and what a report of the result has to contain.

Read this before the first write of a migration. Every rule here is the cheap version of a mistake that was found later by the customer instead.

→ `mcp/docs/api/content-modelling` · `mcp/docs/api/verification-recipes`

## Choose the idempotency key before the first run

A migration run has to survive being run again. Pick the key that identifies an already-imported object — a source identifier, an article number, a page URL — **before** the first write, and do not change it afterwards.

Changing it mid-way without cleaning up what the old key produced is how a second run creates a full duplicate set that then has to be deleted by hand. If the key must change, remove what was written under the old one first.

## Read back every object you wrote

A batch write that reports success can still have missed one object: the response carries no sign of it, and repeating the same call for that object works. So the count of successful responses is not the count of written values.

After a batch, **re-read every entity you touched** and compare the field you set. Retry only the ones that do not match. A sample check scales the wrong way: one miss in a hundred is a handful of empty fields in a catalogue of thousands, and the customer finds them.

→ `mcp/docs/api/silent-no-ops`

## Read many entities in one call

Verifying by id, one request per entity, is what makes agents give up and check a sample. For products there is a read that takes a list of ids and returns them together — use it, compare in one pass, and re-write the mismatches.

```text
cms_api_search { "query": "products by ids", "mutating": false }
```

Where no batch read exists for an entity kind, page the listing and compare against it rather than skipping the check.

## Panel-facing fields cannot be verified by reading

Some structures exist to drive the admin panel: the extra value of a list option, display flags, field settings. The server stores them as opaque values and accepts any shape.

For those, "wrote it, read it back, it matched" proves **nothing** — both the admin and the public read return your own input, wrong shape included, while the panel shows an empty field. Verification means a human opening the screen.

So either get that confirmation, or say plainly that the check is incomplete and name what is unverified. Do not report success on the strength of a read-back.

→ `mcp/docs/api/list-options-and-extra-values#extra-values-sit-in-extended-with-no-locale-key`

## Wait before asserting a calculated value

Aggregates — ratings above all — are calculated after the fact. An empty aggregate immediately after the last write means nothing.

Check, wait, check again, and only then report. A defect report withdrawn a minute later costs more than the wait.

## A success status can carry a login page

An admin session has a short life. Once it lapses, requests can come back with the instance's **login page and a success status** — not a `401`. Parsing then fails on the first character, and code that retries on the status code never retries, because the status is a success.

Through this server that is handled: the call is retried once after re-authenticating, and a second login page is reported as an expired session rather than parsed. Outside this server, treat a body whose first character is `<` as an expired session, stop, and re-verify every write made since the last confirmed read.

## Uploading many files

Files are usually the bulk of a content migration, and they have their own two tools — one for a file on the machine running this server, one for an address it fetches.

Two things to settle before the first upload rather than after the last: the **numeric** preview-template id, because a file stored without one gets no preview and the only repair is uploading it again, and where alternative text will live, because a file record has no field for it.

There is no batch delete for files, so a re-uploaded gallery leaves the previous files in storage, consuming quota. Get it right the first time in preference to re-uploading.

→ `mcp/docs/server/cms-upload-file` · `mcp/docs/api/files-and-uploads`

## Report the result as numbers

"Done" is not a report. What was touched, how much of it matched on re-reading, what is still empty and why, what was left unverified and why:

| line | example |
|---|---|
| written | 120 products, 30 collections |
| verified by re-reading | 119 of 120 |
| retried | 1, after a write that answered success and stored nothing |
| deliberately empty | stock quantity — the source does not publish it, placeholder agreed |
| unverified | option badges — needs a human to open the panel |

A field named in advance is a decision. The same field found by the customer is a defect.

→ `mcp/operating-rules#a-200-means-accepted-not-applied`
