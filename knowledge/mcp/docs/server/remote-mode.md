# Remote mode

Running one server over HTTP for many agents against one instance, with a separate identity per session and a mandatory audit log.

Local mode is the default and speaks stdio. Adding `--http` switches to remote mode; there is no separate mode flag.

→ `mcp/docs/server/configuration` · `mcp/docs/server/authentication`

## Starting the server over HTTP

```bash
oneentry-mcp-platform --http \
  --port 8931 \
  --host 127.0.0.1 \
  --audit /var/log/oneentry-mcp-platform-audit.jsonl \
  --base-url https://your-instance.example/api/admin \
  --allowed-origins https://agent.example
```

It binds to loopback by default and prints one line to standard error on startup, naming the address, the allow level and the base URL.

The transport is Streamable HTTP on `/mcp`: `POST` to initialise and to send requests, `GET` for the event stream, `DELETE` to close a session.

## The audit log is mandatory here

Remote mode refuses to start without `--audit`:

```text
Remote mode requires --audit <path>: every mutation must be auditable.
```

This is not adjustable. A shared server acting for many agents must leave a record of every mutation, or nobody can answer the question "who changed this".

→ `mcp/docs/server/audit-log`

## Identity is per session and comes from headers

The process holds no credentials. Each client supplies its own when it initialises:

| Header | Meaning |
|---|---|
| `x-cms-token` | A pre-issued Admin API access token. Preferred |
| `x-cms-login` + `x-cms-password` | Credentials to log in with. Both required |

`x-cms-token` wins if both are present.

## Why credentials are never tool arguments

Anything an agent can pass as a tool argument, a prompt injection can also cause it to pass. Connection headers are set by the client when the transport is established, before any model output exists, so an instruction hidden in a page title or a product description cannot reach them.

The same reasoning is why the password cannot be a command-line flag: argv is readable by other processes on the machine.

## What is isolated between sessions

Per session, never shared:

- the admin identity and its permission list;
- the access and refresh tokens;
- **confirm tokens** — one issued to a session cannot be used by another, so no agent can spend another agent's approval.

Shared across the process: the operation catalog and the documentation index. Both are read-only and identical for every session, so there is nothing to leak in either.

## The Origin allowlist

Requests carrying an `Origin` header are checked against `--allowed-origins`:

```text
{ "error": "Origin \"https://elsewhere.example\" is not allowed. Start the server with --allowed-origins to permit it." }
```

Requests with **no** `Origin` are always accepted — that is the ordinary case for a non-browser MCP client. The allowlist exists to stop a browser page on an unrelated site from driving the server, not to authenticate clients; that is what the session headers are for.

## The health endpoint

```bash
curl http://127.0.0.1:8931/health
```

```json
{ "ok": true, "mode": "remote", "sessions": 3 }
```

Liveness and the current session count. Use it for process supervision; it says nothing about whether the instance behind the server is reachable — `cms_whoami` is what answers that.

## Running behind a reverse proxy

- Terminate TLS at the proxy and keep the server on loopback. It has no TLS of its own.
- Forward `x-cms-token`, `x-cms-login`, `x-cms-password` and `mcp-session-id` unchanged. Stripping any of them breaks either identity or session continuity.
- Do not buffer the `GET /mcp` response — it is an event stream.
- Requests are capped at 4 MB.

## Failure modes and what they look like to the client

| Situation | Response |
|---|---|
| A request on an unknown or missing session, other than initialise | `400` `Unknown or missing MCP session. Send an initialize request first.` |
| An event-stream or close request for a session that no longer exists | `404` `Unknown MCP session.` |
| A disallowed `Origin` | `403`, with the message above |
| Credentials rejected | The session initialises, and `cms_whoami` reports `authenticated: false` |

A server that started before its instance was reachable begins with an empty catalog and fills it from the first session that authenticates successfully, so an early "no operations" state can resolve itself without a restart.

→ `mcp/docs/server/api-catalog`
