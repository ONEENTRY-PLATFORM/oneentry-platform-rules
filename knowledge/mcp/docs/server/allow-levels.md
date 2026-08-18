# Allow levels and what this server refuses

The write policy: three levels, how the level of an operation is decided, and why a refusal means nothing was sent. Read this when a call comes back refused, or before asking an operator to loosen a server.

The level is the operator's decision, set when the server starts. An agent cannot change it, and no argument to any tool can escalate past it.

→ `mcp/docs/server/confirm-and-dry-run` · `mcp/docs/server/errors-and-refusals`

## The three levels

| Level | GET | POST PUT PATCH | DELETE |
|---|---|---|---|
| `read` (default) | yes | refused | refused |
| `write` | yes | yes | refused |
| `destructive` | yes | yes | yes, confirm-gated |

Set with `--allow` or `ONEENTRY_MCP_ALLOW`. A server with no configuration at all is read-only, so an agent connected to a fresh server cannot change anything by accident.

## What each level permits

- **`read`** — discovery and inspection only. Every mutation is refused locally. This is the right level for analysis, audits, and any session where the agent is figuring out what exists.
- **`write`** — creation and update. Deletes are still refused, and the permanently gated paths still need a confirm token. This is the normal working level for content work.
- **`destructive`** — everything, with deletes gated behind a confirm token that a dry run issues. Use it for a specific task, not as a default.

## Risk is derived from the HTTP method

Each operation is classified when the catalog is built, from its HTTP method alone:

- `GET` → `read`
- `POST`, `PUT`, `PATCH` → `write`
- `DELETE` → `destructive`

The classification is not read from the API document and cannot be influenced by anything you pass. An operation that deletes something through a `POST` is classified `write`, so read what an operation does in `cms_api_describe` rather than trusting the label alone.

`cms_api_describe` reports the `risk` of an operation, and `cms_api_search` can filter by `mutating: true` or `false`.

## Uploading a file is a write

The two upload tools — `cms_upload_file` and `cms_import_file_from_url` — sit behind the same gate as `cms_api_call`. At `read` both are refused before anything is read from disk or fetched from the network, and the refusal is recorded like any other.

They also carry bounds the level does not express, because their **source** is a tool argument: a root directory the local one cannot leave, a size cap, a refusal of any address inside the network the server runs in, and in remote mode an operator allowlist before the URL import works at all.

→ `mcp/docs/server/cms-upload-file#what-both-tools-refuse`

## A level refusal happens before authentication

This ordering is deliberate and worth relying on. When the level forbids an operation, the server refuses **before** it authenticates and before it builds a request. No HTTP call leaves the process:

```text
DELETE /menus/{id} is a "destructive" operation, but this server runs with
--allow=write. No request was sent. Ask the operator to restart it with
--allow=destructive.
```

"No request was sent" is literal. Nothing on the instance changed, nothing was read, and no credentials were used. There is no reason to check whether the operation half-happened.

## Paths that are always confirm gated

Regardless of the allow level, **mutations** under these paths always require a confirm token:

`immutable-settings` · `admins` · `backups` · `modules` · `payments/webhook` · `settings-general` · `system/captcha-keys` · `auth/logout/all-users`

`GET` on those paths is unaffected — reading them is ordinary.

These are the areas where a mistake is either irreversible or invisible: platform limits, the accounts that can undo your work, backups, the module wiring the whole admin panel hangs off, payment callbacks, and global settings. `cms_api_describe` reports `alwaysConfirm: true` for every operation in the group.

The gate is not a formality. Show the human the dry run's `target`, say what you intend to change, and wait for them to agree in this conversation.

## The local permission pre check

Separately from the level, most operations declare the admin permission they require. When the authenticated admin does not hold it, the server refuses locally:

```text
Admin #12 lacks the "menu.delete" permission required by AdminMenusController_remove.
No request was sent — ask for the grant instead of retrying.
```

Two details matter:

- The check only runs when the server was able to read the admin's permissions. If it could not, the check is skipped rather than treated as a denial — so a call may still come back `403` from the instance.
- A permission refusal is final. Retrying, changing the body, or trying a neighbouring operation that needs the same permission will not help. Ask for the grant.

→ `mcp/docs/api/admins-and-permissions`

## Choosing a level when you run the server

- Give an agent the lowest level the task needs, and raise it for the task rather than for the session.
- `destructive` plus an unattended agent is the combination to avoid — the confirm gate assumes a human is present to look at the target.
- In remote mode an audit log is mandatory, so every mutation is recorded regardless of level.

→ `mcp/docs/server/audit-log`

## Asking the operator to raise the level

When you hit a level refusal, do not work around it. Report it in one message:

- the operation id and what it does, in plain words;
- the level the server runs with and the level the operation needs;
- what you intend to change, and what the dry run showed if you could run one.

Then stop. Restarting the server with a different level is the operator's action, not a step you can take.
