# Admins and admin permissions

An admin is an account that operates the admin panel and the Admin API. Its permissions are a map of dotted keys, and this server checks them locally before sending a request.

Admin mutations are permanently confirm-gated, so nothing in this area happens without a human agreeing to it in the conversation.

→ `mcp/docs/server/allow-levels#the-local-permission-pre-check` · `mcp/docs/api/users-and-groups`

## Permissions are dotted keys

Keys look like `menu.update`, `orders.get`, `blocks.preview`, `settings.attributes.create`. The vocabulary is fixed by the platform: a key outside it cannot be held by anyone, and inventing one achieves nothing.

Each operation declares the key it requires. `cms_api_describe` shows it as `permission`, and `cms_whoami` shows what the current admin holds.

## The local check and what it means

Before sending, this server compares the operation's required key against the admin's map. If it is missing, the call is refused locally:

```text
Admin #12 lacks the "menu.delete" permission required by AdminMenusController_remove.
No request was sent — ask for the grant instead of retrying.
```

Nothing was sent, so nothing changed. The refusal is final: retrying, changing the body, or reaching for a neighbouring operation with the same requirement all fail identically.

There is one nuance. If the server could not read the admin's permissions at all, the list is empty, the local check is **skipped**, and refusals arrive as 403 from the instance instead. An empty `permissions` in `cms_whoami` means "unknown", not "none".

## Asking for a grant

Report it in one message: the exact key, the operation you were trying to run, and what you intended to do. That is everything the human needs to make the change.

Then stop. Granting is their action.

A grant made while your session is open is not picked up until the session reconnects — the permission list is read once.

## Some permissions cannot be granted

`orders.export`, `payments.export` and `users.export` are not grantable on current instances. Operations requiring them answer 403 permanently, no matter who is asking.

Report that as a platform limitation rather than a configuration problem, and do not ask a human to grant them — they cannot.

## Never replace an admins permission map

Grant or revoke individual keys. Replacing the whole map is how rights that nobody noticed were needed disappear, and reconstructing it afterwards means guessing.

If a human asks you to "give this admin the same rights as that one", read both maps, list the differences, and let them confirm each addition.

## Admin mutations are permanently gated

Every mutating operation under the admins area requires a confirm token at every allow level. So does everything under platform settings, modules, backups and payment webhooks.

The reasoning is direct: these are the accounts and switches that could undo or hide any other change. A dry run gives you the target; show it, wait for agreement, then confirm.

→ `mcp/docs/server/confirm-and-dry-run`

## Developer accounts

Accounts flagged as developer accounts belong to a different API surface, and the Admin API rejects them with a 401. If credentials that work elsewhere fail here, that is the first thing to check.

→ `mcp/docs/server/authentication#developer-accounts-are-rejected`

## Admins are not user groups

Admin permissions govern the Admin API. Content API access is governed by user groups and their per-route permissions. Granting one never affects the other, and a Content API 403 is never fixed here.

→ `mcp/docs/api/users-and-groups#permissions-are-per-route-and-already-exist`

## Common mistakes

- **Retrying a permission refusal.** It cannot succeed.
- **Inventing a permission key.** The vocabulary is fixed.
- **Replacing a permission map wholesale.** Rights disappear silently.
- **Asking for a grant of an ungrantable export permission.**
- **Expecting a grant to apply mid-session.** Reconnect.
- **Creating an admin to work around a refusal.** Gated, and not the answer.
