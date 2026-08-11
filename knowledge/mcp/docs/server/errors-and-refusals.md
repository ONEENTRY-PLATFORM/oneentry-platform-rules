# Errors and refusals

Every error this server can hand you, what caused it, and the one next action for each. Written to be found by pasting the message you got.

The most important distinction: a **refusal** happens locally and nothing was sent; an **error** came back from the instance and something was attempted.

→ `mcp/docs/server/errors-startup` · `mcp/docs/server/allow-levels`

## How to read an error from this server

Errors arrive as JSON with an `error` field and, usually, context:

```json
{ "error": "…", "opId": "…", "status": 403, "hint": "…" }
```

- No `status` at all means the server decided locally. Nothing was sent, nothing changed.
- `status: 0` means the request was attempted and the instance could not be reached.
- Any other `status` came from the instance.
- `hint` is the next action. Read it before doing anything else.

## Refused because of the allow level

```text
DELETE /menus/{id} is a "destructive" operation, but this server runs with --allow=write.
No request was sent. Ask the operator to restart it with --allow=destructive.
```

The server's write policy forbids this class of operation. Nothing was sent — do not check whether it partly happened.

**Do:** report to the human what you wanted to do, why, and which level it needs. **Do not:** look for a different operation that achieves the same thing; it will be classified the same way.

## Refused because of a missing permission

```text
Admin #12 lacks the "menu.delete" permission required by AdminMenusController_remove.
No request was sent — ask for the grant instead of retrying.
```

The authenticated admin does not hold the permission the operation declares. This is final: retrying, changing the body, or trying a neighbouring operation with the same requirement all fail identically.

**Do:** tell the human the exact permission string to grant. `cms_whoami` lists what the admin currently holds.

## Needs a confirm token

```json
{ "needsConfirm": true, "reason": "DELETE /menus/{id} is irreversible.",
  "target": { "id": 12, "identifier": "main-menu" },
  "confirm": "…", "expiresInSeconds": 300 }
```

Not an error. The operation is a delete, or on the permanently gated list, and a human must see the target first.

**Do:** show the target, get agreement in this conversation, then repeat the identical call with `confirm`.

→ `mcp/docs/server/confirm-and-dry-run`

## Confirm token expired used or mismatched

```text
Confirm token is expired, already used, or does not match these arguments.
Re-run with dryRun: true to get a fresh one.
```

Tokens last five minutes, work once, and are bound to the exact arguments. Any change to the operation id, path, query or body invalidates it.

**Do:** dry run again and get a new one. Never try to reuse or reconstruct a token.

## Unknown opId

```json
{ "error": "Unknown opId \"AdminProductsController_getAll\". No request was sent.",
  "didYouMean": ["AdminProductsController_findAll", "…"],
  "hint": "Operation ids come from cms_api_search — do not construct them by hand." }
```

Either the id was invented, or it exists on a different instance than this one.

**Do:** take a suggestion, or search again by entity name. If the whole catalog is empty every id will be unknown — check `cms_whoami`.

## Unknown docId

```json
{ "error": "Unknown docId \"mcp/docs/api/order\".",
  "candidates": ["mcp/docs/api/orders"], "hint": "Use cms_docs_search to get a valid docId." }
```

A near miss on a document name, or a document that does not exist in the loaded corpus.

**Do:** take a candidate. If candidates are empty and everything is unknown, the documentation is degraded — say so rather than concluding it is missing.

## Could not build the request

```text
Missing path parameter "id" for AdminPagesController_update (/pages/{id}). Pass it as path.id.
```

A path parameter was not supplied. A rarer variant says a parameter is not declared in the catalog at all, which means the cached API document may be stale.

**Do:** add the parameter under `path`. `cms_api_describe` lists every parameter with its location.

## Authentication failed

Messages here name the credential problem directly: no credentials configured, a login rejected with a status, a token that is not usable, or a token rejected with no way to re-authenticate.

Accounts flagged as developer accounts are rejected by the Admin API by design; the 401 says so explicitly.

**Do:** call `cms_whoami` — it distinguishes "credentials configured and rejected" from "no credentials reached the server".

→ `mcp/docs/server/authentication`

## The CMS returned an error

| Status | Hint you will see | What to do |
|---|---|---|
| 400, 422 | Validation rejected the payload; loosely typed fields mean the example is the contract | Re-read the entity's document and the schema example, then fix the shape |
| 403 | The admin lacks a permission; retrying will not help | Ask for the grant |
| 404 | The id may belong to another instance — prefer marker-based operations | Resolve the entity by marker, or list it first |
| 5xx | Server-side failure; do not retry blindly | Stop and report the operation id and request |
| 0 | Is the instance reachable at the configured base URL | Check the base URL and that the instance is up |

The single most common cause of a 400 on this API is an attribute map that is one level deep instead of two. That case does **not** always error — it can answer 201 and store nothing.

→ `mcp/docs/server/payload-conventions#the-flat-map-that-silently-does-nothing`

## A truncated response is not an error

`status: 200` with `truncated: true` and a `_truncated` envelope is a successful call whose result was clipped for display. Narrow the query; do not report a failure.

→ `mcp/docs/server/response-shaping`
