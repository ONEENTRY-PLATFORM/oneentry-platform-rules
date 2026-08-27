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

## A key can exist without any admin holding it

The vocabulary is fixed at any moment but it is not frozen: keys are added to the platform over time, and an admin provisioned before a key existed does not hold it. Nobody is granted it retroactively.

So an operation that worked for months can begin answering `403` while every neighbouring operation still succeeds — the account did not lose anything, the operation gained a requirement. `forms.data.read` and `journal.delete` are two keys where this is the usual explanation.

`AdminsController_getAllAvailablePermissionsKeys` returns every key the instance recognises. Compare it against the admin's own map before concluding anything: a key in that list and absent from the map is a grant to ask for, and a key in neither is one you have misremembered.

→ `mcp/docs/api/form-submissions#both-read-routes-need-the-forms-data-read-permission`

## Signing every admin out has its own permission

`AuthController_logoutAllUsers` — `POST /auth/logout/all-users` — is gated by `admins.totalLogout`. Called with no token it answers `401`; called by an admin without the grant it answers `403 Forbidden resource`, and nothing is signed out.

The operation carries **no permission in the catalog**, so the local pre-check has nothing to compare and cannot stop the call. Unlike the refusals above, this one arrives from the instance after the request was sent.

Do not confuse it with the visitor route. Signing out one *customer* everywhere is a Content API concern on a different path, and the two are never substitutes for each other.

| To sign out | Use |
|---|---|
| admins, across the instance | `POST /auth/logout/all-users` |
| one visitor, on every device | `POST .../users-auth-providers/marker/{marker}/users/logout-all` |

Treat it as destructive whichever way it is pointed: it ends sessions belonging to people who did not ask for it. Show the human what you are about to do and wait.

→ `mcp/docs/api/content-api-sign-in-and-cart#where-the-session-routes-live`

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
- **Reading a new `403` as lost rights.** The operation more often gained a requirement.
- **Calling the total logout to fix a stuck session.** It ends everyone else's too.
