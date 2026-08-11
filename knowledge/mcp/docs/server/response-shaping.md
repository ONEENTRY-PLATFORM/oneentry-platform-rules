# How responses are shaped before you see them

The server caps what it hands to the model, strips binary-looking values, and summarises dry-run targets. None of that is the API limiting itself — it is this server protecting your context.

Knowing the shapes means you can tell "there is no more data" from "you were given part of it".

→ `mcp/docs/server/cms-api-call` · `mcp/operating-rules#truncated-responses-are-deliberate`

## The size cap

24576 bytes by default, adjustable with `--max-response-bytes`. A response under the cap is passed through untouched and `truncated` is `false`.

Raising the cap is an operator decision and rarely the right fix. A response that large is usually a query that should have been narrower.

## A truncated list

When the body is an array, you get a prefix of it plus a report:

```json
{ "items": [ … 30 items … ],
  "_truncated": { "shown": 30, "total": 214,
                  "hint": "Narrow the result with the operation's own limit/offset/filter parameters." } }
```

`shown` and `total` are exact, so you always know how much you are missing.

## A truncated envelope

When the body is an object containing arrays — the usual paginated shape — the largest array is truncated in place and the surrounding metadata is preserved:

```json
{ "total": 214, "page": 1,
  "items": [ … ],
  "_truncated": { "key": "items", "shown": 30, "total": 214, "hint": "…" } }
```

The envelope's own counts stay honest, which is what lets you page correctly from a truncated first response.

## When the body is neither

If the response is a single large object with nothing list-shaped to cut, you get a preview instead:

```json
{ "_truncated": { "hint": "Response exceeded 24576 bytes and is not a list; request a narrower projection.",
                  "preview": "{\"id\":512,\"localizeInfos\":…" } }
```

The preview is valid text but not necessarily valid JSON — it is cut at a byte boundary. Treat it as something to read, not to parse.

## What to do about truncation

Narrow the request. In order of preference:

1. Use the operation's `limit` and `offset` to page deliberately.
2. Use its filters to ask for the subset you actually need.
3. Use a quick-search or by-id operation instead of a full listing, when one exists.

What not to do: call the same operation again hoping for a different result, or conclude that `total` items do not exist because you only saw 30.

## Binary values are stripped

A string longer than 512 characters that looks like encoded binary — a data URI, a long base64 run, or a value under a key like `base64`, `buffer`, `content`, `fileContent` or `blob` — is replaced:

```json
{ "fileContent": "[stripped 184320 chars]" }
```

The rest of the object survives intact. This stops one embedded file from consuming the entire response budget, and it means an upload response still tells you the id, name and URL of what you uploaded.

If you genuinely need the bytes, fetch the file from its URL outside this server.

## Dry run targets are summarised separately

The `target` in a dry run has its own, smaller budget of about 4 KB, because its job is to let a human recognise what is about to change — not to deliver the data.

```json
{ "_summary": { "kind": "list", "count": 214, "ids": [1, 2, 3, "…"],
                "hint": "Summarised: read the target in full with a separate GET if you need it." } }
```

`kind` is `list`, `object` or `value`. A summarised object shows its keys and its identifying field; a summarised scalar shows the first 200 characters. Arrays nested inside an object are collapsed to counts and sample ids.

If the summary is not enough to be sure, read the entity in full with a separate call before confirming anything.

→ `mcp/docs/server/confirm-and-dry-run#reading-the-dry-run-target`

## Truncation is never an error

A truncated response has `status: 200` and `truncated: true`. It is a successful call whose result was clipped for display. Do not report it to the human as a failure, and do not treat `_truncated` as an error envelope.
