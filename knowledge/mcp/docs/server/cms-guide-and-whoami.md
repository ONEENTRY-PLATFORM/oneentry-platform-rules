# The cms guide and cms whoami tools

The two tools that take no arguments and tell you where you are. Call `cms_guide` once at the start of a session; call `cms_whoami` whenever something behaves in a way you cannot explain.

Neither touches the Admin API for its answer beyond the authentication the session already has, so both are safe to call at any allow level.

→ `mcp/docs/server/getting-started` · `mcp/docs/server/errors-startup`

## cms guide

Returns a short markdown briefing, generated from the live state of the server:

- the mode and base URL, and the write policy in one sentence;
- how many operations the catalog holds;
- how many documents and sections the knowledge index holds, and where they came from;
- the loop to follow, and the instruction never to invent a path;
- the API map: the top tags with an operation count each;
- the hard limits — Admin API only, the permanently confirm-gated paths, response capping;
- any catalog warnings.

The API map is the fastest way to see what an instance can do. If a tag you expected is missing, that capability is not exposed on this instance.

## Reading the policy line

```text
Mode: local · Base URL: https://your-instance.example/api/admin · Policy: read-only —
every POST/PUT/PATCH/DELETE is refused locally, without a request being sent
```

Three variants, one per allow level. If it says read-only and you were told the server allows writes, the configuration did not take effect — check `cms_whoami` and the precedence order.

→ `mcp/docs/server/configuration#precedence-flags-then-environment-then-file-then-defaults`

## Knowledge is degraded

When the guide contains this block, the documentation repository could not be reached and only the bundled operating rules are loaded:

```text
> **Knowledge is degraded.** Knowledge repository … is unavailable …
```

Searches will find almost nothing, and every document referenced by a pointer will be missing. Say so to the human rather than concluding that the documentation does not exist.

## cms whoami

Returns the resolved state as JSON:

```json
{ "mode": "local", "baseUrl": "https://your-instance.example/api/admin", "allow": "write",
  "authenticated": true,
  "admin": { "id": 1, "permissionCount": 96, "permissions": ["menu.create", "…"] },
  "catalog": { "operations": 478, "swaggerHash": "…", "builtAt": "…",
               "knownPermissions": 169, "warnings": [] },
  "knowledge": { "source": "github", "repo": "ONEENTRY-PLATFORM/oneentry-platform-rules",
                 "ref": "main", "commit": "a1b2c3d4e5f6", "builtAt": "…",
                 "docs": 53, "sections": 520 },
  "audit": "/var/log/oneentry-mcp-platform-audit.jsonl" }
```

## What to check in it

- **`allow`** — before planning any write. Planning a delete against a read-only server wastes the whole turn.
- **`admin.permissions`** — the exact list the local pre-check uses. Compare it against the `permission` field of the operation you intend to call, and ask for the grant *before* you build the payload.
- **`catalog.operations`** — zero means nothing can be searched or called.
- **`catalog.warnings`** — surfaces mismatches between the instance and the bundled permission map, operations that cannot be called, and a catalog served from a stale cache.
- **`knowledge.source`** — `github` or `cache` is normal, `local` means a local clone is being read, `seed` means degraded.
- **`audit`** — `null` means nothing is being recorded. In remote mode that cannot happen; the server refuses to start without an audit path.

## When you are not authenticated

`admin` is `null` and a `hint` explains which of the two cases it is:

```text
Credentials are configured but authentication failed — check ONEENTRY_CMS_LOGIN /
ONEENTRY_CMS_PASSWORD and that the instance is reachable.
```

```text
No credentials configured. Set ONEENTRY_CMS_LOGIN + ONEENTRY_CMS_PASSWORD
(or ONEENTRY_CMS_TOKEN).
```

Note that an empty `permissions` list is not the same as "no permissions". It means the server could not read the admin's rights, and in that case the local permission pre-check is skipped entirely — refusals will then come from the instance as 403 instead.

→ `mcp/docs/server/authentication`

## Using them together

At the start of a session: `cms_guide` for the policy and the map, then `cms_whoami` if anything in it is surprising.

When a call fails in a way you did not predict: `cms_whoami` first. Most unexplained failures are one of four things it reports directly — the wrong allow level, a missing permission, an empty catalog, or degraded documentation.
