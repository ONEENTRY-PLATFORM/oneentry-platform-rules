# The cms api describe tool

Full detail for one operation: its parameters, its request body schema, the permission it requires, its risk level, and whether it is permanently confirm-gated.

Call it between `cms_api_search` and `cms_api_call`. Building a body without it is guessing.

→ `mcp/docs/server/cms-api-call` · `mcp/docs/server/payload-conventions`

## Arguments

One argument, `opId`, taken from `cms_api_search`.

```json
{ "opId": "AdminPagesController_update" }
```

## What comes back

```json
{ "opId": "AdminPagesController_update", "method": "PUT",
  "path": "/pages/{id}", "fullPath": "/api/admin/pages/{id}",
  "tag": "Pages", "summary": "Update page",
  "permission": "content.update", "risk": "write", "alwaysConfirm": false,
  "params": [ { "name": "id", "location": "path", "required": true, "schema": { … } } ],
  "body": { "contentType": "application/json", "schema": { … }, "required": true },
  "looseFields": [ { "field": "localizeInfos", "sourceType": "CommonLocalizeInfos" } ],
  "responseSummary": "The updated page",
  "next": "Search the knowledge base with cms_docs_search for this entity, then call cms_api_call with dryRun: true first." }
```

- `path` is what you pass to `cms_api_call`; `fullPath` includes the API prefix and is informational.
- `permission` is `null` when the operation declares none.
- `risk` is `read`, `write` or `destructive`, derived from the HTTP method.
- `alwaysConfirm: true` means a confirm token is required at every allow level.

## Parameters

Each entry in `params` has a `name`, a `location` of `path`, `query` or `header`, a `required` flag and a schema. Path parameters are always required.

Pass them by location, not in one bag:

```json
{ "opId": "AdminPagesController_update", "path": { "id": 42 }, "query": { "langCode": "en_US" } }
```

Omitting a path parameter is caught before anything is sent:

```text
Missing path parameter "id" for AdminPagesController_update (/pages/{id}). Pass it as path.id.
```

## Loose fields

`looseFields` lists the body fields whose declared type is not a JSON Schema type. The API document these schemas come from expresses some types in the platform's own type language, and those cannot be normalised.

For a loose field, **the `example` in the schema is the contract**. Copy its shape exactly; do not infer structure from the type name. Client-side validation of your body is advisory only, so a wrong shape is not caught here — it is caught by the instance, usually as a 400, and sometimes not at all.

The two fields that most often appear here are `localizeInfos` and `attributesSets`, and both have shapes that are easy to get wrong.

→ `mcp/operating-rules#trust-the-example-not-the-type`

## Reading the body schema

`body.required` says whether a body is expected at all. `body.schema` is a JSON Schema, with the caveat above about loose fields.

Two habits save turns:

- Start from the schema's `example` when there is one, and change only what you need. The example is real output from a working call far more often than a hand-written schema is correct.
- Send the minimum the operation requires, then read the entity back and add fields in a second update. A large speculative body fails as one opaque 400; a small one fails informatively.

## Unknown operation ids

```json
{ "error": "Unknown opId \"AdminPageController_update\".",
  "didYouMean": ["AdminPagesController_update", "AdminPagesController_create"],
  "hint": "Operation ids come from cms_api_search — do not construct them by hand." }
```

The suggestions are near-matches on spelling. If none of them is what you meant, the operation probably does not exist under the name you assumed — search by entity instead.

## Before you call

Check three things in the describe output before moving on:

1. **`risk` and `alwaysConfirm`** — do you already know this will need a confirm token?
2. **`permission`** — does `cms_whoami` show the admin holding it? If not, ask for the grant now rather than after a refusal.
3. **`looseFields`** — is there a field whose example you need to copy rather than invent?

Then call with `dryRun: true` if the operation mutates anything.
