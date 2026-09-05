# Admins and admin permissions

An admin is an account that operates the admin panel and the Admin API. Its permissions are a map of dotted keys, and this server checks them locally before sending a request.

Admin mutations are permanently confirm-gated, so nothing in this area happens without a human agreeing to it in the conversation.

→ `mcp/docs/server/allow-levels#the-local-permission-pre-check` · `mcp/docs/api/users-and-groups`

## Permissions are dotted keys

Keys look like `menu.update`, `orders.get`, `blocks.preview`, `settings.attributes.create`. The vocabulary is fixed by the platform: a key outside it cannot be held by anyone, and a write that carries one is refused outright.

Each operation declares the key it requires. `cms_api_describe` shows it as `permission`, and `cms_whoami` shows what the current admin holds.

## An unrecognised permission key is refused with 400

A write that carries a key outside the vocabulary is refused with `400`, and **nothing** is stored — not the unrecognised key, and not the valid keys sent beside it. It applies to every write carrying a `permissions` map: creating an admin, and updating one.

The response names every offending key in one go, so a single call finds all the mistakes in a payload rather than one per attempt. A recognised key carrying a value that is neither `true` nor `false` is refused too, in a separate message naming the shape that value should have had. Two keys are not booleans: `admins.modules` takes a list of area identifiers, `pages.scope` a non-empty list of page ids or `false`. `AdminsController_getPermissionsValueTypes` returns exactly those keys with the shape each expects; anything it does not return takes `true` or `false`.

Take the vocabulary from `AdminsController_getAllAvailablePermissionsKeys` and send only keys it returns. A key that reads like the neighbour of a real one is not thereby real: `pages.get` exists, `pages.read` does not.

→ `mcp/docs/api/admin-page-scope#the-value-is-a-list-of-page-ids-not-a-boolean`

This matters most when you edit a map you just read. Change the value you meant to change and send the rest back untouched — a key you invented or mistyped on the way through takes the whole write down with it.

## The local check and what it means

Before sending, this server compares the operation's required key against the admin's map. If it is missing, the call is refused locally:

```text
Admin #12 lacks the "menu.delete" permission required by AdminMenusController_remove.
No request was sent — ask for the grant instead of retrying.
```

Your operation was never sent, so nothing changed. The refusal is final: retrying, changing the body, or reaching for a neighbouring operation with the same requirement all fail identically.

Two things it does not tell you. If the server could not read the admin's permissions at all, the list is empty, the local check is **skipped**, and refusals arrive as 403 from the instance instead — an empty `permissions` in `cms_whoami` means "unknown", not "none". And the check does not know a requirement for every operation: where it knows none it stays quiet and the call goes out, so a call that passed the local check can still answer 403.

Treat a call that was not refused locally as unverified, not as permitted.

→ `mcp/docs/server/allow-levels#the-local-permission-pre-check`

## Asking for a grant

Report it in one message: the exact key, the operation you were trying to run, and what you intended to do. That is everything the human needs to make the change.

Then stop. Granting is their action.

A grant made while your session is open is not picked up until the session reconnects — the permission list is read once.

## Reading admin data needs a permission too

Reads are gated the same way writes are. An account that holds no keys reads nothing, and a missing key answers `403 Forbidden resource` on the read itself.

| What you are reading | Key |
|---|---|
| Pages | `pages.get` |
| Admin accounts | `admins.get` |
| General settings | `settings.general.get` |
| Menus | `menu.get` |
| Users | `users.get` |
| Journal entries and session traffic | `journal.get` |
| Attribute sets and their attributes | `settings.attributesSets.get` |
| Form submissions and their counts | `forms.data.read` |

One key covers every read of that data rather than a single route: the listing, reading one by id, and the search and pagination forms beside them all require it. So a `403` on one of them will not be worked around by reaching for a neighbour.

Helpers stay open, because a write grant has to be usable on its own. Checking whether a login, an email, a marker or a page URL is already taken, reading the list of recognised permission keys, and reading the catalogue of attribute-set types all work without a read key.

## A key can exist without any admin holding it

The vocabulary is fixed at any moment but it is not frozen: keys are added to the platform over time, and an admin provisioned before a key existed does not hold it. Nobody is granted it retroactively.

So an operation that worked for months can begin answering `403` while every neighbouring operation still succeeds — the account did not lose anything, the operation gained a requirement. `forms.data.read`, `pages.get`, `menu.get`, `journal.get`, `settings.general.get`, `settings.attributesSets.get`, `menu.delete`, `journal.delete`, `files.create` and `files.delete` are keys where this is the usual explanation.

`AdminsController_getAllAvailablePermissionsKeys` returns every key the instance recognises. Compare it against the admin's own map before concluding anything: a key in that list and absent from the map is a grant to ask for, and a key in neither is one you have misremembered.

→ `mcp/docs/api/form-submissions#both-read-routes-need-the-forms-data-read-permission` · `mcp/docs/api/files-and-uploads#admin-uploads-and-deletes-need-file-permissions`

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

## Exporting data has its own permissions

`users.export`, `orders.export` and `payments.export` govern the operations under `/export` — everything of a kind, or one auth provider, order storage or payment account at a time.

They are ordinary keys. Each appears in the vocabulary the instance returns, an admin provisioned with the instance holds all three, and an export asked for by a holder runs — the order export answers `200` with a file. Take the key away and the same call answers `403 Forbidden resource` with nothing exported. So handle a refusal here the way you would any other: name the key and ask for the grant. It is not a limitation to report and route around.

Read the key names in that order — the entity first, `export` second. `export.users` is the shape people reach for and it is not in the vocabulary, so a grant asked for under that name cannot be given.

`format` is required and takes `csv` or `xml`. Anything else answers `400` before the export begins, which is the cheapest way to find out you sent `xlsx`.

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
- **Inventing a permission key.** The vocabulary is fixed, and a write carrying an unknown key is refused whole.
- **Replacing a permission map wholesale.** Rights disappear silently.
- **Reporting an export refusal as a platform limitation.** Those keys grant like any other.
- **Expecting a grant to apply mid-session.** Reconnect.
- **Creating an admin to work around a refusal.** Gated, and not the answer.
- **Reading a new `403` as lost rights.** The operation more often gained a requirement.
- **Calling the total logout to fix a stuck session.** It ends everyone else's too.
