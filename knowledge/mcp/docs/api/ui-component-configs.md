# UI Component Configs

How the admin panel's own screens are configured: saved screen templates, assigning a template to an admin, the recipes that feed extra columns, and the constructor agent that proposes a template from a sentence.

Read it when a human asks to change how a table, a form or a record editing page looks in the admin panel for themselves or for someone else. It changes nothing on the public site.

→ `mcp/docs/api/ai-gateway#jobs-are-asynchronous` · `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## Screens templates and recipes

- A **screen** is one part of the admin panel a template can be saved for — a table, a form, the page on which one record is edited, or the tab panel of a section. It is named by a `configId`.
- A **template** — a preset in the API — is a named `config` saved for one screen. It belongs to the admin who created it.
- A **recipe** — a provider in the API — declares where a value that is not part of a row comes from, so a template can show it as a column.

A template's `config` is never a patch. For a table or a form it is the whole setting of the screen. For a record page it is the whole setting of the one part that template covers.

## A record page template covers one part of the page

Save it with `componentType: "entity"`. Without that the admin panel applies the whole `config` on assignment and replaces every other setting of the screen.

One template covers exactly one part:

- **no `attributeSetId`** — the page's tabs and sections. Keys in `config`: `hiddenTabs`, `tabOrder`, `hiddenSections`, `sectionOrder`.
- **with `attributeSetId`** — the field layout of that one attribute set and nothing else:

```json
{ "attributeSets": { "12": { "hiddenAttributes": ["price"],
                             "attributeOrder": ["price", "size"] } } }
