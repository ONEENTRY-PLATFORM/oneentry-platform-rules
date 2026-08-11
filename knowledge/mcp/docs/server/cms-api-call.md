# The cms api call tool

The tool that actually talks to the Admin API. Reads run immediately; mutations go through the allow level, the permission pre-check and, where required, a confirm token.

Never construct an operation id by hand. Ids come from `cms_api_search`, and shapes come from `cms_api_describe`.

→ `mcp/docs/server/cms-api-describe` · `mcp/docs/server/confirm-and-dry-run`

## Arguments

| Argument | Type | Meaning |
|---|---|---|
| `opId` | string, required | Operation id from `cms_api_search` |
| `path` | object | Path parameters, e.g. `{ "id": 42 }` |
| `query` | object | Query parameters, e.g. `{ "limit": 30, "offset": 0 }` |
| `body` | any | Request body; copy the shape from `cms_api_describe` |
| `dryRun` | boolean | Do not send: return the resolved request, the policy decision and the current target |
| `confirm` | string | Confirm token from a dry run of this exact call |

Pass the body as a real object. A JSON string that parses to an object is unwrapped for you, but relying on that is a habit that breaks the moment a body is legitimately a string.

## Calling a read operation

```json
{ "opId": "AdminPagesController_findAllRoot", "query": { "limit": 20, "offset": 0 } }
```

```json
{ "opId": "AdminPagesController_findAllRoot", "status": 200,
  "truncated": false, "body": { "items": [ … ], "total": 14 } }
```

`truncated` tells you whether the server capped what it handed back. When it is `true`, narrow the query rather than calling again.

Reads are not written to the audit log and never require confirmation.

## Always dry run a mutation first

```json
{ "opId": "AdminMenusController_update", "path": { "id": 12 },
  "body": { "localizeInfos": { "en_US": { "title": "Main" } } }, "dryRun": true }
```

You get the resolved request, the policy decision, and the current state of the target so you can check you are changing what you think you are changing.

→ `mcp/docs/server/confirm-and-dry-run#reading-the-dry-run-target`

## Confirm tokens

Deletes and the permanently gated paths answer with `needsConfirm`, a `target`, and a 24-character token valid for five minutes and bound to the exact arguments. Show the target to the human, get their agreement, then repeat the identical call with `confirm` added.

A gated operation returns the confirmation payload **even when you asked for a dry run** — that branch takes precedence.

## Refusals and what each one means

Every refusal below happens locally. Nothing was sent, and nothing changed.

| Message begins | Meaning | What to do |
|---|---|---|
| `Unknown opId "…". No request was sent.` | The id does not exist in this instance's catalog | Use the `didYouMean` list, or search again |
| `… is a "destructive" operation, but this server runs with --allow=…` | The allow level forbids it | Ask the operator; do not look for another route |
| `Admin #… lacks the "…" permission required by …` | The admin does not hold the permission | Ask for the grant; retrying cannot work |
| `Confirm token is expired, already used, or does not match…` | The token no longer applies | Dry run again for a fresh one |
| `Missing path parameter "id" for … Pass it as path.id.` | A path parameter was not supplied | Add it under `path` |

An error from the instance itself looks different — it carries a `status` and usually a `hint`:

| Status | What it usually means |
|---|---|
| 400 or 422 | The payload shape is wrong. Loosely typed fields mean the example is the contract |
| 403 | A permission the local check could not verify. Ask for the grant |
| 404 | The id may belong to another instance — prefer marker-based operations |
| 5xx | Stop and report the operation id and the request. Do not retry with a modified body |
| 0 | The instance was unreachable at all — a network or base-URL problem |

→ `mcp/docs/server/errors-and-refusals`

## Response size and truncation

Responses over the cap — 24 KB by default — come back with a `_truncated` envelope reporting what was shown and what the total is. Long base64-looking strings are replaced with `[stripped N chars]` so a single file payload cannot eat the whole budget.

Truncation is this server protecting your context, not the API limiting itself. Narrow with the operation's own `limit`, `offset` and filter parameters.

→ `mcp/docs/server/response-shaping`

## What gets written to the audit log

When an audit path is configured, every non-`GET` call appends one line: timestamp, mode, operation id, method, path, a hash of the arguments, the admin id where known, the outcome (`denied`, `needs-confirm` or `sent`) and the resulting status. Arguments are hashed, never stored.

Reads are not recorded.

→ `mcp/docs/server/audit-log`

## Worked example create then delete a template

```json
{ "opId": "AdminTemplatesController_create",
  "body": { "identifier": "scratch-template", "type": "page",
            "localizeInfos": { "en_US": { "title": "Scratch" } } },
  "dryRun": true }
```

Policy says `would be sent`. Repeat without `dryRun` and you get `status: 201` with the created object — read the id from it rather than assuming one.

Now remove it:

```json
{ "opId": "AdminTemplatesController_remove", "path": { "id": 137 }, "dryRun": true }
```

This answers `needsConfirm` with the template as `target` and a token. Confirm that `target.identifier` really is `scratch-template`, tell the human, then:

```json
{ "opId": "AdminTemplatesController_remove", "path": { "id": 137 },
  "confirm": "8f3c1a90b7d24e5f6a0b1c2d" }
```

## Mistakes that cost a retry

- **Inventing an operation id.** It is refused with suggestions, but you have burned a turn. Search first.
- **Re-sending a write because the list did not show it.** Writes are visible by id immediately and in lists a moment later. Re-read by id; never re-write.
- **Sending a flat attribute map.** It returns 201 and stores nothing. Read the entity back after creating it.
- **Changing the body between the dry run and the confirmed call.** The token is bound to the arguments and stops matching.
- **Retrying a 403.** Permissions do not change because you asked twice.
- **Retrying a 5xx with a tweaked body.** If the supported route answered 5xx, report it rather than guessing at variants.
