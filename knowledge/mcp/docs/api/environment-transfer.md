# Environment transfer requests with several kinds and duplicates

How to build an environment transfer request that carries more than one kind of entity, what `overwrite` reaches, what delete mode removes, and what to do when the target already holds a **different** entity under the same key — a duplicate. The basics of the operation, the empty `items` of a real run and the kinds that never travel are in the environments document; read that first.

Read this before you send a transfer whose dry run shows `status: "duplicate"` or `status: "conflict"`, or before you group several kinds into one run.

→ `mcp/docs/api/environments#copying-content-from-one-environment-to-another` · `mcp/docs/server/confirm-and-dry-run`

## Several kinds of entity travel in one request

`items` holds one element per kind — `{ type, ids, overwrite?, withRows? }` — and every kind listed goes in the same run. You do not have to order them: what one entity needs in order to exist is added to the plan as a dependency and written before it, whatever position its kind had in `items`.

So "copy these pages and that menu" is one dry run and one real run, not two. Two runs give the human two plans to approve and leave the target half-copied between them.

## Each kind may appear only once in items

A second element with the same `type` answers `400 Each type may appear only once in items`, and nothing is planned or written. Put every id of one kind into the single `ids` list of that kind's element.

The same rule means `overwrite` and `withRows` are decided once per kind in a request. When some pages may be overwritten and others may not, send two separate transfers.

## Overwrite is set per item and dependencies inherit it

A dry run marks an entity `conflict` when the target copy was edited after it last arrived there. A real run leaves a conflict as it is unless the element that asked for it carries `overwrite: true`.

The flag reaches further than the ids you named. A dependency the plan added on behalf of an element takes that element's flag: with `overwrite: false` a parent page edited in the target stays `conflict`; with `overwrite: true` on the element that pulled it in, the same parent reads `update` and is written over.

So before you set `overwrite: true`, read every `conflict` in the dry run, including those marked `isDependency: true`, and show them to the human. The flag writes over all of them, not only over the ids you named.

## A duplicate is a different entity with the same key

A dry run marks an entity `duplicate` when the target already holds a **different** entity under the same unique key: a page with the same address, compared without regard to case, or another kind with the same `identifier`. "Different" means it is not the copy an earlier transfer of this source entity made, so updating it in place would join two unrelated records.

The plan item says which entity it collided with and lays the two side by side:

- `duplicate` — `{ field, value, targetId }`: the key, its value, and the id of the target entity.
- `fields[]` — each field with its `source` and `target` value, `isEqual`, and `isCombinable` (both sides are objects).

Show the human both sides. Which one wins is their decision, not yours.

## A duplicate without a resolution is not written

A real run that names a duplicate but carries no resolution for it still answers `201`, and writes nothing for that entity. The target keeps its own entity untouched and no second one is created.

Because a real run returns an empty `items`, that response cannot tell you a duplicate was passed over. The dry run is the only place a duplicate shows. Always run a dry run first, and treat any `duplicate` in it as a question for the human before the real run.

## Resolving a duplicate with skip replace or merge

A resolution goes in `duplicates` on the request, one per duplicate, with the `syncUid` copied from the dry run's plan item (the source entity's id across environments):

```json
{
  "from": "public",
  "to": "stage",
  "items": [{ "type": "pages", "ids": [101] }],
  "duplicates": [
    {
      "type": "pages",
      "syncUid": "0f1c4f64-4b2a-4a3f-9d2e-6a5b7c8d9e01",
      "strategy": "merge",
      "fields": { "isVisible": "target", "localizeInfos": "combine" }
    }
  ]
}
```

- `skip` — the target keeps its own entity; nothing is written.
- `replace` — the source entity is written over the target one, which keeps its id.
- `merge` — one entity built field by field from `fields`.

After `replace` or `merge` there is one entity in the target, and the next dry run of the same source entity shows it as `update` rather than `duplicate`. The two records are joined from then on.

## What a merge rule may say

