# Getting started with the OneEntry Platform MCP server

How to install the server, point it at an instance, and make your first three tool calls. Read this once; after that `mcp/operating-rules` and the per-tool documents carry the detail.

The server exposes the **Admin API only**. The Content and Developer APIs are deliberately not reachable through it.

→ `mcp/docs/server/configuration` · `mcp/operating-rules`

## What this server is

One MCP server that lets an agent operate a OneEntry instance's Admin API. It has two halves:

- **Knowledge** — this documentation, fetched from a public repository at runtime and searchable through `cms_docs_search` and `cms_docs_read`.
- **Actuation** — every Admin API operation of the instance you point at, built from that instance's own OpenAPI document, with parameter and body schemas, the permission each one requires, and a risk classification. All of it reachable through one generic invoke tool rather than one tool per endpoint, plus two tools for the one thing a JSON body cannot carry: uploading a file.

Nothing is frozen into the package. The catalog always describes the API that is actually running, and the documentation updates without an npm release.

## Install

```bash
npm i -g @oneentry/mcp-platform-server
```

Or skip the install and let your MCP client run it with `npx`, which is what the configuration below does.

Node 20 or newer is required.

## Local mode

Local mode is the default: the server speaks stdio, and the agent and the instance are reachable from the same machine. Add this to `.mcp.json` in the repository you work from:

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

No checkout of the platform is needed. On first run the documentation is downloaded once and cached; later runs reuse the cache and only ask whether the commit moved, at most once an hour.

The base URL must contain `/api/admin` — the server refuses to start otherwise, because exposing only the Admin API is a guarantee rather than a convention.

## Read only until you say otherwise

The server starts **read only**. Every `POST`, `PUT`, `PATCH` and `DELETE` is refused locally, before authentication, without a request leaving the process.

Raising it is the operator's decision, not the agent's:

```json
"env": { "ONEENTRY_MCP_ALLOW": "write" }
```

`write` permits `POST`, `PUT` and `PATCH`. `destructive` additionally permits `DELETE`, which still requires a confirm token obtained from a dry run.

→ `mcp/docs/server/allow-levels`

## Your first three calls

1. **`cms_whoami`** — confirms the mode, the base URL, the allow level, which admin you are authenticated as and which permissions it holds, plus the state of the catalog and the documentation. If anything is misconfigured, this is where it shows.
2. **`cms_guide`** — the current policy, the API map by tag, and the order in which to use the other tools.
3. **`cms_docs_read { "docId": "mcp/operating-rules" }`** — the payload rules. Read this before your first write, not after your first 400.

## Finding and calling an operation

```text
cms_api_search  { "query": "menu position", "mutating": true }
cms_api_describe { "opId": "AdminMenusController_updatePosition" }
cms_api_call    { "opId": "AdminMenusController_updatePosition",
                  "path": { "id": 12 }, "body": { }, "dryRun": true }
```

`cms_api_search` is the only authority on which endpoints exist. Operation ids come from it and are never constructed by hand — a hand-made id is refused with a "did you mean" list rather than sent.

A mutating call should always be dry-run first. The dry run returns the resolved request, the policy decision, and the **current state of the target**, so a human can see what is about to change.

→ `mcp/docs/server/cms-api-call`

## If something does not work

- `cms_whoami` reports `authenticated: false` — the credentials are wrong, or the instance is unreachable. Accounts flagged as developer accounts are rejected by the Admin API by design.
- `cms_guide` says the catalog is empty — the instance did not serve its OpenAPI document. Nothing can be searched or called until it is reachable.
- `cms_guide` says the knowledge is degraded — the documentation repository could not be reached and only the bundled rules are available.
- A call is refused with "No request was sent" — that is the allow level or a missing permission, decided locally. Nothing changed.

→ `mcp/docs/server/errors-startup` · `mcp/docs/server/errors-and-refusals`

## Where to go next

- `mcp/docs/server/doc-map` — the whole corpus, with when to read each document
- `mcp/docs/server/configuration` — every flag, environment variable and config-file key
- `mcp/docs/server/remote-mode` — hosting one server for many agents
- `mcp/docs/api/baseline-data` — what already exists on the instance before you create anything
