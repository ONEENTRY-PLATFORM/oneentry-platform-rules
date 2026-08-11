# Startup and connectivity problems

Failures that happen before any tool call succeeds, or that make every tool call useless. Most are configuration, and most announce themselves clearly.

Startup errors print to standard error as `oneentry-mcp-platform: <message>` and exit with a non-zero code. If your MCP client shows the server as failed with no output, look in its log for that prefix.

→ `mcp/docs/server/configuration` · `mcp/docs/server/cms-guide-and-whoami`

## The base URL is rejected

```text
baseUrl must point at the Admin API (…/api/admin), got "https://your-instance.example".
This server intentionally exposes the Admin API only.
```

Append `/api/admin` to the URL. This is enforced rather than assumed, so no configuration can make the server reach the Content or Developer APIs.

## The password was passed as a flag

```text
Refusing --password: pass ONEENTRY_CMS_PASSWORD via the environment instead.
```

Command-line arguments are visible in the machine's process list. Move it to the environment.

## Remote mode has no audit path

```text
Remote mode requires --audit <path>: every mutation must be auditable.
```

Add `--audit` with a writable path. There is no way to disable it in remote mode.

## A setting is malformed

```text
Expected a number, got "eight thousand"
```

Numeric settings — port, timeouts, the response cap, the knowledge refresh interval — must be numbers. The allow level must be exactly `read`, `write` or `destructive`. The knowledge repository must look like `owner/name`.

A malformed `oneentry-mcp-platform.json` is also a startup error; a missing one is not.

## The local knowledge path cannot be read

```text
Local knowledge path "../rules" could not be read (…). Fix the path, or unset it to read
the knowledge repository from GitHub.
```

Deliberately fatal. Unlike the repository being unreachable, a bad path is a configuration mistake, and silently falling back to the bundled rules would look like an empty corpus.

## The catalog is empty

Not a startup error — the server runs, but nothing can be searched or called:

```text
The operation catalog is EMPTY: … did not serve its API document (…). No API operation
can be searched or called until the instance is reachable.
```

Check, in order:

1. Is the base URL right, including `/api/admin`?
2. Is the instance actually up? Try a request to it independently.
3. Does it expose the Admin API on that host and port?
4. Does reading its API document require credentials the server does not have?

In remote mode a server that started without credentials can fill its catalog from the first session that authenticates, so this may resolve itself once a client connects.

→ `mcp/docs/server/api-catalog`

## The catalog came from the cache

```text
The instance at … was unreachable; this catalog came from the on-disk cache and may not
match the API that is actually running.
```

The server is usable, but the operation list may be out of date. An "unknown operation" against a cached catalog means "possibly stale", not "does not exist". Restart once the instance is reachable.

## Knowledge is degraded

```text
> **Knowledge is degraded.** Knowledge repository … is unavailable … Only the bundled
operating rules are available — module reference docs will not be found.
```

The documentation repository could not be reached and there is no cache. Searches will find almost nothing.

Common causes: no outbound network; an API rate limit, which a GitHub token raises; a repository or ref that does not exist, which usually means a typo in `--knowledge-repo` or `--knowledge-ref`; or `--offline` with an empty cache.

Say so to the human. Do not conclude from an empty search that the documentation has no answer.

→ `mcp/docs/server/knowledge-subsystem`

## Everything answers 401

Credentials are configured and rejected. `cms_whoami` distinguishes this from having no credentials at all.

Check the login and password, that they belong to this instance, and that the account is an ordinary admin — the Admin API rejects accounts flagged as developer accounts by design.

## Requests time out

The default per-request timeout is 30 seconds. A consistent timeout usually means the instance is unreachable rather than slow — the network error hint distinguishes them.

Raising `--request-timeout` is worth doing only for an operation known to be slow, such as a large import.

## The remote server answers 400 or 404 on every request

```text
{ "error": "Unknown or missing MCP session. Send an initialize request first." }
```

The client is not completing the MCP handshake, or a proxy is dropping the `mcp-session-id` header. Both are client-side; the server has nothing to configure here.

→ `mcp/docs/server/remote-mode#running-behind-a-reverse-proxy`
