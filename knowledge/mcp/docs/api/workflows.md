# Workflows and the flow engine inside an instance

An instance carries a flow engine: a configuration of wired nodes that runs inside it, on a timer or on a platform event, and that can call the Admin API on its own. Read this when a requirement is shaped like *when*, *every*, *after* or *older than* — a scheduled publish, an expiry, a reminder, a retention purge — because no operation is named after any of those, and the engine is how they get built.

This document covers the configuration as a whole: how it is stored, how it is saved, which nodes an instance has, and how to run one without waiting for its trigger. The configuration of the individual platform nodes is next door.

→ `mcp/docs/api/workflow-nodes` · `mcp/docs/api/events` · `mcp/docs/server/confirm-and-dry-run`

## What a flow is and where it is stored

One configuration per instance, held as a flat array of nodes. A node carries `id`, `type`, `z` (the id of the tab it belongs to), `wires` (its outgoing connections, one list per output) and whatever configuration its own type reads. A node of type `tab` is a flow's container.

The flow format and the general-purpose node types are Node-RED's: each node set reports the `module` it came from, and most report `node-red`. So `inject`, `function`, `switch`, `change`, `template`, `split` and `delay` behave as that project documents them, and a flow built there transfers here.

```json
{ "flows": [
  { "id": "nightly", "type": "tab", "label": "nightly publish" },
  { "id": "t1", "type": "interval-node", "z": "nightly", "interval": 60,
    "wires": [ [ "b1" ] ] },
  { "id": "b1", "type": "admin-api", "z": "nightly", "method": "get",
    "path": "/pages", "params": "{\"langCode\":\"en_US\"}",
    "wires": [ [] ] } ] }
```

## A save replaces the whole configuration

`WorkflowsController_saveFlows` — `POST /workflows/flows` — takes `{ "flows": [...] }` and answers `201` with a `rev`. It is a **replace**, not a merge: nodes you leave out are gone, and whatever they were running stops. So read the current configuration with `WorkflowsController_getFlows` first, add to the array that came back, and send the whole of it.

`rev` is a hash of the stored configuration. The same array saved twice answers the same `rev`, and an empty configuration always answers the same one, so `rev` identifies the content rather than the save. Compare it to notice that someone else has changed the flows; never parse it.

This server gates every `/workflows` write behind confirmation at any allow level, so a save or a run goes through twice: once for the token, once with it.

→ `mcp/docs/server/confirm-and-dry-run`

## The operations and the permissions they need

| Operation id | Call | Permission |
|---|---|---|
| `WorkflowsController_getFlows` | `GET /workflows/flows` | `workflows.get` |
| `WorkflowsController_saveFlows` | `POST /workflows/flows` | `workflows.update` |
| `WorkflowsController_getNodes` | `GET /workflows/nodes` | `workflows.get` |
| `WorkflowsController_getCmsEventsOutputExample` | `GET /workflows/nodes/cms-events/output-example` | `workflows.get` |
| `WorkflowsController_getSwaggerDocs` | `GET /workflows/nodes/admin-api/swagger.json` | `workflows.get` |
| `WorkflowsController_getNodeCredentials` | `GET /workflows/credentials/{type}/{id}` | `workflows.get` |
| `WorkflowsController_executeToNode` | `POST /workflows/execute-to/{nodeId}` | `workflows.update` |

The credentials read answers text values in the clear and replaces a secret with `has_<field>: true`, so it tells you whether a credential is set and never what it is.

## Which node types your instance has

`GET /workflows/nodes` answers the installed node **sets**. Each set carries `id`, `name`, `module`, `version`, `enabled` and a `types` list — the node types it provides, which is what goes in a node's `type`. Read it before promising that a node kind exists: sets differ between instances.

The platform's own sets report `module: "workflows-internal"` and provide `admin-api`, `cms-api`, `cms-events`, `cms-create-order`, `interval-node`, `notification`, `telegram` and `whatsapp`. Everything else in the list comes from the engine itself or from an installed contribution.

## Run a flow without waiting for its timer

`POST /workflows/execute-to/{nodeId}` runs the chain up to one node and answers `201` with `{ "success": true, "output": <the node's outgoing message> }`. An optional `payload` in the body becomes the starting message. It is how a node's configuration gets tested without saving a trigger and waiting for it.

Three refusals, all `400` with `success: false`:

- `Node <id> not found in flows` — the node has to be in the saved configuration first. Save, then run.
- `Execution timeout (30s)` — the chain did not reach that node within thirty seconds. A node whose only input comes from `interval-node` always answers this, because the executor does not fire the timer; drive it from an `inject` node while testing.
- The node's own error text, for anything the node itself refused.

## Build a schedule no endpoint offers

Searching the catalogue for `schedule`, `publish`, `expire`, `retention` or `purge` finds nothing, and a page's create body carries `isVisible` and no dates. That is not the answer to "can it publish on a date". These are the primitives, and each requirement below is a flow of three or four nodes:

| Requirement | Built from |
|---|---|
| publish or expire at a time | `interval-node` → `function` deciding which records are due → `admin-api` writing `isVisible` |
| review and approval | `cms-events` on the form or catalogue event → `admin-api` moving the record, `notification` telling a human |
| stale-content reminder | `interval-node` → `admin-api` listing by date → `notification` |
| delete submissions past a retention period | `interval-node` → `admin-api` listing submissions → `admin-api` deleting the ones past it |

Report a requirement of this shape as buildable from the platform's own primitives rather than as unsupported, and say which nodes it needs. What a flow cannot do is anything the Admin API cannot: it holds no extra reach, and every call it makes is an ordinary one.

## Common mistakes

- **Saving only your own nodes.** The save replaces everything; read the configuration and send it back whole.
- **Treating `rev` as a version to increment or send back.** It is a hash of the content, and the save takes no `rev` at all.
- **Assuming a saved flow needs arming.** It runs from the moment it is saved, and removing it from the configuration stops it just as immediately.
- **Running `execute-to` on a node that is not saved, or behind a timer.** The first answers not found, the second times out at thirty seconds.
- **Promising a node kind without reading `GET /workflows/nodes`.** Which sets are installed differs between instances.
- **Calling a schedule-shaped requirement unsupported** because no operation is named after it.

→ `mcp/docs/api/workflow-nodes` · `mcp/docs/api/journal`
