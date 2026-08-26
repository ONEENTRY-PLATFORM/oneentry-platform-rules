# Visitor sign-in and the guest cart

What a site does on its account and cart pages: register a visitor, sign them in, and keep a cart for someone who has not signed in at all. All of it is the Content API, so this server cannot make these calls — you are writing the code that will.

Four requirements below are invisible in the schema, and three of them are met with a success answer that stores nothing.

→ `mcp/docs/api/users-and-groups` · `mcp/docs/api/content-api-reads`

## The guest cart needs a UUID in x-guest-id

A cart belonging to nobody is keyed by the `x-guest-id` request header, and the value has to be a **UUID**. Anything else is accepted and kept nowhere:

```text
POST /api/content/users/me/cart/items   x-guest-id: shop-visitor-0001
  → 201 { "items": [ { "productId": 1, "qty": 2 } ], "total": 1 }
GET  /api/content/users/me/cart         x-guest-id: shop-visitor-0001
  → 200 { "items": [], "total": 0 }     ← and so at 2, 10, 30 and 60 seconds
```

The `201` echoes the cart back, so nothing in the response says the item was not kept. A plain UUID works; a UUID with a prefix such as `guest_<uuid>` does not.

Generate one UUID per visitor, store it where the site keeps it between page loads, and send the same value on every cart call.

## The cart lives under users me cart

The path is `/api/content/users/me/cart`, with `/items` beneath it for adding and removing. `/api/content/users/cart` — the address most people try first — answers `404 Cannot GET`.

That `404` is a wrong path and not a missing feature, and it is worth checking before anything else when the cart appears not to exist.

## Sign-up and sign-in do not take the same body

Two operations one path segment apart, with opposite requirements:

| | `sign-up` | `auth` |
|---|---|---|
| `formIdentifier` in the body | optional | rejected as not allowed |
| `langCode` in the body | required | rejected as not allowed |
| `formData` | optional, locale-keyed when sent | not sent |
| `x-device-metadata` header | not needed | required |

So a body that registers a visitor cannot be reused to sign them in, and the error it gets back names the field rather than the difference.

A registration therefore needs no more than the locale and the credentials. Omit `formIdentifier` and the form attached to the auth provider in the URL is used; send it and the value you send is honoured:

```json
{ "langCode": "en_US",
  "authData": [ { "marker": "email", "value": "visitor@your-instance.example" } ] }
```

## Changing the address a visitor signs in with

The profile update takes the sign-in field — the one flagged isLogin — like any other field of the form, and writing it moves the credential: from then on the visitor signs in with the new value and the old one stops working. Send it in `formData` under its own marker, exactly as you would any other field.

The value is stored as sent rather than escaped, and the refusals are the same ones an operator meets in the admin panel. Read them before you build the account page, so the site can tell the visitor which of them it hit.

→ `mcp/docs/api/users-and-groups#the-islogin-field-is-what-a-user-signs-in-with`

## Sign-in needs the x-device-metadata header

`auth` without it answers `400` naming the header and nothing else. The value is a JSON object, the same shape the visitor form route takes:

```json
{ "fingerprint": "web-3f0c2a91-1d44-4a0b-9d2e-6b71f0c2a913",
  "deviceInfo": { "os": "web", "browser": "site", "location": "en_US" } }
```

Registration does not ask for it. The requirement appears at sign-in, which is why it usually surfaces after the account already exists.

Two different `400`s come out of it, and they mean different things: `Missing x-device-metadata header` when it is absent, `Invalid x-device-metadata format` when the value is not JSON, `fingerprint` is not a string, or `deviceInfo` is not an object.

→ `mcp/docs/api/rating-forms-and-reviews#importing-reviews-from-another-system`

## Where the session routes live

There is no `auth` prefix. Signing out and renewing a token live under the **auth provider's marker**, next to sign-in, which is why looking for `/api/content/auth/logout` returns `404` and reads as "sign-out is not supported":

```text
POST .../users-auth-providers/marker/email/users/auth        sign in, returns a token pair
POST .../users-auth-providers/marker/email/users/refresh     exchange a refresh token for a new pair
POST .../users-auth-providers/marker/email/users/logout      sign out on this device
POST .../users-auth-providers/marker/email/users/logout-all  sign out everywhere
GET  .../users-auth-providers/marker/email/users/sessions    list the active devices
```

`email` is the marker of the default password provider; substitute the one you are using.

