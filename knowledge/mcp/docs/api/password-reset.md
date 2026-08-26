# Resetting a visitor password

How a site lets a visitor who cannot sign in choose a new one. The reset is anonymous — it carries no session — and a one-time code is what proves the account belongs to whoever is asking.

Read this before wiring a forgot-password screen, and before concluding from a provider's settings that the code can be skipped.

→ `mcp/docs/api/content-api-sign-in-and-cart#signing-in-with-a-one-time-code-instead-of-a-password` · `mcp/docs/api/events#configuring-a-form-submitted-email-event`

## The code is required on every provider

`code` is not an activation code and no provider setting turns it off. The provider option that activates new accounts by code governs registration only; a provider with it switched off still refuses a password change that carries no code, and still issues codes on request.

That is not a gap to work around. The route takes no token and identifies the account by nothing but a name the visitor types, so the code is the only evidence the account is theirs. There is no operation that changes a password without one, and none that changes the password of the signed-in visitor by checking the old one.

## Two calls in order

Ask for the code, then send it back with the new password. Both calls go to the same provider marker, and both are reached with the site's app token — no admin session anywhere in this flow.

```text
POST /api/content/users-auth-providers/marker/email/users/generate-code
  { "userIdentifier": "visitor@your-instance.example", "eventIdentifier": "reset" }

POST /api/content/users-auth-providers/marker/email/users/change-password
  { "userIdentifier": "visitor@your-instance.example", "eventIdentifier": "reset",
    "code": "407063", "password1": "chosen-secret", "password2": "chosen-secret",
    "type": 1 }
```

`eventIdentifier` must name an existing send-code event on the first call — that is what delivers the code, and a name that matches none answers `404`. On the second call the same field names the event that notifies the visitor that the password changed; nothing is checked there, so a name that matches nothing still changes the password and only the notification is lost.

`password1` and `password2` must be equal and non-empty. Include `type` with the value `1`: an instance that still requires it answers `404` when it is absent, which reads as if the route or the account were missing.

## What each refusal means

| status | trigger | fix |
|---|---|---|
| `400` | the code does not match a live code of that account, or is absent, empty or expired | ask for a fresh code and send it unchanged |
| `400` | no account with that identifier under this provider | check the identifier and the provider marker together |
| `400` | the two passwords differ, or one is empty | send them equal |
| `404` | the marker names no active provider | list the providers and take the marker from there |
| `405` | the instance's plan does not include this operation | nothing in the body will change it |
| `429` | too many wrong codes, or a new code asked for too soon | see the two limits on the code path |

A wrong code counts against the same attempt limit sign-in uses, and the count is per visitor. Exhaust it and the code being guessed is destroyed, so the correct value stops working too.

→ `mcp/docs/api/content-api-sign-in-and-cart#two-limits-guard-the-code-path`

## The answer is the boolean true

A successful reset answers the bare JSON value `true`, with `201` rather than the `200` the schema shows. Read the body, not the status digit — code written against `=== 200` treats a completed reset as a failure and offers the visitor a retry that then fails on the spent code.

## A reset signs the visitor out everywhere

Every session that account holds on that provider is dropped as part of the change. A token obtained before the reset answers `401` afterwards, on any route that needs one.

So a site that resets a password mid-visit must clear whatever it cached and send the visitor through sign-in again with the new password. Do not assume the tokens you were holding still work, and do not treat that `401` as a sign the reset failed — it is the proof that it succeeded.

→ `mcp/docs/api/content-api-sign-in-and-cart#where-the-session-routes-live`

## Common mistakes

- **Expecting a provider flag to remove the code requirement.** It governs registration; the reset always takes a code.
- **Calling the change without asking for a code first.** There is nothing to send, and the answer is `400`.
- **Reading the `404` from the first call as no such user.** It means no send-code event carries that identifier.
- **Leaving `type` out.** Send `1`.
- **Keying on `200`.** The answer is `201` with `true` in the body.
- **Reusing a token held from before the reset.** Every session is gone; sign in again.
- **Sending the code the visitor pasted with spaces around it.** It is compared as it arrives.

→ `mcp/docs/api/verification-recipes`
