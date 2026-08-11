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

An order's state is not one field. It moves along axes — the lifecycle of the order itself, its fulfilment, and its payment — and the payment axis is managed by the platform rather than set by hand.

Read `mcp/docs/api/order-statuses` before changing any status. Setting one directly when a transition exists for it produces a record whose history does not explain itself.

## Attaching a payment

A payment is a separate entity linked to the order, with its own session and lifecycle. The order's payment axis reflects that lifecycle; it is not something you write.

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

## Verifying an order change

Read the order back by id, check the axis you changed, and check that the axes you did not change are untouched. Then check it appears correctly in the listing you would use to find it — allowing a few seconds, because listings lag.

→ `mcp/docs/api/verification-recipes#orders`

## Common mistakes

- **Writing a status directly when a transition exists.** Use the transition.
- **Recomputing totals.** They are historical values.
- **Assuming a storage exists.** It is not baseline data.
- **Requesting a whole order list.** Page it; the response will be capped.
- **Treating a refund as reversible.** It is not.