**Only some of them want `x-device-metadata`.** `auth`, `refresh`, `logout` and OAuth sign-in require it. `logout-all` and `sessions` identify the caller from the bearer token alone and answer normally without it — so a client that sends the header everywhere is not wrong, but one that cannot build it can still sign out of everything.

## What a fingerprint decides

The fingerprint in `x-device-metadata` is the identity of the *device*, and three behaviours follow from it:

- Signing in again with the **same** fingerprint replaces that device's session and leaves other devices alone. Repeat sign-in is therefore always safe and always returns a fresh pair — it does not accumulate sessions.
- Signing in with a **different** fingerprint adds a session beside the first. `sessions` lists them, one entry per device.
- `refresh` must carry the fingerprint the refresh token was issued to.

Closing a session ends its access token **immediately**, before the token's own expiry — a read with it answers `401`. So sign-out is real server-side state, not a client discarding a cookie.

## Signing in with a one-time code instead of a password

An auth provider created with `isCheckCode` set signs a visitor in by code, and its form does not need a password attribute at all. Two calls, both on the same provider marker:

```text
POST /api/content/users-auth-providers/marker/email/users/generate-code
  { "userIdentifier": "visitor@your-instance.example", "eventIdentifier": "auth" }
POST /api/content/users-auth-providers/marker/email/users/auth
  { "authData": [ { "marker": "email", "value": "visitor@your-instance.example" } ],
    "code": "407063" }
```

`eventIdentifier` is not a free label: it must name an **existing send-code event** of the forms module. No such event, or an event carrying that identifier with a different form type, answers `404 Event "<identifier>" with formType "send_code" not found`, and no code is created or destroyed. An empty or absent value is refused earlier, with `400 eventIdentifier should not be empty`.

A success there means the event was found — not that anything reached the visitor. Delivery still depends on the account having the notification contact filled in.

→ `mcp/docs/api/events#configuring-a-form-submitted-email-event`

The code is delivered to the visitor through the provider's notification channel, never in the response. `auth` answers with the same token pair the password path returns, and a code works once — a second attempt with it answers `401`. A code may also be passed as an `authData` entry whose marker is `code`; on a provider with `isCheckCode` a non-empty entry of that name switches the request to the code path, so do not leave a stale one in the body of a password sign-in.

Registration through such a provider issues the first code by itself, so ask for one only when the visitor needs a new one.

The same code, asked for the same way, is what a forgotten-password reset takes — on every provider, whatever its activation setting says.

→ `mcp/docs/api/password-reset`

## Two limits guard the code path

Both answer `429`, and both are per visitor rather than per address:

- **Asking for another code too soon.** A code can be reissued about half a minute after the previous one was sent; inside that window the call is refused and the code already sent stays valid. Show the visitor a countdown rather than retrying.
- **Getting the code wrong five times.** The fifth wrong code destroys the code being guessed, so the correct value stops working too, and the visitor has to request a new one. The count is shared by every call that checks a code — sign-in, code check, activation and password change alike.

## notificationData carries the email address only

On registration, `notificationData` takes `email` and nothing beside it. The phone fields are refused as a string and as an array alike, so there is no value that gets them through — leave them out:

```json
{ "notificationData": { "email": "visitor@your-instance.example" } }
```

Sending `email` as a list is refused too. It is one string.

## Checking that the whole chain works

Register, sign in, keep the `accessToken`, and read the visitor back — the profile comes back carrying the form fields the registration wrote. For the cart: add an item, read the cart, remove it, read again.

Run the cart half **twice** with the same guest id, in two separate requests. One request that adds and reads is not a check: the answer to the add is an echo, and only a later read proves the cart survived.

→ `mcp/docs/api/verification-recipes#the-general-pattern`

## Common mistakes

- **A guest id that is not a UUID.** Every add answers `201` and the cart stays empty.
- **Treating that `201` as proof.** Read the cart back in a second request.
- **Looking for the cart under `users/cart`.** It is `users/me/cart`.
- **Reusing the registration body for sign-in.** Two of its fields are rejected there.
- **Omitting `x-device-metadata` at sign-in.** It answers `400` naming the header.
- **Requesting a fresh code on every click.** Inside the reissue window it answers `429`; the code already sent is the working one.
- **Putting phone fields into `notificationData`.** Only `email` is accepted.

→ `mcp/docs/api/forms-and-form-data` · `mcp/docs/api/content-api-reads`
