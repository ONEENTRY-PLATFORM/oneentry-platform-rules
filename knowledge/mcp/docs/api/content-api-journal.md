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
| `AdminContentApiErrorsController_clearAll` | `DELETE /journal/content-api-errors` | `journal.delete` |

`journal.get` opens **neither** of them. An admin who reads the change journal without trouble still gets `403 Forbidden resource` here, and the message does not name the key it wanted. The two keys are independent of each other as well: holding the errors key leaves the stats call at `403`. Clearing the error listing is the one write here, and it takes a third key, `journal.delete` — the same one the change journal clear asks for.

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

Every token is checked. Anything other than `4xx`, `5xx` or a code from 400 to 599 answers `400` with a message naming the token, for example `Invalid statusCodes item "junk": expected 4xx, 5xx or a code 400-599`. One bad token in a longer list refuses the whole call, so `statusCodes=4xx,junk` is a `400`, not a filter on `4xx`. Codes below 400, such as `200`, are refused too: only refused requests are recorded here.

`path` matches anywhere inside the recorded path, not just at its start, and ignores case: `path=cart` finds `/api/content/users/me/cart/items/999`. It is a plain substring. `%`, `_` and `\` are matched as those characters, so `path=products_all` finds only paths that contain that exact text.

## One error on its own carries the request context

`GET /journal/content-api-errors/{id}` answers the same record with the request attached: `errorCode`, `errorStack`, `requestId`, `appTokenSerial`, `ip`, `userAgent`, `requestQuery` and `requestBodyPreview` beside the summary fields. Any of them may be `null`.

That detail is the reason to fetch one by id at all — the listing's `errorMessage` is truncated and tells you what was refused, while the detail tells you who asked and with what. An id nothing matches answers `404 Content API error #<id> not found`.

Treat the detail as sensitive: `ip`, `userAgent` and `requestBodyPreview` describe a real visitor's request. Report the shape of a problem, never the record.

## Clearing the error journal takes an optional window

`DELETE /journal/content-api-errors` needs `journal.delete`. The read key `journal.viewContentApiErrors` does not open it: an admin who can only read this listing gets `403 Forbidden resource`, so a read-only grant really is read-only.

The window goes in the query, with the same flat `from` and `to` the listing takes:

```text
DELETE /journal/content-api-errors?from=2026-11-01T00:00:00.000Z&to=2026-11-02T00:00:00.000Z
```

The call answers `200` with the number of records it removed:

```json
{ "deleted": 42 }
```

Send no window and every record on the instance goes; the listing's last-24-hours default does **not** apply here. Send only `from` or only `to` and the open side is unbounded. A window is checked like the listing's: `from` later than `to`, a span wider than 31 days, or a value that is not ISO 8601 answers `400`, and nothing is removed.

`statusCodes` and `path` are refused on this call with `400` rather than ignored: the clear narrows by time only. Nothing on the instance brings removed records back, so read what you need first, and confirm a window by sending it to the listing and checking `total` before you clear it.

## The request count summary and when it answers 503

`GET /journal/content-api-stats` takes one parameter, `period`, with exactly three values — `HOUR_1`, `HOURS_24` and `DAYS_7`. It defaults to `HOURS_24`, so a call with no `period` is the last day. Anything else answers `400 period must be one of the following values: HOUR_1, HOURS_24, DAYS_7`.

This is the one call in the area that depends on something outside the instance's own records. When that source cannot be reached the call answers **`503`**, with the grant in place and the parameter correct — so read a `503` here as the counts being unavailable right now, not as a permission or a parameter problem. Retrying later is reasonable; changing the request is not. Read the body of a successful call rather than assuming its shape.

## Common mistakes

- **Using `journal.get` here.** It opens neither listing; the two keys are separate from it and from each other.
- **Sending `createdAt[from]` and `createdAt[to]`.** Ignored; this listing takes the flat names.
- **Reading an empty listing as no errors.** With no window it covers the last 24 hours.
- **Sending a status class other than `4xx` or `5xx`.** Any other token answers `400`; there is no `2xx` here.
- **Calling the clear with the view key.** It needs `journal.delete`.
- **Calling the clear with no window by accident.** Without `from` and `to` it removes every record.
- **Reading a `503` on the summary as a missing permission.** The grant is fine; the counts are not available.
- **Putting a request detail into a report.** It contains a visitor's address and body.

→ `mcp/docs/api/journal#not-the-same-record-as-the-servers-audit-log` · `mcp/docs/api/silent-no-ops`
