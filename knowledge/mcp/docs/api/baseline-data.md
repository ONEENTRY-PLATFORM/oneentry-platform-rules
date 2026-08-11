# Baseline data every instance already has

A OneEntry instance is never an empty database. Groups, permissions, modules, types, locales, block types, default templates and settings records are provisioned with it. An agent that treats the API as blank will either hit a duplicate error or — much worse — create a shadow record that nothing references.

Read this before the first `POST` of any session.

→ `mcp/operating-rules#baseline-data-already-exists-do-not-recreate-it` · `mcp/docs/api/attribute-sets`

## Read before you create anything

The identifiers below are what a typical provisioned instance carries. They are **not** a specification: instances are provisioned by different routes, ids do not match between them, and a minimal instance may be missing whole categories.

Treat every list here as "expect to find these", and a read against your own instance as the truth. The last section shows which reads to run.

## The rule that matters most

**List first, create second.**

Before creating a locale, module, general type, attribute set type, user group, permission, template or order status: list them and match by identifier or name. If it exists, use it. If it genuinely does not and a create looks necessary, say so to the human and stop — on these entity kinds a create is almost always a mistake, and several cannot be undone through the API.

## Silent duplicates versus hard failures

A hard failure is self-correcting: you learn the thing already exists and move on. A silent duplicate is not — the call answers 201, you believe you succeeded, and the instance now has a record that nothing points at.

| Creating a duplicate of | What happens |
|---|---|
| locale code | **fails** — the code is unique |
| general type name | **fails** — the name is unique |
| template identifier | **fails** |
| menu identifier | **fails** |
| order status identifier | **fails** — unique across the whole instance |
| import template identifier | **fails** |
| a permission path within a group | **fails** |
| a group-to-permission link | **fails** |
| **user group identifier** | **succeeds silently** — a second group nothing references |
| **module identifier** | **succeeds silently** — a shadow module |
| **attribute set type name** | **succeeds silently** — an ambiguous type |
| **a singleton settings record** | **succeeds silently** — a second record, only one of which is read |

Treat a failure on the first group as confirmation, not as a problem to route around.

## User groups and the guest group

A user group with identifier `guest` is provisioned with the instance, normally as id 1. Every Content API permission is attached to it, and unauthenticated access is decided by that group's rules.

The group identifier is **not unique**. Creating another group called `guest` succeeds and gives you a group no permission is attached to — which then looks like a permissions problem rather than a duplicate. To open up guest access, adjust the rules on the existing group. Never create one.

A second group with identifier `user`, for signed-in customers, exists on fully provisioned instances but is absent on minimal ones. Check for it; do not assume it, and do not invent it.

→ `mcp/docs/api/users-and-groups`

## Content API permissions

Around 110 permission records exist already, one per Content API path, each carrying a section and a set of rules, and all attached to the guest group — pages, products, blocks, forms, attribute sets, locales, templates, users, menus, payments, orders storage, events, filters, search and more.

A permission path is unique per group, so creating one that already exists **fails**; the operation you want updates the existing record's rules. The section value is a closed vocabulary you cannot extend.

If a Content API call returns `403 Permission data not found`, the fix is granting the route to the group — not creating a permission that is already there.

## Admin permission keys

An admin's rights are a map of dotted keys such as `orders.get`, `blocks.preview` or `settings.attributes.create`. The vocabulary is fixed by the platform; a key outside it can never be held by anyone.

- Read the authenticated admin's own map — `cms_whoami` reports it — rather than guessing.
- Grant or revoke individual keys. Never replace the whole map: removing rights the human still needs is easy and reconstructing them is not.
- `orders.export`, `payments.export` and `users.export` are **not grantable** on current instances, so those operations answer 403 permanently. Report that instead of retrying.

→ `mcp/docs/api/admins-and-permissions`

## Modules

Eighteen admin modules are provisioned, addressed by identifier:

`settings`, `forms`, `catalog`, `content`, `admins`, `blocks`, `journal`, `menu`, `users`, `payments`, `events`, `orders`, `workflows`, `collections`, `discounts`, `import-data`, `subscriptions`, `filters`

Module identifiers are **not unique**, so a duplicate create succeeds and produces a shadow module that appears in listings and is wired to nothing.

