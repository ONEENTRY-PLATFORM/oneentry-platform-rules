# Admin account owner and credentials

Every instance has one project owner among its administrators, and changes to another administrator's sign-in details or permissions follow rules the instance enforces on every call, whichever client sends it.

Read this before editing another administrator's login, e-mail, password or permissions, and before reading a `403` on an admin write as a missing grant.

→ `mcp/docs/api/admins-and-permissions#never-replace-an-admins-permission-map` · `mcp/docs/api/admin-sign-in`

## Which administrator is the project owner

An admin account carries `isOwner`, and exactly one account on an instance has it set to `true`. `GET /admins/me` and the admin reads return it beside the other account fields. It is read-only: an `isOwner` sent in a create or update body is ignored, and so is any attempt to set it on another account.

Check `isOwner` in `GET /admins/me` before planning an admin change. Several of the refusals below depend on whether you are the owner, not on the keys you hold.

## Nobody but the owner changes the owner account

Another administrator cannot update the owner's account, change its permissions or delete it. `AdminsController_update`, `AdminsController_updatePermissions` and `AdminsController_remove` on the owner's id answer `403` for anyone else, whatever keys they hold, and nothing is changed.

The owner edits its own account through the same operations. Tell the human that only the owner can make the change; there is no grant that lets you do it.

## Changing another admin login or e-mail

Editing another administrator's profile fields needs `admins.update`. Changing that administrator's `login` or `email` additionally needs `admins.manageCredentials`. Without it the update answers `403` and nothing in the body is stored; a body that repeats the stored login and e-mail unchanged is accepted.

`admins.manageCredentials` is an ordinary key in the vocabulary, but no account holds it until the owner grants it, including accounts provisioned with the instance. Changing a login signs the target administrator out of every session.

## No operation sets another admin password

A `password` in an update of another administrator's account answers `403`, and the password is not changed. There is no way to choose a password for someone else.

To give another administrator a way back in, start the ordinary reset for their address: `POST /auth/send-code` with `{"email": "..."}` mails them a code, and they set the new password themselves. Completing a reset signs them out of every session.

## Changing your own password needs the current one

An update of your own account that carries `password` must also carry `currentPassword`. Missing or wrong, the update answers `403` and the password stays as it was. With the right value the new password applies and your session continues.

```json
{ "login": "editor", "email": "editor@your-instance.example",
  "password": "new-secret", "currentPassword": "old-secret" }
```

## A new e-mail waits for confirmation

Changing `email` — your own or, with `admins.manageCredentials`, someone else's — does not change the address at once. The update answers `200`, the account keeps its old `email`, and the new address appears in `pendingEmail` until it is confirmed. A code is mailed to the new address, and the old address is told about the request.

Confirm with `POST /auth/confirm-email` and `{"email": "<new address>", "code": 12345678}`. It takes no token. It answers `201` and `true`, the new address replaces the old one, `pendingEmail` becomes `null`, and the administrator is signed out of every session. A wrong code answers `400` and changes nothing; repeated wrong codes answer `429` with `Retry-After`.

Do not report an e-mail change as done after the update alone. Read `pendingEmail` back and tell the human a code is waiting in the new mailbox.

## Who may change permissions

`admins.updatePermissions` lets you change another administrator's permission map, within three limits. Each answers `403` and stores nothing:

- **Your own map.** Only the owner may change its own permissions.
- **Keys you do not hold.** You can set a key to `true` only if your own map holds it as `true`; for `admins.modules` you can add only areas your own list contains. Revoking a key is always allowed, and keys sent unchanged are not checked.
- **Owner-only keys.** Only the owner grants `admins.updatePermissions` and `admins.manageCredentials`.

The same limits apply to the `permissions` map of `AdminsController_create`: you cannot create an account holding more than you hold. When the human asks for a grant you cannot give, name the key and say the owner has to give it.

## Admin changes sign the target out

The target administrator is signed out of every session after another administrator changes its permissions or login, after its new e-mail is confirmed, and after its password reset completes. The access token they hold answers `401` on its next request, renewal answers `401` too, and they sign in again. The administrator is mailed a notice of the change.

The owner changing its own permissions keeps its session. A whole-account update with the stored login, e-mail and map unchanged signs nobody out.

## Common mistakes

- **Sending a password for another administrator.** Always `403`; start a reset instead.
- **Reporting an e-mail change as applied.** It waits in `pendingEmail` until the code is confirmed.
- **Asking for `admins.manageCredentials` from anyone but the owner.** Only the owner can grant it.
- **Granting a key you do not hold.** Refused whole; ask the owner.
- **Retrying a write on the owner account.** It cannot succeed for anyone else.
