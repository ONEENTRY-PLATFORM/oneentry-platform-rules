# UI Component Configs

How the admin panel's own screens are configured: saved screen templates, assigning a template to an admin, the recipes that feed extra columns, and the constructor agent that proposes a template from a sentence.

Read it when a human asks to change how a table or form looks in the admin panel for themselves or for someone else. It changes nothing on the public site.

→ `mcp/docs/api/ai-gateway#jobs-are-asynchronous` · `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## Screens templates and recipes

- A **screen** is one table or form of the admin panel, named by a `configId`. `AdminUiComponentConfigsController_findScreens` lists every key.
- A **template** — a preset in the API — is a named `config` saved for one screen. It belongs to the admin who created it.
- A **recipe** — a provider in the API — declares where a value that is not part of a row comes from, so a template can show it as a column.

A template's `config` is always the whole setting for its screen, never a patch.

## Two permissions for the constructor

- `uiComponentConfigs.edit` — create, change and delete your own templates and recipes, and run the constructor agent.
- `uiComponentConfigs.assign` — list, make and remove assignments, and change or delete a template or recipe another admin owns.

Reading templates, recipes, screens and your own assignments needs neither. Changing someone else's template without `assign` answers `403`.

## Assigning a template to an admin

`AdminUiComponentConfigsController_assignPreset` takes `adminId`, `configId` and `presetId` and answers `201` with the assignment.

An admin holds at most one assignment per screen, so assigning again to the same screen replaces the previous template. The call is safe to repeat.

- `400` — the template was saved for a different screen than `configId`.
- `404` — no such template, or no such admin.

`AdminUiComponentConfigsController_findAssignments` lists assignments, filtered by `adminId` or `configId`.

## Removing an assignment

`AdminUiComponentConfigsController_removeAssignment` takes `adminId` and `configId` in the path and answers `{ "ok": true }`. The admin returns to the default look of that screen.

It answers `404` when that admin has nothing assigned on that screen — nothing to remove, not a failure to report.

## What an admin sees applied to them

`AdminUiComponentConfigsController_findMyAssignments` returns the calling admin's assignments, one per screen, each with the template's `config` and `assignedAt`. The admin panel applies these on its own. There is no call to read what another admin sees other than listing their assignments.

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
- **Assigning twice and expecting two.** One per admin per screen; the second replaces the first.
- **Changing another admin's template with only `edit`.** Needs `assign`.
