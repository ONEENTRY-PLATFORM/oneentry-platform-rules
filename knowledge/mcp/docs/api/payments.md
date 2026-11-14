# Payments

Payment accounts connect an instance to a payment provider; payment sessions are individual attempts to take money. The order's payment state follows from those sessions rather than being written by hand.

This is the most consequential area of the API. Webhook paths are permanently confirm-gated, and refunds are not reversible.

→ `mcp/docs/api/orders` · `mcp/docs/api/order-statuses`

## Accounts and providers

An account holds the configuration for one provider on this instance: its credentials, its mode, and how it should behave. The account type says which provider it is; several are supported, and one type exists for custom arrangements handled outside any provider.

**No payment accounts are provisioned.** Every account on an instance was created by someone, so an instance may legitimately have none.

## Why a second account of the same provider is refused

Every provider type can be connected once per instance. Creating an account of a type that already has one, or changing an existing account to such a type, answers `400`. Only the `custom` type can repeat. To switch the provider's credentials or mode, update the account you have; do not add another.

An account's `identifier` is unique too. A create or update that reuses one held by another account answers `409`. Check a candidate first with `GET /api/admin/payments/accounts/marker-validation/{marker}` — it answers `valid: false` when the identifier is taken. Keeping an account's own identifier in an update is fine.

## Never read or write credentials casually

An account's configuration contains provider credentials. Two rules:

- Do not print account configuration into a conversation, a report or a commit. If you must show that an account exists, show its type and its name.
- Do not create or modify an account on a human's behalf without them supplying the credentials in that moment and asking you to.

## Why a provider key reads back as asterisks

A provider credential that is set — `secretKey`, `signingSecretKey`, `serverKey`, `apiKey`, `callbackToken`, in `settings` or `testSettings` — reads back as `********` from every account call: list, read, create and update. An empty value stays empty, so you can still tell a key that is not set. The `status` of an account is worked out from the real keys, so a masked key still reports `connected`.

On update, `********`, an empty string or a credential left out of the body all keep the stored key. Only a new value replaces it, and `null` removes it.

Through this server a set key may show as the redaction placeholder instead of `********`; both mean the key is set. Leave the field out of the update to keep it.

→ `mcp/docs/server/response-shaping#credentials-are-redacted`

## Sessions are the lifecycle

A payment session represents one attempt: created, then moving through the provider's flow to a terminal state. The order's payment axis reflects where the session got to.

You read sessions. You do not advance them by writing statuses — the provider's callbacks do that, and a hand-written status that contradicts the session is worse than none.

The one exception is an order whose payment account is of type `custom`, where there is no provider and no callback to wait for: the payment axis on such an order **is** written by hand, because nothing else will write it. For every other account type the write is refused or lands on a different axis, and the next link covers which.

→ `mcp/docs/api/order-statuses#the-payment-axis-is-managed-for-you`

## Why webhook paths are gated

Provider callbacks arrive on webhook paths, and those paths are how the platform learns that money moved. Mutating anything under them is permanently confirm-gated at every allow level, regardless of configuration.

Reading is unaffected. If a task seems to require changing webhook configuration, stop and hand it to a human — this is infrastructure, not content.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Refunds

Two distinct things:

- a **refund** performed against a payment;
- a **refund request** raised by a customer, which an operator then acts on.

Reading requests is routine work. Issuing a refund is not: it moves money, it cannot be undone through the API, and a partial refund issued twice is a real loss.

Never issue one without explicit agreement in the conversation, quoting the order, the payment and the amount. Dry run first and show the target.

## Diagnosing a payment that looks stuck

1. Read the order and note **all** its axes — the lifecycle may have moved while payment did not.
2. Read the payment session and see which state it reached.
3. Check whether the provider is configured on this instance at all.
4. Check the timing: a session that completed seconds ago may not be reflected in listings yet.

If the session is terminal and the order's payment axis disagrees, that is a platform-side issue to report, not something to fix by writing the status.

## Split payments and staged charges

Where an instance takes payment in stages, each stage is its own session against the same order. Reading only the first one gives a misleading picture of what has been collected.

Read all the sessions for an order before reporting what a customer has paid.

## Common mistakes

- **Writing an order's payment status.** It is derived.
- **Exposing account configuration.** It contains credentials.
- **Assuming a provider is configured.** Accounts are not baseline data.
- **Issuing a refund without explicit agreement.** Irreversible.
- **Reading one session and calling it the total paid.** Stages exist.

→ `mcp/docs/api/verification-recipes#orders`
