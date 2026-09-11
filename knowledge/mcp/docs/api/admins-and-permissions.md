# Admins and admin permissions

An admin is an account that operates the admin panel and the Admin API. Its permissions are a map of dotted keys, and this server checks them locally before sending a request.

Admin mutations are permanently confirm-gated, so nothing in this area happens without a human agreeing to it in the conversation.

→ `mcp/docs/server/allow-levels#the-local-permission-pre-check` · `mcp/docs/api/users-and-groups`

## Permissions are dotted keys

Keys look like `menu.update`, `orders.get`, `settings.attributes.create`. The vocabulary is fixed by the platform: a key outside it cannot be held by anyone.

Each operation declares the key it requires. `cms_api_describe` shows it as `permission`, and `cms_whoami` shows what the current admin holds.

## An unrecognised permission key is refused with 400

A write that carries a key outside the vocabulary is refused with `400`, and **nothing** is stored — not the unrecognised key, and not the valid keys sent beside it. It applies to every write carrying a `permissions` map — creating an admin, updating one.

The response names every offending key in one go, so one call finds every mistake in the payload.

Take the vocabulary from `AdminsController_getAllAvailablePermissionsKeys` and send only keys it returns. A key that reads like the neighbour of a real one is not thereby real: `pages.get` exists, `pages.read` does not.

It matters most when you edit a map you just read: change the value you meant to change and send the rest back untouched.

## A recognised key with the wrong value is a different 400

The two refusals are told apart by their wording, not their status. One says the key is outside the vocabulary; the other says the key is real and the value is not the shape it takes, and names that shape. An unknown key is a name to fix; an invalid value is a value to fix on a grant that does exist. Either way nothing is stored, the valid keys sent beside the offending one included.

Most keys take `true` or `false`. Two do not: `admins.modules` takes a list of area identifiers, and `pages.scope` a non-empty list of page ids, or `false` for no restriction.

`AdminsController_getPermissionsValueTypes` names those keys and the shape each expects. Call it before building a map out of the vocabulary — the key list alone does not say which keys are not flags.

The shapes are published, not enforced alike. `pages.scope` is refused when the value is the wrong shape; `admins.modules` stores whatever it is sent, `true` included. Read the map back after writing it.

→ `mcp/docs/api/admin-page-scope#the-value-is-a-list-of-page-ids-not-a-boolean`

## Creating an admin can answer 405

An instance allows only so many admin accounts. Once it holds that many, `AdminsController_create` answers `405` and creates nothing. The check runs before the permissions map is read, so the status says nothing about the body: rewriting it, dropping keys or retrying answer the same.

Free a seat by deleting an account nobody uses, or tell the human the instance needs a larger allowance. Until then, delegate by editing an account that already exists.

## The local check and what it means

Before sending, this server compares the operation's required key against the admin's map. If it is missing, the call is refused locally:

```text
Admin #12 lacks the "menu.delete" permission required by AdminMenusController_remove.
No request was sent — ask for the grant instead of retrying.
```

Your operation was never sent, so nothing changed. The refusal is final: retrying, changing the body, or reaching for a neighbouring operation with the same requirement all fail identically.

Two things it does not tell you. If the server could not read the admin's permissions, the list is empty, the check is **skipped**, and refusals arrive as `403` from the instance instead — an empty `permissions` in `cms_whoami` means "unknown", not "none". And it knows a requirement for only some operations: where it knows none the call goes out, so a call that passed can still answer `403`.

Treat a call that was not refused locally as unverified, not as permitted.

→ `mcp/docs/server/allow-levels#the-local-permission-pre-check`

## Asking for a grant

Report it in one message: the exact key, the operation, and what you intended to do.

Then stop. Granting is their action.

A grant made while your session is open is not picked up until the session reconnects — the permission list is read once.

## Reading admin data needs a permission too

Reads are gated the same way writes are. An account that holds no keys reads nothing, and a missing key answers `403 Forbidden resource` on the read itself.

| What you are reading | Key |
|---|---|
| Pages | `pages.get` |
| Admin accounts | `admins.get` |
| Menus | `menu.get` |
| Users | `users.get` |
| Journal entries and session traffic | `journal.get` |
| One attribute set, and the attributes inside it | `settings.attributesSets.get` |
| Form submissions and their counts | `forms.data.read` |

One key covers every read of that data rather than a single route: the listing, reading one by id, and the search and pagination forms beside it all require it. A `403` on one is not worked around by reaching for a neighbour.

Helpers stay open. Checking whether a login, an email, a marker or a page URL is already taken, and reading the permission vocabulary, work without a read key. So do the two reads the admin panel issues on every load: `GET /settings-general` and the attribute-set listing `GET /attributes-sets` answer for any signed-in admin, and `401` with no token.

