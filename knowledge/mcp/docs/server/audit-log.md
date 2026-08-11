# The audit log

What the server records when an agent changes something, what it deliberately does not record, and how to read the file afterwards.

Optional in local mode, mandatory in remote mode. Enable it with `--audit <path>` or `ONEENTRY_MCP_AUDIT_PATH`.

→ `mcp/docs/server/remote-mode` · `mcp/docs/server/allow-levels`

## What is recorded

One JSON object per line, appended. Every non-`GET` call produces a line, whether it was sent, refused or stopped for confirmation.

```json
{"at":"2026-08-10T09:14:22.106Z","mode":"remote","adminId":1,
 "opId":"AdminMenusController_remove","method":"DELETE","path":"/menus/{id}",
 "argsHash":"9f2b41c7a8de0315","outcome":"sent","status":200}
```

| Field | Meaning |
|---|---|
| `at` | Timestamp, ISO 8601 |
| `mode` | `local` or `remote` |
| `adminId` | The authenticated admin, when the operation declares a permission and the identity resolved |
| `opId` | The operation id |
| `method` | The HTTP method, uppercase |
| `path` | The operation path, without the API prefix |
| `argsHash` | A hash of the arguments — first 16 hex characters |
| `outcome` | `denied`, `needs-confirm` or `sent` |
| `status` | The HTTP status, present only for `sent` |

## Reads are not recorded

`GET` calls are not written. The log answers "what changed and who changed it", not "what was looked at" — and recording reads would bury the mutations in noise.

A refusal is recorded even though nothing was sent, because "the agent tried to delete this and was stopped" is exactly the sort of thing an operator wants to see.

## Arguments are hashed not stored

`argsHash` is a hash of the operation id together with the normalized arguments. That is enough to prove two calls were identical, or that a confirmed call matched the dry run that preceded it, without putting request bodies — which may contain customer content — into a log file.

It also means the log cannot answer "what exactly did it set the title to". If you need that, read the entity's own history in the admin panel.

## Reading the log

```bash
# every mutation an admin made today
grep '"adminId":1' audit.jsonl | grep '2026-08-10'

# everything that was refused
grep '"outcome":"denied"' audit.jsonl

# deletes that actually went through
grep '"method":"DELETE"' audit.jsonl | grep '"outcome":"sent"'
```

A `needs-confirm` line followed by a `sent` line with the **same** `argsHash` is the signature of a properly confirmed destructive change. A `needs-confirm` with no matching `sent` is an approval that was requested and never used — usually a human saying no, which is the system working.

## Operational notes

- The file is opened in append mode and never rotated by the server. Rotate it externally; a rotation that moves the file mid-run is handled on the next write.
- A failed write never fails the call. The server prints a line to standard error and carries on — losing an audit entry is better than losing a change the human asked for, and the noise makes the misconfiguration obvious.
- The directory must exist and be writable before the server starts.
- Nothing in the file is a credential: no tokens, no passwords, no confirm tokens.

## Why it is mandatory remotely

One shared process acting for many agents, each with its own identity, is exactly the situation where "who did this" stops being answerable from memory. The server refuses to start in remote mode without an audit path rather than letting that configuration exist.
