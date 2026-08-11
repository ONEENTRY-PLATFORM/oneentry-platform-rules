# Configuring the OneEntry Platform MCP server

Every setting the server accepts, where it can come from, and which combinations are valid. Read this when a server is behaving differently from what you expected, or when you are writing an MCP client configuration for the first time.

`cms_whoami` reports what the server actually resolved. When this document and that output disagree, believe the output.

→ `mcp/docs/server/getting-started` · `mcp/docs/server/remote-mode`

## Precedence flags then environment then file then defaults

Four sources, highest first:

1. Command-line flags
2. Environment variables
3. `oneentry-mcp-platform.json` in the current working directory
4. Built-in defaults

A setting present in a higher source wins outright; there is no merging of individual values. Two exceptions exist by design: the admin password and the GitHub token are read **only** from the environment, never from a flag or the file.

## Command line flags

| Flag | Meaning | Default |
|---|---|---|
| `--base-url <url>` | Admin API base URL, must contain `/api/admin` | `http://localhost:3003/api/admin` |
| `--allow <level>` | `read`, `write` or `destructive` | `read` |
| `--login <login>` | Admin login; the password comes from the environment | none |
| `--audit <path>` | JSONL audit log; required in remote mode | none |
| `--http` | Serve Streamable HTTP instead of stdio | off |
| `--port <n>` | HTTP port | `8931` |
| `--host <addr>` | HTTP bind address | `127.0.0.1` |
| `--allowed-origins a,b` | Comma-separated Origin allowlist | empty |
| `--request-timeout <ms>` | Per-request timeout to the Admin API | `30000` |
| `--max-response-bytes <n>` | Response cap handed to the model | `24576` |
| `--knowledge-repo <owner/name>` | Documentation repository | `ONEENTRY-PLATFORM/oneentry-platform-rules` |
| `--knowledge-ref <ref>` | Branch, tag or commit | `main` |
| `--knowledge-path <dir>` | Read a local clone instead of the network | none |
| `--knowledge-ttl <ms>` | How long a cached commit stays fresh | `3600000` |
| `--cache-dir <path>` | Where the documentation and the OpenAPI snapshot are cached | `${XDG_CACHE_HOME:-~/.cache}/oneentry-mcp-platform` |
| `--offline` | Never touch the network: cache and bundled rules only | off |
| `--help`, `-h` | Print usage and exit | — |

`--password` is **refused** with an explicit error. Command-line arguments are visible in the process list of the machine.

## Environment variables

| Variable | Equivalent flag |
|---|---|
| `ONEENTRY_CMS_BASE_URL` | `--base-url` |
| `ONEENTRY_CMS_LOGIN` | `--login` |
| `ONEENTRY_CMS_PASSWORD` | *(environment only)* |
| `ONEENTRY_CMS_TOKEN` | *(environment only)* — a pre-issued access token instead of login and password |
| `ONEENTRY_MCP_ALLOW` | `--allow` |
| `ONEENTRY_MCP_AUDIT_PATH` | `--audit` |
| `ONEENTRY_MCP_PORT`, `ONEENTRY_MCP_HOST` | `--port`, `--host` |
| `ONEENTRY_MCP_ALLOWED_ORIGINS` | `--allowed-origins` |
| `ONEENTRY_MCP_REQUEST_TIMEOUT_MS` | `--request-timeout` |
| `ONEENTRY_MCP_MAX_RESPONSE_BYTES` | `--max-response-bytes` |
| `ONEENTRY_MCP_KNOWLEDGE_REPO`, `_REF`, `_PATH`, `_TTL_MS` | the matching `--knowledge-*` flags |
| `ONEENTRY_MCP_OFFLINE` | `--offline` |
| `ONEENTRY_MCP_CACHE_DIR` | `--cache-dir` |
| `ONEENTRY_GITHUB_TOKEN` | *(environment only)* — raises the API rate limit; the documentation repository is public, so it is optional |

Boolean variables accept `1`, `true`, `yes` or `on`, case-insensitively.

## The oneentry-mcp-platform json file

A flat JSON object read from the current working directory. A missing file is fine; a malformed one is a startup error.

```json
{
  "baseUrl": "https://your-instance.example/api/admin",
  "allow": "write",
  "login": "automation",
  "maxResponseBytes": 32768,
  "knowledge": { "ref": "v1.2.0" }
}
```

Keys mirror the flags in camelCase: `baseUrl`, `allow`, `login`, `password`, `token`, `auditPath`, `cacheDir`, `requestTimeoutMs`, `maxResponseBytes`, `port`, `host`, `allowedOrigins`, and a nested `knowledge` object with `repo`, `ref`, `path` and `refreshTtlMs`.

There is no `.env` support. A `.env` file next to the server is not read.

## The base URL must point at the Admin API

The URL must contain `/api/admin`, and a trailing slash is stripped. Anything else is a startup error:

```text
baseUrl must point at the Admin API (…/api/admin), got "https://your-instance.example".
This server intentionally exposes the Admin API only.
```

This is a guarantee rather than a convention: the Content and Developer APIs cannot be reached through this server at any allow level.

## Never pass a password as a flag

Passing `--password` fails immediately:

```text
Refusing --password: pass ONEENTRY_CMS_PASSWORD via the environment instead.
```

Use `ONEENTRY_CMS_PASSWORD`, or a pre-issued token in `ONEENTRY_CMS_TOKEN`. In remote mode neither is needed on the process at all — each session brings its own identity in connection headers.

## Minimal local configuration

```json
{
  "mcpServers": {
    "oneentry-mcp-platform": {
      "command": "npx",
      "args": ["-y", "@oneentry/mcp-platform-server"],
      "env": {
        "ONEENTRY_CMS_BASE_URL": "https://your-instance.example/api/admin",
        "ONEENTRY_CMS_LOGIN": "your-admin-login",
        "ONEENTRY_CMS_PASSWORD": "your-admin-password"
      }
    }
  }
}
```

Add `"ONEENTRY_MCP_ALLOW": "write"` when the agent is meant to change things, and an audit path if you want a record of what it changed.

## Minimal remote configuration

```bash
oneentry-mcp-platform --http \
  --port 8931 \
  --audit /var/log/oneentry-mcp-platform-audit.jsonl \
  --base-url https://your-instance.example/api/admin \
  --allowed-origins https://agent.example
```

An audit path is **mandatory** in remote mode; the server refuses to start without one. Credentials are not configured on the process — clients supply them per session.

## Checking what the server actually resolved

`cms_whoami` reports the mode, base URL, allow level, the authenticated admin and its permissions, the catalog state with any warnings, the documentation source with its repository, ref and commit, and the audit path.

`cms_guide` reports the same policy in one line, plus the API map. If a setting you passed does not appear in either, it did not take effect — check the precedence order above and whether the variable name is exact.

→ `mcp/docs/server/cms-guide-and-whoami` · `mcp/docs/server/errors-startup`
