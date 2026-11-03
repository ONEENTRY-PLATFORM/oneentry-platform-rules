# Naming a data environment by marker

An environment's own name is fixed for its whole life, so a client configured with it has to be edited to be pointed anywhere else. A **marker** removes that: it is a stable name bound to one environment, and an administrator can re-point it without anyone touching the clients that send it.

Read this when somebody asks you to "point staging at the new environment", or when a request names an environment and you are deciding which of the two headers to quote.

→ `mcp/docs/api/environments#a-request-names-its-environment-in-a-header` · `mcp/docs/api/environments#which-data-set-your-own-calls-reach` · `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission`

## A marker is a second way to name an environment

A request sends `x-environment-marker: <marker>` instead of `x-environment: <name>`. Everything after the name is resolved is identical: the same set of data, the same refusal when the environment is not ready, the same answer shapes.

One marker names one environment; an environment carries as many markers as a project needs. Re-pointing one moves every client that sends it, at once and with no change on their side. That is the only reason markers exist — if nothing will ever need moving, the environment name is enough.

A marker starts with a lowercase letter or a digit and continues with lowercase letters, digits, hyphens and underscores, up to sixty-four characters. The shape is wider than an environment name, which takes no hyphens, so `shop-staging` is a marker but never an environment name.

Sending both headers is allowed when they name the **same** environment. When they name two different ones the request answers `400 x-environment and x-environment-marker point to different environments` — picking one of the two silently would write somewhere the client did not intend.

## What a marker that resolves to nothing answers

A malformed marker and a marker nothing is bound to answer the way a bad environment name does, and for the same reason: the check runs before credentials are looked at.

| Surface | Answer |
|---|---|
| Admin API | `400` naming the cause — `Invalid x-environment-marker header value` or `Unknown environment marker "<marker>"` |
| Content API | `401 Invalid or missing app token` |

So a `400` here is never about your token, and on the Content API the mistake is indistinguishable from a token problem. When a storefront starts answering `401` right after somebody changed its configuration, check the spelling of the marker before reissuing anything.

The public real-time channel reads the marker too — in the connection headers, or beside the application token in the connection payload as `environmentMarker`. A marker it cannot resolve **closes the connection** instead of opening one on the main set of data.

→ `mcp/docs/api/environments#what-a-wrong-environment-name-answers`

## Managing the markers of an environment

| operation | what it does |
|---|---|
| `AdminEnvironmentMarkersController_findAll` | lists every marker with the id of the environment it names |
| `AdminEnvironmentMarkersController_create` | binds a new marker, taking `marker` and `environmentId` |
| `AdminEnvironmentMarkersController_update` | re-points an existing marker, taking `environmentId` |
| `AdminEnvironmentMarkersController_remove` | removes a marker |

Reading the list is open to any signed-in administrator — it is what you quote to whoever configures a client. The three writes need `environments.manage` and answer `403 Forbidden resource` without it, exactly as the registry of environments does.

A marker value already in use answers `409`. An `environmentId` no environment has answers `404`. Binding a marker to an environment that is still being created is accepted on purpose, so the marker is ready for the moment the environment is.

Deleting an environment takes its markers with it, and each of those names becomes free for reuse. Nothing warns you first: read the list back after a delete if anything was configured against one of them.

## Why renaming a marker does nothing

**The marker value cannot be changed.** The update reads `environmentId` and nothing else, so a `marker` sent beside it answers `200` and leaves the old value in place — one more write that reports success and does not do the work. To rename one, remove it and bind the new value.

Re-pointing is the operation that does work, and it is the one to reach for when the request is "move this client": the client's configuration stays exactly as it is.

→ `mcp/docs/api/silent-no-ops#the-table`

## What this client can and cannot do here

**No tool here sends either header.** `cms_api_call` takes no headers, so you cannot read or write an environment by naming it, by name or by marker. Maintaining the marker list is therefore work you do *for* other clients, not a way to move your own session.

Your own calls land wherever the server was started, and `cms_whoami` is the only thing that tells you where. Re-pointing a marker does not move this session, and a write you make after re-pointing one still goes to the same place it went before.

So when a human asks you to point a client at another environment: re-point the marker, then tell them which clients send it. When they ask you to *write* something in that environment, check `cms_whoami` first and say so if it names a different one.

→ `mcp/docs/server/cms-api-call`

## Common mistakes

- **Renaming a marker by sending `marker` to the update.** It answers `200` and keeps the old value; remove and bind instead.
- **Expecting a re-pointed marker to move your own session.** The header is the operator's choice, not yours.
- **Reading a Content API `401` as a dead token.** An unresolvable marker answers the same message.
- **Using a hyphenated marker as an environment name.** Environment names take no hyphens.
- **Assuming a deleted environment leaves its markers behind.** They go with it, and the names free up.
