# Collections and their rows

A collection is a named list of rows an instance keeps outside the page tree: each row holds form-shaped data, and a site reads the collection by its marker. This document covers creating one, adding and reading rows, and the two delete operations — where the permission is not the one the key names suggest.

Read it before clearing rows, because one field in the delete body overrides the rest.

→ `mcp/docs/api/form-submissions#what-a-submission-carries` · `mcp/docs/api/admins-and-permissions#permissions-are-dotted-keys`

## Addressing a collection by id or by marker

Every collection has a numeric `id` and a text `identifier`, and the operations come in both forms. The row operations are split between them, so the pair you need for one task is often one of each:

| Operation | Path |
|---|---|
| `AdminCollectionsController_create` | `POST /integration-collections` |
| `AdminCollectionsController_findAll` | `GET /integration-collections` |
| `AdminCollectionsController_findAllRowsByCollectionId` | `GET /integration-collections/{id}/rows` |
| `AdminCollectionsController_createCollectionRow` | `POST /integration-collections/marker/{marker}/rows` |
| `AdminCollectionsController_removeRow` | `DELETE /integration-collections/{id}/rows/{rowId}` |
| `AdminCollectionsController_removeAllRows` | `DELETE /integration-collections/{id}/rows` |

Adding a row is addressed by marker; reading and deleting rows are addressed by the numeric id. Read the collection once and keep both values rather than guessing which an operation takes.

## Create a collection and add a row

A collection needs an `identifier` and a locale-keyed title, and nothing else:

```json
{ "identifier": "city-guides",
  "localizeInfos": { "en_US": { "title": "City guides" } } }
```

The answer is `201` with the created record, including the `id` the row operations need. A row is then posted under the collection's marker, with its values as the ordinary list of marker-and-value pairs:

```json
{ "formIdentifier": "city-guides",
  "formData": [ { "marker": "name", "value": "Lisbon" } ] }
```

Reading rows answers `{ "items": [...], "total": <number> }`; each row carries `id`, `createdDate`, `updatedDate`, `formData` and, where a row was created against an entity, `entityId` and `entityType`. A numeric id no collection has answers `404 Collection not found`.

## Deleting rows needs the row key not the collection key

Both delete operations — the single row and the bulk one — require **`collections.row.delete`**. That is the one worth checking against an assumption: `collections.collection.delete` opens neither of them, and an admin holding only that key gets `403 Forbidden resource` from both, with the message naming nothing.

So the key that deletes a whole collection does not let you empty one, and clearing rows is not a weaker form of the same right. Ask for `collections.row.delete` by name.

A `rowId` the collection does not hold answers `404 Row not found`.

## deleteAll overrides the ids you send with it

The bulk delete takes a body with two fields, and they are not independent:

```json
{ "deleteAll": false, "ids": [11, 12] }
```

With `deleteAll: false` — or with the field left out altogether — the listed `ids` are removed and nothing else. With **`deleteAll: true` every row in the collection goes, and `ids` is ignored**: sending `{"deleteAll": true, "ids": [11]}` empties the collection, and the answer is the same `200 true` either way. There is no response field that says how many rows went, so the body you sent is the only record of what you asked for.

Decide which of the two you mean before you build the body, and never carry `deleteAll: true` over from an earlier call.

## What the bulk delete answers and what it does not check

A successful call answers `200` with a body of `true`, and an id the collection does not hold is **accepted in silence** — `{"deleteAll": false, "ids": [999]}` answers `200 true` having deleted nothing. So the status proves the call was understood, not that any row existed. Read the rows back and compare the count.

Two malformed bodies fail differently. Neither a `deleteAll` nor a usable `ids` — `{"deleteAll": false, "ids": []}` — answers `400 Not enough data for deletion`. A body with no `ids` key at all, `{}`, answers `5xx`: send both fields and the call behaves.

## Common mistakes

- **Asking for `collections.collection.delete` to clear rows.** Both row deletes need `collections.row.delete`.
- **Sending `deleteAll: true` with a narrow `ids` list.** The list is ignored and the collection is emptied.
- **Reading `200 true` as proof rows existed.** Unknown ids are accepted silently.
- **Posting a row to the numeric id.** Row creation is addressed by the collection's marker.
- **Omitting `ids` from the bulk body.** It answers `5xx`; send both fields.
- **Expecting a count back from the bulk delete.** It answers a bare `true`.
- **Deleting the form the collection is built on.** It answers `409` until the collection is re-pointed or gone — `mcp/docs/api/forms-and-form-data#delete-a-form`.

→ `mcp/docs/api/content-api-permission-rules` · `mcp/docs/api/silent-no-ops`