```

Those values are the attributes' markers — their `identifier`, never their id. Which set an opened record turns out to use is not known in advance, so an admin is usually given several templates for the same page — one per part. Each is applied to its own part and leaves the rest alone.

The tab panel of a section carries tabs only: `hiddenTabs` and `tabOrder`, never `attributeSetId`. A hidden tab cannot be reached, and that includes its direct address — the panel moves to the first tab still shown.

Tab and section keys are admin panel identifiers — short strings such as `"1"` or `"version-history"`. No call returns them; read them out of a template already saved for that screen. Tables and forms never use `attributeSetId`.

## Screen keys a record page is saved under

`AdminUiComponentConfigsController_findScreens` answers with the table and form catalog plus every key a template already exists for. A record page or a section tab panel is in that answer only once someone has saved a template for it, so the list is not the full set of keys.

Record pages: `admin-page`, `content-page`, `catalog-product-page`, `collection-page`, `template-page`, `block-page`, `blocks-slide-edit-page-layout`, `form-page`, `event-page`, `discount-page`, `order-page`, `user-page`, `user-group-page`, `user-auth-provider-page`, `user-permission-page`.

Section tab panels: `catalog-navigation`, `orders-navigation`, `payments-navigation`, `users-navigation`.

## The attribute set is fixed when the template is saved

- `attributeSetId` is optional on `AdminUiComponentConfigsController_createPreset` and is an integer of 1 or more. `0` or a negative value answers `400`.
- It comes back on every template. `null` means the template covers the whole screen.
- `AdminUiComponentConfigsController_updatePreset` does not take it. A request carrying one answers `200`, saves the other changes and leaves the set as it was. To cover a different set, save a new template.
- `AdminUiComponentConfigsController_findPresets` filters by `configId` only. Filter `items` by `attributeSetId` yourself.
- Field layout has to be saved **with** `attributeSetId`. A template without it covers tabs and sections, and field layout in its `config` is not applied.

## Two permissions for the constructor

- `uiComponentConfigs.edit` — create, change and delete your own templates and recipes, and run the constructor agent.
- `uiComponentConfigs.assign` — list, make and remove assignments, and change or delete a template or recipe another admin owns.

Reading templates, recipes, screens and your own assignments needs neither. Changing someone else's template without `assign` answers `403`.

## Assigning a template to an admin

`AdminUiComponentConfigsController_assignPreset` takes `adminId`, `configId` and `presetId` and answers `201` with the assignment. The attribute set is taken from the template itself — the body has no field for it.

An admin holds at most one assignment per screen **and attribute set**. Assigning again for the same screen and the same set replaces the previous template and refreshes the date; assignments for other sets of that screen stay. The call is safe to repeat.

On a record page one admin can therefore hold one assignment for the whole screen and one for each attribute set at the same time.

- `400` — the template was saved for a different screen than `configId`.
- `404` — no such template, or no such admin.

`AdminUiComponentConfigsController_findAssignments` lists assignments, filtered by `adminId` or `configId`.

## Removing an assignment

`AdminUiComponentConfigsController_removeAssignment` takes `adminId` and `configId` in the path and answers `{ "ok": true }`.

With no query parameter it removes the assignment for the whole screen. With `?attributeSetId=<id>` it removes only the assignment for that set and leaves the others. An `attributeSetId` below 1 answers `400`.

It answers `404` when that admin has nothing assigned there — nothing to remove, not a failure to report.

The part whose assignment was removed returns to its default look the next time the admin panel loads, provided the admin has not changed that part themselves since it was assigned.

## What an admin sees applied to them

`AdminUiComponentConfigsController_findMyAssignments` returns the calling admin's assignments, one per screen and attribute set. Each carries `attributeSetId` — `null` for the whole screen — beside `componentType`, the template's `config` and `assignedAt`. The admin panel applies these on its own. There is no call to read what another admin sees other than listing their assignments.

## A template that is still assigned cannot be deleted

Deleting a template assigned to any admin answers `409` with the number of admins holding it. Remove those assignments first, then delete.

A recipe that a saved template refers to is protected the same way: its deletion answers `409` naming the screens that use it.

## Saving a template that refers to an unsaved recipe

Creating or updating a template checks every recipe its `config` names. A reference to a recipe that is not saved, not available on that screen, or missing a required binding is refused with `400` listing each problem.

Save the recipe first with `AdminUiComponentConfigsController_createProvider`, then the template. This matters most for the constructor agent's output, below.

## The constructor agent and its result

`AdminAiGatewayController_suggestUiComponentConfig` turns a sentence into a template for one screen. It needs `uiComponentConfigs.edit` and an AI account.

Required: `routerMarker`, `configId`, `prompt`, and `schema` — the description of allowed settings for that kind of component. Send `currentConfig` so the agent edits what is there, and an `idempotencyKey`.

The submission is a job. Once `completed`, `output` holds:

- `config` — the complete proposed setting, never a delta.
- `warning` — one sentence for the human, or `null`.
- `warningCode` — `null`, or `exhausted` when the run hit a limit before producing anything usable; `config` is then `{}`.
- `providers` — recipes the proposal uses that are **not saved** anywhere.
- `trace` — the steps the agent took. For diagnosis only; never present it as the answer.

The API description lists only the first three fields. The other two are present all the same.

## Applying what the constructor proposed

The job's result is a proposal. Nothing is saved, and nothing is shown to anyone, until you write it.

1. Check `warningCode`. On `exhausted`, stop and report.
2. Show the human `warning` if there is one, and what the proposal changes.
3. Save each entry of `providers` with `AdminUiComponentConfigsController_createProvider`, keeping its `id`.
4. Save `config` as a template with `AdminUiComponentConfigsController_createPreset` or `AdminUiComponentConfigsController_updatePreset`.
5. Assign it only if the human asked for it to apply to someone.

→ `mcp/docs/api/ai-gateway#a-completed-job-with-an-empty-result-ran-out-of-budget`

## Common mistakes

- **Treating a completed job as an applied change.** Nothing is saved until you save it.
- **Saving the template before its recipes.** Refused with `400`.
- **Deleting a template that is still assigned.** Refused with `409`; unassign first.
- **Assigning twice and expecting two.** One per admin, screen and attribute set; the second replaces the first. Assignments for different sets of one screen live side by side.
- **Saving a record page's field layout without `attributeSetId`.** It is never applied.
- **Saving a record page template without `componentType: "entity"`.** Assigning it replaces every other setting of that screen.
- **Expecting `findScreens` to name a record page.** It is in the answer only once a template exists for it.
- **Changing another admin's template with only `edit`.** Needs `assign`.
