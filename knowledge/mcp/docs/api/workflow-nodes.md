# Configuring the platform nodes of a flow

The four nodes a flow is actually built from are `admin-api` (do the work), `cms-api` (ten domain actions), `cms-events` (react to something happening) and `interval-node` (decide when). Their configuration is in this document and in no API document: the node's own fields are not declared anywhere the catalogue can show you, and the one refusal that names a constraint names none of the values it would accept.

Read this before writing a node, and read the neighbouring document for how the configuration is stored and saved.

→ `mcp/docs/api/workflows` · `mcp/docs/api/events` · `mcp/docs/server/api-catalog`

## Configure the admin-api node

This is the node that does work, and the one most requirements end up resting on:

- `method` — the HTTP method, in either case: `get` and `GET` both work, as do `post`, `put` and `delete`.
- `path` — the Admin API path without the `/api/admin` prefix. A `{name}` placeholder in it is filled from `params`; one with no matching key fails the run with `Missing parameter: <name>`.
- `params` — the query parameters, as a **JSON string**, not an object. Left out, the incoming message's own `params` is used.
- `data` — the request body, also as a JSON string. Left out, the message's `data` is used.
- `operation` — optional, an operation id. Reading the flows back then adds an `outputExample` for it, empty where none is known.

Both strings are templates over the message, so `{{payload.id}}` inside one is substituted before the call — a node reading an id out of what reached it needs no code node in front of it.

```json
{ "id": "b1", "type": "admin-api", "z": "nightly",
  "method": "get", "path": "/pages/{id}",
  "params": "{\"id\":{{payload.id}},\"langCode\":\"en_US\"}",
  "wires": [ [] ] }
```

The node has no field for a token: the instance issues the request against its own Admin API. So a flow reaches any operation an operator could, and a write it makes is a real write.

## Which operations the admin-api node offers

`GET /workflows/nodes/admin-api/swagger.json?langCode=en_US` answers the whitelist the flow editor's picker shows — a small fraction of the API. Add `full=true` and the same call answers the whole Admin API instead.

Nothing stops the node calling a path outside the whitelist; the whitelist is what the editor offers, not what the node allows. This server reads the same document to build its catalogue, which is why a catalogue that looks short is a sign the unfiltered one was not available.

→ `mcp/docs/server/api-catalog`

## Configure the cms-api node

Here `method` is **not** an HTTP verb. It names one of ten actions, and anything else — `GET` included, and a `method` key that is absent altogether included — fails the run with `400`, a `success: false` body, and a message saying `property method has failed the following constraints: isEnum` while naming none of the values it accepts. There is no default.

The argument travels in `data` as a JSON string template, exactly as on `admin-api`. With no `data`, the incoming message's `payload` is the argument instead.

```json
{ "id": "a1", "type": "cms-api", "z": "nightly",
  "method": "findCollectionRows",
  "data": "{\"collectionIdentifier\":\"stock\",\"langCode\":\"en_US\",\"limit\":5,\"offset\":0}",
  "wires": [ [] ] }
```

Anything the node cannot find comes back in that same `400` shape — `Collection not found`, `Orders storage not found`, `Payment session not found`. That is the configuration working and the argument being wrong, which is the opposite of the `isEnum` answer: one says the value of `method` is not a value, the other says the work was attempted.

## The ten actions the cms-api node takes

Each takes a single object, and these are the keys it reads:

| `method` | Keys in its argument |
|---|---|
| `findCollectionRows` | `collectionIdentifier`, `langCode`, `limit`, `offset` |
| `getCollectionRow` | `collectionIdentifier`, `id`, `langCode` |
| `createCollectionRow` | `collectionIdentifier`, `langCode`, plus the row's own fields |
| `updateCollectionRow` | `collectionIdentifier`, `id`, `langCode`, plus the fields to change |
| `findOrdersByStatus` | `storageId`, `statusIdentifier`, `langCode` |
| `updateOrderStatus` | `storageId`, `orderId`, `statusIdentifier`, `langCode` |
| `getPaymentSessionById` | `id` |
| `getPaymentSessionByOrderId` | `orderId` |
| `createPaymentSession` | the payment-session body |
| `updatePaymentSession` | `id`, plus the fields to change |

A collection is addressed by `collectionIdentifier`, its textual identifier, never by its numeric id. Anything outside these ten — a page, a product, a form submission, a user — is `admin-api` work rather than this node's.

## Configure the cms-events node

`eventMarker` is the `identifier` of an **event that already exists on the instance**, not a name you compose. List the events with `AdminEventsController_findAll` and take one of theirs, or create the event first with `actions.isWorkflows: true` on it so that it is handed to a flow rather than sent as a mail.

`GET /workflows/nodes/cms-events/output-example?eventMarker=<identifier>` is how the message that will arrive is seen in advance. A marker no event carries answers `200` with `{ "outputExample": "" }` — an empty string, not an error — so an empty answer means there is no such event, never that the query was malformed. An absent or blank `eventMarker` answers `400 eventMarker is required`.

The message's shape follows the module the event watches: a product, a form submission, a sign-in, a user, a refund, an order or a discount. Events cannot watch the content module at all, so there is no page or block marker to put here and no `page_updated` to guess at — a timer over the page listing is what gets built instead.

Reading the flows back adds the resolved `outputExample` to every `cms-events` node, so a saved flow says what each of its event nodes receives.

→ `mcp/docs/api/events#which-modules-an-event-can-watch`

## Configure the interval node

`interval-node` has no inputs: it is where a timed flow starts. Its only configuration is `interval`, a whole number of **minutes**. A value that is not a whole number, or no `interval` key at all, means one minute. Each tick sends a message whose `payload` is a timestamp.

A saved flow is live at once, with no further call to arm it: measured on an instance, a flow saved with `interval: 1` fired a minute later and its `admin-api` node performed a real admin write.

An hourly job is `interval: 60` and a daily one `1440`. There is no cron expression and no "at 03:00", so a flow that has to act at a time of day ticks more often than that and decides in a `function` or `switch` node whether this tick is the one.

## Common mistakes

- **Sending an HTTP verb as the `cms-api` node's `method`.** That node names ten actions; `admin-api` is the one that takes verbs and paths.
- **Reading the `isEnum` refusal as a malformed body.** The field name is right and the value is not, and the message never lists the values.
- **Sending `params` or `data` as objects.** Both are JSON strings on both nodes, and an object there is not read.
- **Inventing an `eventMarker`.** It has to be an existing event's `identifier`, and an empty `outputExample` is how the instance says there is none.
- **Looking for a cms-events marker for a page or a block.** Events cannot watch content; use a timer.
- **Reading the editor whitelist as the limit of `admin-api`.** Ask the same endpoint with `full=true`.
- **Treating `interval` as seconds.** It is minutes, and `interval: 30` is half an hour.

→ `mcp/docs/api/workflows` · `mcp/docs/api/events`
