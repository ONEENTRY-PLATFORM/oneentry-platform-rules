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
| `formIdentifier` in the body | required | rejected as not allowed |
| `langCode` in the body | required | rejected as not allowed |
| `formData` | locale-keyed, `{ "en_US": [ … ] }` | not sent |
| `x-device-metadata` header | not needed | required |

So a body that registers a visitor cannot be reused to sign them in, and the error it gets back names the field rather than the difference.

## Sign-in needs the x-device-metadata header

`auth` without it answers `400` naming the header and nothing else. The value is a JSON object, the same shape the visitor form route takes:

```json
{ "fingerprint": "web-3f0c2a91-1d44-4a0b-9d2e-6b71f0c2a913",
  "deviceInfo": { "os": "web", "browser": "site", "location": "en_US" } }
```

Registration does not ask for it. The requirement appears at sign-in, which is why it usually surfaces after the account already exists.

→ `mcp/docs/api/rating-forms-and-reviews#importing-reviews-from-another-system`

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
- **Putting phone fields into `notificationData`.** Only `email` is accepted.

→ `mcp/docs/api/forms-and-form-data` · `mcp/docs/api/content-api-reads`