## A key can exist without any admin holding it

The vocabulary is fixed at any moment but it is not frozen: keys are added to the platform over time, and an admin provisioned before a key existed does not hold it. Nobody is granted it retroactively.

So an operation that worked for months can begin answering `403` while every neighbouring operation still succeeds — the account did not lose anything, the operation gained a requirement. `forms.data.read`, `pages.get`, `menu.get`, `journal.get`, `menu.delete` and `files.create` are keys where this is the usual explanation.

`AdminsController_getAllAvailablePermissionsKeys` returns every key the instance recognises. Compare it against the admin's own map: a key in that list but not in the map is a grant to ask for; a key in neither is one you have misremembered.

→ `mcp/docs/api/form-submissions#both-read-routes-need-the-forms-data-read-permission` · `mcp/docs/api/files-and-uploads#admin-uploads-and-deletes-need-file-permissions`

## Signing every admin out has its own permission

`AuthController_logoutAllUsers` — `POST /auth/logout/all-users` — is gated by `admins.totalLogout`. Called with no token it answers `401`; called by an admin without the grant it answers `403 Forbidden resource`, and nothing is signed out.

The operation carries **no permission in the catalog**, so the local pre-check cannot stop the call: this refusal arrives from the instance after the request was sent.

Do not confuse it with the visitor route. Signing out one *customer* everywhere is a Content API concern on a different path.

| To sign out | Use |
|---|---|
| admins, across the instance | `POST /auth/logout/all-users` |
| one admin, while updating them | `AdminsController_update` with `isLogoutAccounts: true` |
| one visitor, on every device | `POST .../users-auth-providers/marker/{marker}/users/logout-all` |

The middle one is a field on the ordinary admin update, not a route: sending it ends that admin's sessions as part of the save. It is ignored when the admin being updated is the one you are signed in as, so it cannot end your own session — but it is still a side effect of a call whose subject is something else, so leave it out unless signing that person out is part of what you were asked to do.

Treat it as destructive: it ends sessions belonging to people who did not ask for it. Show the human what you are about to do and wait.

→ `mcp/docs/api/content-api-sign-in-and-cart#where-the-session-routes-live`

## Exporting data has its own permissions

`users.export`, `orders.export` and `payments.export` govern the operations under `/export` — everything of a kind, or one auth provider, order storage or payment account at a time.

They are ordinary keys: each appears in the vocabulary, an admin provisioned with the instance holds all three, and an export asked for by a holder runs. Without the key the call answers `403 Forbidden resource` and nothing is exported, so handle it as any other refusal — name the key and ask for the grant.

Read the key names in that order — the entity first, `export` second. `export.users` is the shape people reach for; it is not in the vocabulary, so a grant under that name cannot be given.

`format` is required and takes `csv` or `xml`; anything else answers `400` before the export begins.

## Never replace an admins permission map

Grant or revoke individual keys. Replacing the whole map is how rights nobody noticed were needed disappear, and reconstructing it afterwards means guessing.

If a human asks you to "give this admin the same rights as that one", read both maps, list the differences, and let them confirm each addition.

## Admin mutations are permanently gated

Every mutating operation under the admins area requires a confirm token at every allow level. So does everything under platform settings, modules, backups and payment webhooks.

A dry run gives you the target; show it, wait for agreement, then confirm.

→ `mcp/docs/server/confirm-and-dry-run`

## Developer accounts

Accounts flagged as developer accounts belong to a different API surface, and the Admin API rejects them with `401`. If credentials that work elsewhere fail here, check that first.

→ `mcp/docs/server/authentication#developer-accounts-are-rejected`

## Admins are not user groups

Admin permissions govern the Admin API. Content API access is governed by user groups and their per-route permissions. Granting one never affects the other, and a Content API `403` is never fixed here.

→ `mcp/docs/api/users-and-groups#permissions-are-per-route-and-already-exist`

## Common mistakes

- **Retrying a permission refusal.** It cannot succeed.
- **Inventing a permission key.** The vocabulary is fixed, and a write carrying an unknown key is refused whole.
- **Replacing a permission map wholesale.** Rights disappear silently.
- **Reporting an export refusal as a platform limitation.** Those keys grant like any other.
- **Expecting a grant to apply mid-session.** Reconnect.
- **Creating an admin to work around a refusal.** Gated, capped, not the answer.
- **Reading a new `403` as lost rights.** The operation more often gained a requirement.
- **Calling the total logout to fix a stuck session.** It ends everyone else's too.
