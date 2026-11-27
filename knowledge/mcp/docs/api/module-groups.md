# Module groups in the admin menu

The admin panel menu sorts its sections (modules) into groups such as content, sales, audience and system. Groups are an instance setting shared by every administrator, not a personal preference.

Read this before reorganising the menu or when a section appears under the wrong heading.

→ `mcp/docs/api/modules#module-visibility` · `mcp/docs/api/modules#modules-and-permissions`

## Reading the groups and which group a section is in

`GET /api/admin/modules/groups` lists the groups in menu order, each with `id`, `identifier`, `localizeInfos` (title per locale), `isVisible` and `position`. Any signed-in administrator can read it.

`GET /api/admin/modules` carries `groupId` on every section. `null` means the section belongs to no group, and the panel shows it in its default place. A newly created custom module starts with `groupId: null`.

## Which permission keys change the groups

| Call | Key |
|---|---|
| Create, rename, delete or reorder a group | `settings.modules.changePositions` |
| Show or hide a group | `settings.modules.switching` |
| Move a section to another group | `settings.modules.changePositions` |

A missing key answers `403` and nothing changes.

## Creating, renaming and deleting a group

- `POST /api/admin/modules/groups` with `{ "identifier": "promo", "localizeInfos": { "en_US": { "title": "Promo" } } }` adds a visible group at the end of the list. An identifier another group already holds answers `400`.
- `PUT /api/admin/modules/groups/{id}` changes the identifier or the titles; send only what changes.
- `DELETE /api/admin/modules/groups/{id}` removes the group. Its sections stay and get `groupId: null`. A second delete of the same id answers `404`.
- `PUT /api/admin/modules/groups/{id}/position` with `{ "position": { "leftObjectId": null, "rightObjectId": 12 } }` moves the group; both ids are group ids.

## Hiding a group

`PUT /api/admin/modules/groups/{id}/change-visibility` with no body flips `isVisible` and returns the group. It is the only way to change it: `isVisible` sent to the create or rename call is ignored, so a new group is always visible.

## Moving a section to another group

Only `PUT /api/admin/modules/{id}/position` changes a section's group, through its optional `groupId`:

- absent: the group stays and only the order changes;
- `null`: the section leaves its group;
- a group id: the section moves there; an unknown id answers `404` and nothing moves.

`groupId` sent to `POST /api/admin/modules` or `PUT /api/admin/modules/{id}` is ignored. The call still succeeds, so read `GET /api/admin/modules` back to confirm a move.

## Common mistakes

- Moving a section with `PUT /api/admin/modules/{id}` and trusting the `200`. Use the position call.
- Sending `isVisible: false` when creating a group to hide it. Create it, then call change-visibility.
- Expecting a data environment to share the main set's groups. Each environment keeps its own, and a transfer between them does not carry group changes.

→ `mcp/docs/api/modules#address-them-by-identifier` · `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`
