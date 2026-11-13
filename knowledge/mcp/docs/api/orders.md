# Orders

An order records what a customer bought, at what price, in what state, and how it was paid for. Orders live inside an **order storage** — a container that defines the statuses and rules an order follows.

Orders are the area where a careless write has real-world consequences, so treat every mutation here as a change to a business record.

→ `mcp/docs/api/order-statuses` · `mcp/docs/api/payments`

## What an order is made of

- the **storage** it belongs to, which supplies its available statuses;
- **products** with quantities and the prices they were bought at;
- a **customer**, or a guest identity where guest ordering is allowed;
- a **status** on each of its status axes;
- **payment** information, where a payment was attached;
- form data captured at checkout — delivery details, chosen intervals, notes;
- attribute values, keyed by locale and attribute like everywhere else.

## Finding the operations

```text
cms_api_search { "query": "orders", "mutating": false }
cms_api_search { "query": "orders storage" }
```

Note the split: operations on **order storages** configure the container, operations on **orders** work with the records inside it. Both use the word "orders", so read the paths in the search results rather than the summaries alone.

## Order storages come first

An instance may have no order storage at all — it is not baseline data. Without one there is nowhere for an order to live, and the statuses an order can take do not exist yet.

Before working with orders, list the storages and read the one you will use. Its identifier is how everything else refers to it.

## Create an order

Creating an order through the Admin API is not the normal path — orders usually arrive from a storefront. When you do need to, expect to supply the storage, the products with their quantities and prices, the customer, and the initial status.

The prices are recorded on the order rather than looked up, which is what makes an order a historical record rather than a live calculation. Sending a price that does not match the product's current price is legitimate and sometimes correct; it is also how a mistake becomes permanent.

Dry run, and show the human the totals before sending.

## Order statuses

An order's state is not one field. It moves along axes — the lifecycle of the order itself, its fulfilment, and its payment — and the payment axis is managed by the platform except where the order is settled outside any provider.

Read `mcp/docs/api/order-statuses` before changing any status. There is no transition entity to look for: a status moves either by firing a trigger the storage's map pairs with it, or by writing it with `change-status`. Which of the two is right, and which axis actually moves, depends on the storage's `statusMode`.

## Attaching a payment

A payment is a separate entity linked to the order, with its own session and lifecycle. The order's payment axis reflects that lifecycle, and on a provider-backed order it is not something you write — only where the account is of type `custom` is a hand-written payment status the intended mechanism.

→ `mcp/docs/api/payments`

## Discounts on an order

Discounts apply to an order through the discount rules configured on the instance, and their effect is recorded on the order when it is processed. The discount configuration itself lives in its own area.

Two consequences: a discount added after an order exists does not retroactively change it, and an order's totals are what were computed at the time — do not recompute them and write the result back.

→ `mcp/docs/api/discounts`

## Listing filtering and paging orders

Order listings are the largest responses on the API, and this server caps what it hands you. Always page deliberately, with the operation's own `limit` and `offset`, and use its filters to narrow by status, date or customer.

A truncated response reports `shown` and `total`, so you always know how much you did not see.

→ `mcp/docs/server/response-shaping`

## Refunds

Refunds have their own operations, and a customer-initiated refund request is a different thing from a refund an operator performs. Read both before acting: refunding is a money movement, and it is not undoable through the API.

Never issue one without explicit human agreement in the conversation, quoting the order and the amount.

Three operations, all on the order's storage and all gated on `payments.refunds`:

```text
POST /orders-storage/{storageId}/refund/{id}/preview   AdminOrdersStorageController_previewRefund
POST /orders-storage/{storageId}/refund/{id}           AdminOrdersStorageController_refundOrder
GET  /orders-storage/{storageId}/refund/{id}           AdminOrdersStorageController_findRefunds
```

`{id}` is the **order**, not the refund. The preview answers `200` and writes nothing; the history read answers the refunds already issued, each with its amount and note.

## Previewing a refund before issuing one

Both write bodies have the same shape: a `note`, and a `products` map keyed by the **catalog product id** — not by a line number and not by a marker.

```json
{ "note": "wrong size",
  "products": { "118": { "quantity": 2 } } }
```

The preview answers what would be taken and what would remain: `refundAmount`, `remainingProducts` (each `productId`, `quantity`, `originalPrice`), and the before-and-after totals as `oldTotalDue`, `newTotalDue`, `newTotalRaw`, `newTotalSumWithDiscount`, `oldBonusApplied`, `newBonusApplied` and `discountConfig`.