Each key in `fields` is a field name from the plan item's `fields[]`, and each value is one of three rules:

- `source` — take the source value.
- `target` — keep the target value.
- `combine` — join two objects key by key; where both have a key, the source wins. Only for fields with `isCombinable: true`.

A field with no rule takes the **source** value. So `merge` with only `{ "isVisible": "target" }` writes the source's title and content and keeps the target's visibility.

A rule outside the three answers `400 Unknown merge choice` naming the rule and the field, before anything is planned. `combine` on a field that is not an object on both sides fails that one entity in the run and leaves the target unchanged — and since a real run's `items` is empty, nothing in the response says so. Use `combine` only where the dry run shows `isCombinable: true`.

## A resolved duplicate still reads duplicate in a dry run

A dry run sent with resolutions still marks each of those entities `duplicate`, and adds a `strategy` key with the resolution that will apply. That is the confirmation the resolution was matched, not a sign it was ignored.

A resolution whose `syncUid` no longer points at a duplicate — the collision was resolved in the meantime, or the id was mistyped — is ignored, and the response adds a line to `warnings` naming the type and the `syncUid`. Read `warnings` before the real run: a mistyped `syncUid` is reported there, and the real duplicate it was meant for is then passed over without being written.

## An overwrite keeps the old version except on error pages

When a run writes over an entity that already exists in the target, the target's previous state is kept in that entity's version history first, so a deliberate overwrite can still be rolled back from there. That holds for every kind that travels except error pages: an error page is written over with nothing kept, and its version history stays empty afterwards.

So before you overwrite error pages, read the error page in the target and show the human its current values. There is no copy to restore them from later.

## Delete mode removes every target-only entity of a kind

`mode: "delete"` does not delete the ids you name. For each kind in `items` it compares the whole kind in both environments and removes every target entity that has no counterpart in the source. `ids` is accepted and ignored in this mode, so naming one entity does not narrow the run to it.

- A real run needs `confirm: true`. Without it, and without `dryRun: true`, the call answers `400` and nothing is planned or deleted. It does not fall back to returning the plan.
- A dry run with `dryRun: true` lists each entity that would go, with `status: "delete"`, or `status: "blocked"` and a `reason` when an ordinary delete of it would be refused too.
- Nothing deleted this way is kept in version history. There is no rollback.

Always send the dry run first, show the human the whole list per kind, and send the real run only after they agree to every entity on it.

## Real runs into one target run one at a time

Real runs that name the same `to` are carried out one after another, in the order they were accepted, whatever environment each one copies from. Each run plans against the target as the run before it left it. Runs into different targets do not wait for each other.

So sending the same real run twice, by a repeated click or by two people at once, writes each entity once: the later run finds the copy the earlier one made and treats it as an `update`. There is no need to guard against a repeat on your side.

A queued run builds its plan only when it starts. If another run into the same target was accepted before yours, the dry run you showed the human may describe a target that run is about to change. Let the earlier run finish, read the target, and dry-run again before you ask for approval.

## Common mistakes in a transfer request

- **Repeating a kind in `items`.** `400`; one element per kind, all its ids in one list.
- **Sending the real run without a dry run.** A duplicate is passed over with a `201`, and nothing in the response shows it.
- **Reading `duplicate` with a `strategy` as "not resolved".** The dry run always shows duplicates; `strategy` says what will happen.
- **Setting `overwrite: true` after reading only the ids you named.** Dependencies take the flag too; read their conflicts first.
- **Using `combine` on a plain value.** It fails that entity without a word; check `isCombinable`.
- **Naming ids in delete mode to narrow it.** They are ignored; the whole kind is compared and every target-only entity goes.
- **Approving a dry run while another run into the same target is pending.** Runs into one target go one at a time; dry-run again after the earlier one finishes.

→ `mcp/docs/api/environments#copying-content-from-one-environment-to-another` · `mcp/docs/server/confirm-and-dry-run`