Module mutations are also on this server's permanently confirm-gated list and need explicit human confirmation regardless of the allow level — precisely because a mistake here is not obvious afterwards.

→ `mcp/docs/api/modules`

## General types

General types classify pages, blocks, products, forms, orders and discounts. Expect to find, addressed by name:

`product`, `product_preview`, `common_page`, `catalog_page`, `error_page`, `external_page`, `common_block`, `product_block`, `similar_products_block`, `form`, `order`, `service`, `discount`, plus one per dynamic block type.

The name **is unique**, so creating one of these fails. Ids are not contiguous — earlier types were removed and their numbers were never reused — and they differ between instances, so **resolve a general type by name, never by a remembered id**.

→ `mcp/docs/api/general-types`

## Attribute set types and field types

Attribute set types say what an attribute set can be attached to. Expect:

`forAdmins`, `forBlocks`, `forOrders`, `forPages`, `forProducts`, `forUsers`, `forForms`, `forUserGroups`, `forEvents`, `forDiscounts`, `system`

Their names are **not unique**, so a duplicate is one of the silent kind. Resolve by name and use what is there.

Attribute field types are a closed list of nineteen — `string`, `text`, `textWithHeader`, `integer`, `real`, `float`, `dateTime`, `date`, `time`, `file`, `image`, `groupOfImages`, `radioButton`, `list`, `spam`, `button`, `entity`, `timeInterval`, `json`. Pick from it; a payload naming a type outside it is rejected.

→ `mcp/docs/api/attribute-types`

## Locales

The full set of locale entries — `en_US`, `ru_RU`, `fr_FR` and 179 more — is provisioned, and only **`en_US` is active** on a fresh instance. Codes are unique, so a create always fails; the operation you want is **activate**.

Read the active codes before writing any localized content, and write every active locale rather than assuming one.

→ `mcp/docs/api/locales`

## Block types and their default templates

Nine dynamic block types are provisioned, each with a Content API endpoint of its own: frequently ordered, slider, trending, recently viewed, repeat purchase, personal recommendations, cart complement, cart similar and wishlist similar.

Each ships a default template whose identifier is the block type name followed by `_default`, for example `trending_block_default`. Template identifiers are unique, so re-creating one fails — and existing blocks already point at it, so a hand-rolled replacement behaves differently from every other block on the instance.

→ `mcp/docs/api/block-types`

## Settings records are singletons

General settings, plan limits, usage counters, discount settings and event e-mail settings each exist as exactly **one** record. There is no meaningful second one.

Update them. A create succeeds silently, leaves two records, and the platform reads only one of them — which makes the change appear to have no effect.

→ `mcp/docs/api/settings`

## Order statuses are globally unique

An order status identifier is unique across the **whole instance**, not per order storage. Two storages cannot both have a status called `new`, so a create that looks obviously safe can fail because another storage claimed the name.

On upgraded instances you will also find payment-axis statuses whose identifier is an existing one with `-payment` appended. Those are provisioned; do not duplicate or rename them.

→ `mcp/docs/api/order-statuses`

## What may be missing on a minimal instance

Starter content is not part of the baseline. On a minimal instance these may simply not exist: pages, products, product statuses, order storages and their statuses, payment accounts, authentication providers, menus, the `user` group, and the default attribute set block templates expect.

If something the human asked you to use is absent, say so and ask — reconstructing a starter project by hand produces something that looks right and behaves differently.

## Every needless record consumes instance quota

Instances enforce a total-record limit across the business entity types — pages, products, blocks, forms, menus, users, permissions, templates, order statuses, modules, groups and more. When it is reached, **all** further creates fail, including ones unrelated to whatever filled it, and an accidental duplicate keeps consuming that budget until someone finds it. This is why "list first, create second" is not merely tidy.

## How to check what your instance already has

Run the reads before the writes. Find the operations with `cms_api_search`, then call them:

```text
cms_api_search { "query": "locales" }
cms_api_search { "query": "general types" }
cms_api_search { "query": "modules", "method": "get" }
cms_api_search { "query": "user groups", "method": "get" }
cms_api_search { "query": "attributes sets types" }
cms_api_search { "query": "templates", "method": "get" }
```

The results of those calls, not this document, are the truth for your instance. Where they disagree, believe the instance and tell the human that the documentation is out of date.
