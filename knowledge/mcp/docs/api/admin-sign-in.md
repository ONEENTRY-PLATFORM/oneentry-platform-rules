# Administrator sign-in and the second factor

How `POST /api/admin/auth/login` answers, and what changes when the instance requires a second factor: sign-in becomes two calls, the first one carries no tokens, and a six-digit code arrives by e-mail.

Read this when a sign-in that used to return tokens returns something else, when a code is refused, or before turning the second factor on for an instance that automation signs in to.

→ `mcp/docs/api/settings#general-settings` · `mcp/docs/server/authentication#two-ways-to-authenticate`

## Sign in answers a challenge instead of tokens

With the second factor off, `POST /api/admin/auth/login` with `login` and `password` answers `201` with a token pair and no `mfaRequired` key. That is the only shape most instances ever produce.

With `user.adminMfaRequired` on and the password correct, the same call answers `201` with a challenge and **no** `accessToken`:

```http
POST /api/admin/auth/login
{ "login": "automation", "password": "…" }

{ "mfaRequired": true,
  "mfaToken": "a1b2c3d4…f9e8",
  "method": "email",
  "expiresIn": 600 }
```

`mfaToken` is an opaque string that identifies this attempt, `method` is `email`, and `expiresIn` is the seconds the code stays valid. A six-digit code is sent to the e-mail address on the administrator's account.

Branch on the presence of `mfaRequired`, never on the setting. A wrong login or password still answers `401 Login or password incorrect`, whichever way the setting is set.

## Finish the sign-in with the e-mailed code

`POST /api/admin/auth/login/mfa` with the `mfaToken` from the challenge and the code from the e-mail answers `201` with the ordinary token pair:

```http
POST /api/admin/auth/login/mfa
{ "mfaToken": "a1b2c3d4…f9e8", "code": "123456" }

{ "userId": 7, "accessToken": "…", "refreshToken": "…" }
```

From there the tokens behave like any others — one session per admin account, and a later sign-in with the same account revokes them.

Two refusals, and they mean different things:

- `400` — `code` is not six digits, or `mfaToken` is missing. The attempt is still usable; send a well-formed body.
- `401 Invalid or expired sign-in code` — the code is wrong, the attempt has expired, the code was already spent, or the `mfaToken` is unknown. This one is terminal for that `mfaToken`.

A code works once. Asking the same `mfaToken` twice with the code that just worked is a `401`, not a second token pair.

## Getting a new code

There is no resend route. Call `POST /api/admin/auth/login` again with the same login and password: it answers a fresh challenge with a new `mfaToken`, and a new code is sent. The previous `mfaToken` is not usable afterwards.

Never loop on this to "make the code arrive". Each sign-in counts against the sign-in limit, and the codes do not queue — only the newest one works.

## When the code cannot be sent

If the instance cannot send the message, the sign-in call answers `503 Could not send the sign-in code` and no challenge comes back. Nothing is wrong with the login or the password.

This is an instance-side mail problem, not something a caller can retry its way out of. Report it; an operator has to fix outbound mail before anyone can sign in. An instance whose mail does not work and whose second factor is on has locked out every account that does not already hold a token.

## Too many attempts answers 429

Repeated calls to `POST /auth/login/mfa` for one account or from one address are limited. Over the limit the call answers `429 Too many sign-in code attempts, retry after the indicated delay` with a `Retry-After` header giving the wait in seconds.

The limit is checked before the code is looked at, so once it trips the call answers `429` **even with the right code in hand**. Wait out `Retry-After`; do not retry in a loop, because every attempt counts. A code cannot be found by trying values.

→ `mcp/docs/server/authentication#why-sign-in-answers-429`

## Turning the second factor on

`user.adminMfaRequired` is a boolean in the `user` section of the general settings, written with the ordinary general settings update under `settings.general.update`. It is off unless someone set it.

Turning it on does not end sessions that are already open — a token issued before the change keeps working to its own expiry. Everything signing in after the change needs a code.

Before turning it on, check two things: that outbound mail works, and that nothing signs in with a login and a password unattended. Through this server, settings paths are permanently confirm-gated, so the write needs a confirm token from a dry run either way.

## What this means for unattended sign-in

Anything that signs in with a login and a password and has no inbox cannot finish a two-step sign-in. This server is in that category: configured with `ONEENTRY_CMS_LOGIN` and `ONEENTRY_CMS_PASSWORD` against an instance that requires a second factor, it gets a challenge where it expects tokens and starts unauthenticated.

Give it `ONEENTRY_CMS_TOKEN` instead — a token obtained by completing the two steps by hand. The failure and the fix are spelled out in `mcp/docs/server/authentication#two-ways-to-authenticate`.

→ `mcp/docs/server/configuration#environment-variables` · `mcp/docs/api/password-reset` · `mcp/docs/api/admins-and-permissions#personal-settings-live-on-the-admin-account`
