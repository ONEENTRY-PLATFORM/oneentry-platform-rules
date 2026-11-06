# Status triggers and status maps for an order

A status does not move on its own. Each axis of an order storage has a **status map** pairing a trigger name with a status identifier, and a status moves when a trigger fires or when it is written directly. This document covers the trigger vocabulary, the maps, and the operations that write them.

Read it before you try to make an order change state, and before you go looking for a transition — there is no such thing here.

→ `mcp/docs/api/order-statuses#which-axis-a-status-belongs-to` · `mcp/docs/api/order-statuses#the-payment-axis-is-managed-for-you`

## The trigger names a map may use

Trigger names are a closed catalogue, and a key outside it is refused:

```text
order.created            order.products_changed
payment.link_created     payment.waiting           payment.partially_paid
payment.paid             payment.expired           payment.canceled
refund.pending           refund.partial            refund.full
```

The `order` and `fulfillment` maps may use any of them. The **`payment` map may not**: it takes only the payment and refund triggers other than `payment.link_created`, so `order.created` or `payment.link_created` in a payment map is refused rather than ignored.

## A storage can define triggers of its own

The catalogue is not the whole vocabulary. A storage can register its own triggers, and the `order` and `fulfillment` maps accept those beside the catalogue names. The `payment` map does not: a custom name there answers `400 Unknown trigger: <name>` exactly as a misspelt one does, which is the same refusal the previous section describes for a catalogue name it does not take.

```text
POST /orders-storage/{id}/triggers            { "identifier": "custom.ready-for-pickup", "localizeInfos": {...} }
POST /orders-storage/{id}/orders/{orderId}/status-trigger   { "trigger": "custom.ready-for-pickup" }
```

The `identifier` **must** start with `custom.`; without the prefix the create answers `400 Custom trigger identifier must start with 'custom.'`. It is also fixed: an update answers `200 true` and changes only the label, so a trigger named wrongly has to be deleted and made again, and every map pointing at the old name has to be rewritten with it.

Firing a name that is neither in the catalogue nor registered on the storage answers `400 Unknown trigger: <name>`. Firing a catalogue name the map says nothing about answers `201 true` and moves no status — the call succeeding is not evidence the order changed.

So "the status did not move" is worth checking against the map and this list before it is treated as a fault: a trigger with no entry was never going to move anything, and it reports success either way.

## Where the status maps live

The two groups of maps are written through different operations, and sending one to the other's route is refused rather than ignored:

| Map | Operation and body |
|---|---|
| `order`, `fulfillment` | `PUT /orders-storage/{id}/status-maps` with `{ axis, statusMap }` |
| `payment` | `PUT /payments/status-maps` with `{ orderStorageId, statusMap }` |

`axis: "payment"` on the storage route answers `400 Status map for axis 'payment' is not managed here` — the payment map is not reachable from the storage at all. On the payments route every value has to name a status that exists **and** sits on the payment axis, or the call answers `400` and stores nothing.

Both map writes are gated, by two different keys: the storage route needs `orders.storage.order-status.update`, the payments route `payments.settings.update`. Without the key the call answers `403 Forbidden resource` and stores nothing. Reading the two is not alike — `GET /orders-storage/{id}/status-maps/{axis}` answers for any signed-in administrator, while `GET /payments/status-maps/{id}` needs `payments.settings.get`.

The operations that move or remove a status declare theirs too:

| Operation | Permission |
|---|---|
| `change-status` on an order | `orders.storage.order-status.update` |
| firing a trigger on an order | `orders.storage.order-status.update` |
| deleting a status | `orders.storage.order-status.delete` |

Two keys read almost alike and are not interchangeable. `orders.storage.order.update` gates editing the order record itself, through `PUT /orders-storage/{id}/order/{orderId}`; every operation in the table above wants the `order-status` key instead. Ask for `orders.storage.order-status.update` when `change-status` or a trigger is refused, and for `orders.storage.order.update` when editing the order is. Each operation's own response text names the key it checks, so reading it is enough.

A status an order already carries cannot be deleted: `DELETE` on it answers `400 Status is used on orders and cannot be deleted`. Move the orders off it first.

## There is no such thing as a transition

Nothing in this API models a move from one status to another as an entity. There is no transition to read, create or invoke, and searching for one wastes a call and ends in a guess.

What exists is three things:

- a **status map** per axis, pairing a trigger name with a status identifier;
- **firing a trigger** on an order, which moves whatever the map for that axis pairs with it;
- **`change-status`**, which writes a status onto an axis directly.

Prefer firing a trigger where the map already pairs one with the status you want: that is the move the storage was configured to make. Reach for `change-status` when no trigger covers it. Neither is a transition, and neither carries side effects beyond the status it writes.

```text
cms_api_search { "query": "order status" }
cms_api_search { "query": "status maps" }
```

→ `mcp/docs/api/order-statuses#changing-the-status-of-an-order` · `mcp/docs/api/orders` · `mcp/docs/api/silent-no-ops#the-table`
