# Form submission scope filters

Besides the keys that pick one entity, the form submission listing takes keys that narrow it to a part of the catalogue, the page tree or the user base — the same boundaries an interface module uses. This document covers those keys and the `400` answers that guard them.

Read it when listing the submissions about a category, a branch of pages or a group of users.

→ `mcp/docs/api/form-submissions#which-keys-the-filter-body-accepts` · `mcp/docs/api/interface-module-scopes#what-the-scope-of-each-base-holds`

## Keys that narrow submissions to a module

| Key | What it keeps |
|---|---|
| `pageIds` | submissions about pages in these branches, and about products linked to at least one such page. A submission about a catalogue page itself comes back unless `productFilter` is sent |
| `isNested` | with `pageIds`, `true` also takes subpages at any depth |
| `productFilter` | submissions about products that pass all these conditions, in the body shape of the category product listing |
| `pageTypes`, `pageFilter` | with `pageIds`: page types and page conditions as in the page listing. A page that fails hides its branch, and the submissions about its products unless another linked page passes |
| `userGroupIds`, `userAuthProviderIds` | for forms bound to the users section: submissions about users of these groups, subgroups included, and auth providers. Other submissions are untouched |
| `userFilter` | with either of the two above: conditions on the attributes of the user form, as in `POST /users/paginated` |

`pageIds` combines with every other key, `entityIdentifier` included, and `total` counts what is left.

→ `mcp/docs/api/page-listing-filters#narrowing-one-level-by-type-or-condition`

## What productFilter does with and without pageIds

With `pageIds`, sending `productFilter` — even as `[]` — keeps only the submissions about products for forms bound to the catalogue, dropping those about the category pages themselves. Without `pageIds`, an empty `productFilter` changes nothing.

## Two different isNested flags

The body's `isNested` is about subpages under `pageIds`. The `isNested` query parameter of the same route is about the reply hierarchy of submissions. Sending one where you meant the other is accepted and silently answers a different question.

## Combinations that answer 400

| Body | Answer |
|---|---|
| `pageIds: []` | `400 pageIds should not be empty` |
| a repeated id in `pageIds` | `400` saying the elements must be unique |
| `pageTypes` or `pageFilter` without `pageIds` | `400 pageTypes and pageFilter require pageIds` |
| `userFilter` without `userGroupIds` and `userAuthProviderIds` | `400 userFilter requires userGroupIds or userAuthProviderIds` |

## Submitting about a category page

A form bound to the catalogue takes a product id or a page URL as `moduleEntityIdentifier`. A URL makes the submission about the category page itself. A global binding takes it; a selective one takes it only if the page is selected, directly or under a nested entry, and not excepted. Otherwise the submission answers `400`.

→ `mcp/docs/api/forms-and-form-data#a-form-must-be-bound-before-it-accepts-submissions`

## Common mistakes

- **Sending `pageTypes` without `pageIds`.** It answers `400`; the types narrow a branch, not the whole form.
- **Expecting `pageIds` to return only page submissions.** Products linked to those pages come too.
- **Confusing the body and query `isNested`.** One is subpages, the other replies.

→ `mcp/docs/api/form-submissions` · `mcp/docs/api/interface-module-data`
