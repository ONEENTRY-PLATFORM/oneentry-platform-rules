# Modules

A module is a section of the admin panel — catalogue, orders, forms, users — and the thing general types and attribute set types attach to so entities appear in the right place.

Eighteen are provisioned. Mutating them is permanently confirm-gated, and almost every task that seems to need it does not.

→ `mcp/docs/api/baseline-data#modules` · `mcp/docs/api/general-types`

## The modules you will find

`settings`, `forms`, `catalog`, `content`, `admins`, `blocks`, `journal`, `menu`, `users`, `payments`, `events`, `orders`, `workflows`, `collections`, `discounts`, `import-data`, `subscriptions`, `filters`

Read them rather than trusting the list:

```text
cms_api_search { "query": "modules", "method": "get" }
```

## Address them by identifier

The identifier is the stable handle. Ids are local to the instance, and while these particular ones tend to be stable, nothing guarantees it.

Identifiers here are **not unique**, which is the dangerous part: creating a second module called `catalog` succeeds, produces a shadow module wired to nothing, and everything keyed off the real one carries on ignoring it.

## Why mutations are gated

Modules are the wiring the admin panel hangs off. General types attach to them; attribute set types attach to them; visibility of whole sections follows from them.

A change here does not break one entity, it changes what an operator can see and reach. That is why every mutation needs a confirm token at every allow level, and why the right response to "the module configuration needs changing" is usually to ask why.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Module visibility

A module can be hidden, which removes its section from the admin panel without deleting anything. Content of that kind still exists and is still reachable through the API.

So "the orders section is missing" may be visibility rather than data, and hiding a module is a reversible way to simplify a panel — unlike deleting anything inside it.

## Modules and permissions

What an admin can do inside a module is governed by permission keys, not by the module record. Hiding a module does not remove rights, and granting rights does not reveal a hidden module.

Diagnose the two separately.

→ `mcp/docs/api/admins-and-permissions`

## When a task really involves modules

Rarely. The legitimate cases are inspection — reading which modules exist, to resolve a general type's placement or explain why a section is absent — and, occasionally, visibility changes an operator explicitly asks for.

Creating a module is not how a new content kind is introduced; general types and attribute sets are.

## If you think you need to change one

1. Say what you believe the change achieves.
2. Read the module and show the human its current state.
3. Dry run and show the target.
4. Wait for explicit agreement, then confirm.

If the goal is "make this content appear in the admin panel", the answer is almost certainly a general type or an attribute set type, not a module.

## Common mistakes

- **Creating a module.** Identifiers are not unique; you get a shadow.
- **Changing module wiring to fix a missing entity.** Look at the general type instead.
- **Confusing visibility with permissions.** Two independent mechanisms.
- **Treating the gate as a formality.** It exists because these changes are wide.
