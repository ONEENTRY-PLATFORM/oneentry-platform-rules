# End to end workflows

Recipes that chain the tools together for the tasks agents are actually asked to do. Each one names the documents to read at the point they matter.

They all share the same skeleton: find out what exists, read the rules for the entity, dry run, confirm, read back.

→ `mcp/operating-rules` · `mcp/docs/server/cms-api-call`

## Start of a session

1. `cms_guide` — the policy, the API map, any warnings.
2. `cms_whoami` — the allow level and the permissions you actually hold. Do this before planning writes, not after being refused.
3. `cms_docs_read { "docId": "mcp/operating-rules" }` — before your first write.

If the guide says the knowledge is degraded or the catalog is empty, stop and tell the human. Neither state is something to work around.

## Explore an unfamiliar instance

```text
cms_api_search { "query": "pages", "mutating": false }
cms_api_call   { "opId": "AdminLocalesController_findAllActive" }
cms_api_call   { "opId": "AdminGeneralTypesController_findAll" }
```

Read what exists before deciding what to add. The active locales and the general types shape every payload you will write afterwards, and both are instance data rather than constants.

→ `mcp/docs/api/baseline-data#how-to-check-what-your-instance-already-has`

## Create a content entity

1. Read the entity's document — `mcp/docs/api/pages`, `mcp/docs/api/products`, `mcp/docs/api/blocks`.
2. Read the active locale codes.
3. Read the attribute set the entity will use, and build the inner attribute keys from it.
4. `cms_api_describe` on the create operation; copy the example for any loose field.
5. Call with `dryRun: true`, check the policy decision.
6. Call for real; keep the id from the response.
7. **Read it back by id** and confirm the attributes are populated.

Step 7 is not optional. An attribute map one level deep instead of two answers 201 and stores nothing.

→ `mcp/docs/server/payload-conventions#attributessets-is-two-levels-deep`

## Update an existing entity

1. Read the entity by id first. You need its current state to know what you are changing, and some update operations expect fields you must carry over.
2. Send the smallest body that expresses the change.
3. Dry run, read the `target`, then send.
4. Read back.

For products specifically, always include `blocks` — send `blocks: []` when you have nothing to set — and never include `forms`.

→ `mcp/docs/api/products`

## Reorder something

Never patch a position field. Find the entity's dedicated position operation:

```text
cms_api_search { "query": "menus position" }
```

Ordering is a lexorank string on parent-scoped operations and a number on flat lists. The dedicated operation handles the surrounding items; a direct patch does not.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## Delete something

1. Confirm you have the right entity: read it by id and check its identifier or marker.
2. `cms_api_call` with `dryRun: true` — this returns `needsConfirm`, the target and a token.
3. Tell the human exactly what will be deleted, quoting the target's identifying fields.
4. Wait for them to agree in this conversation.
5. Repeat the identical call with `confirm`.

The token lasts five minutes. If they take longer, dry run again — that cost is intentional.

→ `mcp/docs/server/confirm-and-dry-run`

## Grant access to something

A `403 Permission data not found` from the Content API is a permission that exists but is not granted to the group, not a permission that needs creating. Around 110 of them are provisioned already.

1. List the user groups; identify the group in question — `guest` for unauthenticated visitors.
2. List its permissions and find the path.
3. Update the rules on the **existing** record.

Creating a permission that already exists fails; creating a second group called `guest` succeeds and does nothing useful.

→ `mcp/docs/api/users-and-groups` · `mcp/docs/api/baseline-data#content-api-permissions`

## Investigate why a list looks wrong

1. Is the response truncated? A `_truncated` envelope means you saw part of it. Narrow the query.
2. Was the entity created seconds ago? Lists lag; read it by id to confirm it exists.
3. Are you filtering on an attribute that is not indexed? Only indexed attributes are usable in list filters.
4. Is the locale right? Content written for one locale is absent in another.

→ `mcp/docs/api/index-attributes` · `mcp/docs/server/response-shaping`

## Verify a change end to end

Read back by id, then confirm the change is visible through the surface a user would actually use — the listing, the filter, the search. Allow a few seconds between the two.

`mcp/docs/api/verification-recipes` gives per-area sequences with the fields worth asserting.

## When to stop and ask

- The allow level or a permission blocks you. Report the exact level or permission string and stop.
- A gated operation needs confirmation. Show the target and wait.
- Something the human referred to does not exist on this instance. Ask rather than creating it.
- A supported operation answers 5xx. Report the operation id and the request; do not retry with a modified body.
