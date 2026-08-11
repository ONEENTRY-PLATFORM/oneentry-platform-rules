# Dry runs and confirm tokens

How to preview a mutation before it happens, and how the confirmation gate on deletes and protected paths works. Read this before your first write of a session.

The short version: pass `dryRun: true`, read the `target` it returns, show it to the human, then repeat the call with the `confirm` token if one was issued.

→ `mcp/docs/server/cms-api-call` · `mcp/docs/server/allow-levels`

## Always dry run a mutation first

```json
{ "opId": "AdminPagesController_update", "path": { "id": 42 },
  "body": { "localizeInfos": { "en_US": { "title": "About us" } } },
  "dryRun": true }
```

The response tells you three things:

```json
{ "dryRun": true,
  "request": { "method": "PUT", "url": "https://your-instance.example/api/admin/pages/42",
               "body": "{\"localizeInfos\":…" },
  "policy": { "allow": "write", "risk": "write", "decision": "would be sent" },
  "target": { "id": 42, "localizeInfos": { "en_US": { "title": "About" } } } }
```

`request` is the call as it would actually go out, after path substitution and query building. `policy` is the decision the server reached. `target` is the **current state of what you are about to change**.

The body preview is cut at 2000 characters. That is a display limit only — the real call sends the whole body.

## Reading the dry run target

`target` is fetched with the sibling `GET` on the same path, so it is the live object, not something remembered. It is the most valuable part of the response: it is how you catch an id that points at a different entity than you assumed, a field you are about to overwrite, or a delete aimed at the wrong row.

Large targets are summarised rather than dumped:

```json
{ "_summary": { "kind": "list", "count": 214, "ids": [1, 2, 3, "…"],
                "hint": "Summarised: read the target in full with a separate GET if you need it." } }
```

`kind` is `list`, `object` or `value`. For an object too large to show, you get its keys and its identifying field. If the summary is not enough to decide, read the entity in full with a separate call before proceeding.

When the operation has no sibling `GET` — a create, for instance — there is no target to show, and its absence is not an error.

## When a confirm token is required

Two cases, both independent of the allow level:

- **Deletes**, at `--allow=destructive`. The reason reads `DELETE /menus/{id} is irreversible.`
- **Any mutation on a permanently gated path** — `immutable-settings`, `admins`, `backups`, `modules`, `payments/webhook`, `settings-general`, `system/captcha-keys`, `auth/logout/all-users`. The reason reads `/admins is on the permanently confirm-gated list; it always needs an explicit confirm token.`

Everything else at a sufficient allow level runs on the first call.

## What a needs confirm response looks like

```json
{ "needsConfirm": true,
  "reason": "DELETE /menus/{id} is irreversible.",
  "request": { "method": "DELETE", "url": "…/api/admin/menus/12" },
  "target": { "id": 12, "identifier": "main-menu" },
  "confirm": "8f3c1a90b7d24e5f6a0b1c2d",
  "expiresInSeconds": 300,
  "next": "Show the target to the human. Then repeat this exact call with the confirm token added." }
```

Note the ordering: for a gated operation this response comes back **even when you asked for `dryRun: true`**. The confirmation branch takes precedence, and the payload contains everything a dry run would have told you anyway.

## Confirm tokens

- 24 hexadecimal characters.
- Valid for **five minutes**, single use. Using it consumes it, whether or not the call then succeeds.
- Bound to a hash of the operation id **and the exact arguments**. Change the id, a path parameter, a query value or one byte of the body and the token no longer matches.
- Held in memory, per session. In remote mode a token issued to one session is meaningless to another.

To use one, repeat the identical call with `confirm` added:

```json
{ "opId": "AdminMenusController_remove", "path": { "id": 12 },
  "confirm": "8f3c1a90b7d24e5f6a0b1c2d" }
```

If the token is rejected:

```text
Confirm token is expired, already used, or does not match these arguments.
Re-run with dryRun: true to get a fresh one.
```

That is the whole remedy — request a new one. Do not attempt to reconstruct or reuse a token.

## Show the target to the human

The gate exists so a person can see what is about to change. Getting a token and immediately spending it in the next tool call defeats it entirely.

The expected shape of a gated turn:

1. Call with `dryRun: true`; receive `needsConfirm`, the target and the token.
2. Tell the human, in your own words, what will change — quoting the identifying fields from `target`.
3. Wait for them to agree **in this conversation**.
4. Repeat the call with `confirm`.

If they do not answer within five minutes, the token expires and you start again. That is the intended cost.

## Mistakes that waste a token

- Changing anything between the dry run and the confirmed call. Even reordering keys in the body changes the hash.
- Retrying a failed confirmed call with the same token — it was consumed on the first attempt.
- Assuming a dry run on a gated path is "just a preview". It mints a real token; treat the response as the confirmation prompt it is.
- Running a dry run for every operation in a batch and then spending all the tokens at once. Confirm one change at a time, with the human's answer between them.
