# The public API traffic and error journal

Two listings describe what the **public** API has been doing: a per-route request count for a period, and a record of every request the public API refused. Neither is the change journal, and neither is reached with the key that opens it.

Read this when you need to show what a storefront has been asking for, when you are chasing a `4xx` a site reports and cannot reproduce, or before clearing anything here.

→ `mcp/docs/api/journal#reading-the-journal` · `mcp/docs/api/admins-and-permissions#permissions-are-dotted-keys`

## Two listings and two separate permission keys

| Operation | Path | Key |
|---|---|---|
| `AdminContentApiStatsController_getStats` | `GET /journal/content-api-stats` | `journal.viewContentApiStats` |
| `AdminContentApiErrorsController_getList` | `GET /journal/content-api-errors` | `journal.viewContentApiErrors` |
| `AdminContentApiErrorsController_getById` | `GET /journal/content-api-errors/{id}` | `journal.viewContentApiErrors` |
| `AdminContentApiErrorsController_clearAll` | `DELETE /journal/content-api-errors` | `journal.viewContentApiErrors` |

`journal.get` opens **neither** of them. An admin who reads the change journal without trouble still gets `403 Forbidden resource` here, and the message does not name the key it wanted. The two keys are independent of each other as well: holding the errors key leaves the stats call at `403`.

## Reading the list of refused public requests

The errors listing answers `{ "items": [...], "total": <number> }`, newest first. Each item is a summary — `id`, `createdAt`, `statusCode`, `method`, `path` and `errorMessage`:

```json
{ "items": [ { "id": 119,
               "createdAt": "2026-11-05T06:37:22.663Z",
               "statusCode": 400,
               "method": "DELETE",
               "path": "/api/content/users/me/cart/items/999",
               "errorMessage": "productId must be an integer" } ],
  "total": 74 }
```

`limit` defaults to **30** and is capped at **200** — ask for more and the call answers `400 limit must not be greater than 200`, ask for `0` and it answers `400 limit must not be less than 1`. Page with `offset`.

## This listing takes a flat window not a nested one

`from` and `to` are **flat** here, both ISO 8601, and both optional. Leave them off and the call covers **the last 24 hours only** — so an error from last week is absent from the default answer rather than missing from the record.

```text
GET /journal/content-api-errors?from=2026-11-01T00:00:00.000Z&to=2026-11-05T00:00:00.000Z
```

This is the third naming convention in this area, and carrying a window across from the change journal is the usual mistake: `createdAt[from]` and `createdAt[to]` sent here are accepted and then **ignored**, and the answer is the unfiltered period with nothing to notice. Confirm a window works by asking for a range in the future and seeing `total` come back `0`.

What you do send is checked. A value that is not ISO 8601 answers `400 Invalid period: from/to must be ISO 8601 dates`, a `from` later than the `to` answers `400 Invalid period: from must be <= to`, and a span wider than 31 days answers `400 Invalid period: maximum supported range is 31 days`. There is no way to ask this listing for everything at once.

## Narrowing by status code and by path

`statusCodes` is one comma-separated string, and it mixes two kinds of token: the ranges `4xx` and `5xx`, and exact codes. `statusCodes=4xx,500` means "every client error, plus 500".

A token it does not recognise is **dropped without comment**. Send `statusCodes=junk` and the call answers `200` with the whole unfiltered period rather than an empty list or a `400` — and a typo inside a longer list silently widens the filter instead of narrowing it. Check `total` against the unfiltered one before trusting the result.

`path` matches anywhere inside the recorded path, not just at its start: `path=cart` finds `/api/content/users/me/cart/items/999`. It is a plain substring, so no wildcards are needed and none are interpreted.

## One error on its own carries the request context

`GET /journal/content-api-errors/{id}` answers the same record with the request attached: `errorCode`, `errorStack`, `requestId`, `appTokenSerial`, `ip`, `userAgent`, `requestQuery` and `requestBodyPreview` beside the summary fields. Any of them may be `null`.

That detail is the reason to fetch one by id at all — the listing's `errorMessage` is truncated and tells you what was refused, while the detail tells you who asked and with what. An id nothing matches answers `404 Content API error #<id> not found`.

Treat the detail as sensitive: `ip`, `userAgent` and `requestBodyPreview` describe a real visitor's request. Report the shape of a problem, never the record.

## Clearing the error journal ignores every window

`DELETE /journal/content-api-errors` removes **every** record on the instance. It takes no window: a `from` and `to` in the body or the query are ignored, and the call answers `200` with the count it removed:

```json
{ "deleted": 42 }
```

So unlike the change journal — where the window is the whole safety mechanism and is required — there is no narrow form of this call, and nothing on the instance brings the records back. Read what you need first, then clear, and only with explicit agreement in the conversation.

Note which key guards it: `journal.viewContentApiErrors`, the same key that merely reads. An admin given a read-only grant on this journal can erase all of it, and there is no separate delete key to withhold. Do not treat the view grant as read-only when you decide what an account may do.

## The request count summary and when it answers 503

`GET /journal/content-api-stats` takes one parameter, `period`, with exactly three values — `HOUR_1`, `HOURS_24` and `DAYS_7`. It defaults to `HOURS_24`, so a call with no `period` is the last day. Anything else answers `400 period must be one of the following values: HOUR_1, HOURS_24, DAYS_7`.

This is the one call in the area that depends on something outside the instance's own records. When that source cannot be reached the call answers **`503`**, with the grant in place and the parameter correct — so read a `503` here as the counts being unavailable right now, not as a permission or a parameter problem. Retrying later is reasonable; changing the request is not. Read the body of a successful call rather than assuming its shape.

## Common mistakes

- **Using `journal.get` here.** It opens neither listing; the two keys are separate from it and from each other.
- **Sending `createdAt[from]` and `createdAt[to]`.** Ignored; this listing takes the flat names.
- **Reading an empty listing as no errors.** With no window it covers the last 24 hours.
- **Trusting a `statusCodes` filter without checking `total`.** An unrecognised token is dropped and the filter widens.
- **Expecting the clear to take a window.** It removes everything, whatever you pass.
- **Treating the errors view grant as read-only.** The same key clears the journal.
- **Reading a `503` on the summary as a missing permission.** The grant is fine; the counts are not available.
- **Putting a request detail into a report.** It contains a visitor's address and body.

→ `mcp/docs/api/journal#not-the-same-record-as-the-servers-audit-log` · `mcp/docs/api/silent-no-ops`