Always preview first and quote `refundAmount` to the human before the real call. The amount is recomputed against the order's discounts, so it is not the product price times the quantity and cannot be worked out by hand.

## What an issued refund answers

The real call answers `201`:

```json
{ "status": "success", "orderRefundId": 12, "amount": 7,
  "succeeded": [ { "id": 12, "sessionId": 501, "amount": 7, "status": "succeeded" } ],
  "failed": [] }
```

`amount` is what was actually returned, and each entry of `succeeded` is one charge that gave money back. `status` is `success` when every charge behind the refund went through and `partial` when some did not — then `failed` lists those. Report a `partial` as partial; never as done.

An order can be refunded more than once, a few products at a time, until what was ordered is used up. After that the next call answers `400 Refund amount should be > 0`. The history read lists every refund issued, each with its `id`, `status`, `amount`, `products` and `note` — read it before a second refund rather than trusting your memory of the first.

## Refunding an order paid in stages

An order paid in stages is refunded **by stage**, not by quantity. Each product entry names the stage markers to return in `stages`:

```json
{ "note": "deposit returned",
  "products": { "118": { "stages": ["deposit"] } } }
```

Read the markers from `GET /api/admin/payments/stages/order/{id}`, which lists every stage with its `marker`, `value` and `status`. On such an order:

- `quantity` answers `400` — it has no meaning there;
- a product entry without `stages` answers `400`;
- a marker the product does not have answers `400` naming it;
- a stage that was never paid answers `5xx`, and nothing is written.

So name only stages whose `status` reads `completed`, and preview first.

## What a refund body ignores without saying so

Four parts of that body fail quietly, and all four leave `refundAmount` smaller than you meant:

- a product id the order does not contain contributes **nothing** — no error, no mention;
- in the preview, a `quantity` higher than the number ordered is **clamped** to the ordered count, so asking for 11 of 10 previews 10 — the real refund refuses the same body with `400`;
- `quantity` omitted, or `0`, selects nothing;
- `stages` is for an order paid in stages; on an order paid in one charge it is ignored (the reverse is refused — see above).

So a preview answering `refundAmount: 0` means the selection matched nothing — check the ids against the order's products before reading it as "there is nothing to refund". `products` must be present even when empty: a body carrying only `note` answers `5xx`.

## Why a refund answers 400 when its preview worked

The preview works on any order; issuing does not. Three refusals account for nearly all of it:

- `400 Payment session not found` — the order has no completed payment. An order that was never actually paid cannot be refunded however its status axes read, and no body changes that. Confirm a completed payment before promising a refund.
- `400 Refund amount should be > 0` — the selection matched nothing. Checked before the payment is looked at, so this one hides the first.
- `404 Order not found` — the `{id}` is wrong. `{storageId}` is not checked against it, so a mismatched pair fails on the order.

→ `mcp/docs/api/payments#split-payments-and-staged-charges`

## Why a site gets 404 for an order that exists

On the Content API a visitor sees only their own orders. Two reads take an order id:

```text
GET /api/content/orders/{id}                  the order itself
GET /api/content/payments/stages/order/{id}   the payment stages of that order
```

- With the token of the visitor who placed the order, both answer `200`.
- With another visitor's token, both answer `404` — the same answer as an id that does not exist.
- With no sign-in, the order read answers `401` and the stages read answers `404`.
- An id that is not an integer answers `400` once the caller is past sign-in.

Starting a payment follows the same rule. `POST /api/content/payments/sessions` with the `orderId` of a signed-in visitor's order answers `201` only with that visitor's token. Another visitor's token, or no sign-in, gets `404 Order not found`, and the visitor's open payment session stays usable. An order placed without sign-in accepts the call without a token.

A `404` from any of these calls therefore does not prove the order is missing. Look the order up through the Admin API before reporting that it does not exist, and never create it again.

## Verifying an order change

Read the order back by id, check the axis you changed, and check that the axes you did not change are untouched. Then check it appears correctly in the listing you would use to find it — allowing a few seconds, because listings lag.

→ `mcp/docs/api/verification-recipes#orders`

## Common mistakes

- **Searching for a status transition.** There is none; fire a trigger or use `change-status`.
- **Recomputing totals.** They are historical values.
- **Assuming a storage exists.** It is not baseline data.
- **Requesting a whole order list.** Page it; the response will be capped.
- **Treating a refund as reversible.** It is not.
