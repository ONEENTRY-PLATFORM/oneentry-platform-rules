# Confining an admin to part of the page tree

`pages.scope` is the admin permission that limits an account to one or more branches of the page tree. The ordinary keys decide whether an account may write pages at all; this one decides **which** pages it may write.

Read it before delegating a section of a site to someone, and when a page write answers `403` while the same account edits other pages happily.

→ `mcp/docs/api/admins-and-permissions#an-unrecognised-permission-key-is-refused-with-400` · `mcp/docs/api/pages#the-page-tree`

## The value is a list of page ids not a boolean

Almost every permission is `true` or `false`. Two are not: `admins.modules` takes a list of area identifiers, and `pages.scope` takes a list of page ids.

```json
{
  "permissions": {
    "pages.get": true,
    "pages.create": true,
    "pages.update": true,
    "pages.delete": true,
    "pages.scope": [12, 40]
  }
}
```

Those ids are the roots of the allowed branches. Each listed page and every descendant below it, to any depth, is in scope — the listed page itself included.

Nothing checks that the ids exist. A list naming a page that was deleted is stored happily and simply grants nothing, so read the pages back after granting.

## There is no empty list

`[]` is refused with `400`. So is `true`, a bare number, a string, and a list carrying `0`, a negative number, a fraction or a numeric string. The refusal is the one an unrecognised key gets, names `pages.scope` as the offender, and stores **nothing** — not the scope, and not the valid keys sent beside it.

To give an account the whole tree back, send its permissions **without the key**. Absence means no restriction, and it is the only way to lift one.

The map is written whole, so leaving the key out has to be deliberate: read the map, delete that one entry, send the rest back untouched.

→ `mcp/docs/api/admins-and-permissions#never-replace-an-admins-permission-map`

## Which page operations the scope governs

| Operation | What has to be in scope |
|---|---|
| `AdminPagesController_create` | the `parentId` in the body |
| `AdminPagesController_update` | the page id in the path |
| `AdminPagesController_remove` | the page id in the path |
| `AdminPagesController_deleteMany` | every id in the body list |

Reads are not governed by it. A scoped account still lists the root pages and reads any page by id or by page URL, so what it can see and what it can change are different sets.

## Creating a page needs a parent inside the scope

The check is on the parent, because that is what puts the new page inside a branch. A scoped account therefore **cannot create a root page**: a body with no `parentId`, or with `parentId: null`, answers `403`.

A `parentId` that is present but is not a positive whole number — `0`, a negative number, a non-numeric string — answers `403` too. Do not read that as an argument the operation would otherwise have accepted; a scoped account has no way to create a page outside its branches.

A `parentId` naming a page that does not exist answers `403` as well, not `404`. The same holds for the path id on update and delete: outside the scope and never existed are the same answer, so a `403` here is not evidence that the page is real.

## A bulk delete is refused whole

`AdminPagesController_deleteMany` checks every id before it deletes anything. One id outside the scope refuses the call and **nothing** is deleted — the ids that were in scope survive untouched.

An `ids` that is not a list, or a list carrying a value that is not a positive whole number, is refused the same way.

So a `403` from this operation never leaves a half-finished delete behind. Split the list, verify each id sits under one of your scope roots, and send it again.

## What a refusal tells you

The `403` names the page ids that caused it, so one call finds every offending id in the request rather than one per attempt. When the refusal is about the parent of a new page it says so instead.

Read it as "this page is outside the branches I was given", not as "I lack the permission to edit pages". The two need different actions:

| Reading | Action |
|---|---|
| The account lacks `pages.update` | Ask for the key |
| The account has it, the page is elsewhere in the tree | Ask for the branch to be added to `pages.scope`, or work on a page inside it |

`cms_whoami` shows the map, so check whether `pages.scope` is in it before reporting a page write as impossible.

## The check is on the page not on the destination

Update and delete look at where the page **currently sits**. An update carrying a `parentId` that points outside the branch is not refused, and the page ends up outside.

That has a consequence worth knowing before you send it: the account can no longer edit the page it just moved. Every later update or delete of it answers `403`, and only an unscoped account can move it back.

Several page operations are not governed by the scope at all — `AdminPagesController_updatePagePosition`, `AdminPagesController_moveMany`, `AdminPagesController_copy` and `AdminPagesController_setVisibility`. Do not assume a scoped account is confined to its branches for everything it can do to a page; assume it for the four operations in the table above and check the rest.

## Common mistakes

- **Sending `[]` to lift the restriction.** It is a `400`. Omit the key.
- **Sending the ids as strings.** `["12"]` is refused; `[12]` is not.
- **Reading a `403` on a page write as a missing permission.** Check `pages.scope` first.
- **Retrying a bulk delete id by id after a `403`.** Nothing was deleted; fix the list instead.
- **Treating a `403` as proof the page exists.** A page that never existed answers the same.
- **Assuming a scoped account cannot reach other pages at all.** Reads are unrestricted.

→ `mcp/docs/api/pages#update-a-page` · `mcp/docs/api/admins-and-permissions#the-local-check-and-what-it-means`
