# Order statuses and status axes

An order's state is not a single field. It moves along several independent axes, and one of them is managed by the platform rather than written by hand.

Get this wrong and you produce orders whose history does not explain itself, which is expensive to unpick.

→ `mcp/docs/api/order-status-triggers` · `mcp/docs/api/orders` · `mcp/docs/api/payments`

## Statuses belong to a storage

Statuses are defined inside an order storage, which is the container an order lives in. An instance with no order storage has no order statuses.

Read the storage first, and read the statuses it defines. They are customer configuration; there is no platform-wide list and no set of markers you can assume.

## The identifier is unique across the instance

This one surprises people: an order status identifier must be unique across the **whole instance**, not within its storage. Two storages cannot both define a status called `new`.

So a create that looks obviously safe can fail because a different storage already claimed the name. Read the existing statuses before choosing an identifier, and expect to qualify names when an instance has several storages.

## Several axes not one field

An order carries a state on more than one axis at a time:

- the **order lifecycle** — where the order is in its own process;
- **fulfilment** — how far the goods or service have got;
- **payment** — whether and how it has been paid.

They move independently. An order can be paid and unfulfilled, or fulfilled and refunded. Reading only one axis and calling it "the status" is how contradictory reports get produced.

## Which axis a status belongs to

A status names its axis in `axis`, one of `order`, `fulfillment` and `payment`. Which axis a status written **without** the field lands on is decided by the storage's `statusMode`, so read the storage before creating one:

- `basic`, the mode a storage starts in, puts it on `payment` — and refuses an explicit `order` or `fulfillment` with `400 In basic mode only payment axis statuses can be created`;
- `advanced` puts it on `order` and accepts all three.

`axis` is fixed once the status exists. Sending a different one on an update answers `200 true` and leaves the axis as it was, so a status cannot be moved between axes — create one on the axis you want.

The axes share one list and one identifier space, so a storage's statuses come back interleaved and you filter them yourself:

```json
[ { "identifier": "created",         "axis": "order",   "isDefault": true },
  { "identifier": "paid",            "axis": "order",   "isFinalSuccess": true },
  { "identifier": "created-payment", "axis": "payment", "isDefault": true },
  { "identifier": "paid-payment",    "axis": "payment", "isFinalSuccess": true } ]
```

Each axis has its **own** default, so more than one status in a storage carries `isDefault: true` and none of them is wrong. Reading the list for "the default status" and taking the first match picks an axis at random.

An axis with no statuses defined is normal — a storage that never tracks despatch simply has none on `fulfillment`, and that is not something to fill in unasked.

## The payment axis is managed for you

The payment axis reflects what actually happened to the money: a payment session completing, a webhook arriving, a refund settling. The platform maintains it.

Do not write it directly. A hand-set payment status that disagrees with the payment record is worse than no status at all, because everything downstream believes it.

The API stops you only part of the way, and what decides it is the order's payment account. `PUT /orders-storage/{id}/orders/{orderId}/change-status` with `axis: "payment"` is **accepted** when that account is of type `custom` — the type for money settled outside any provider — and the write takes effect. For every other account type an `advanced` storage answers `400 Payment status is managed automatically and cannot be changed manually`.

A `basic` storage does not answer that at all: the same call writes the **order** axis instead and answers `200 true`. So the axis you named is not necessarily the axis that moved. Read the order back and compare both before reporting what you did.

With `axis` left out, `change-status` targets the order axis, and a status belonging to another one answers `400 Status '<identifier>' belongs to axis '<axis>', not 'order'`.

A **status map** is what connects an event to a status. There is one per axis, and each is a flat object from a trigger name to a status identifier — nothing nested, and no other keys:

```json
{ "statusMap": { "payment.waiting":  "created-payment",
                 "payment.paid":     "paid-payment",
                 "payment.expired":  "expired-payment",
                 "payment.canceled": "canceled-payment" } }
```

Read the map rather than inferring the pairing from the names. The `-payment` suffix above is a habit that keeps identifiers unique across the instance, not a rule the platform applies, and a map may point at any identifier at all.

→ `mcp/docs/api/order-status-triggers#where-the-status-maps-live`

## Statuses carry meaning flags

A status can be marked as a cancellation with `isCancelFinal`, or as a successful conclusion with `isFinalSuccess`. Both default to `false`, so a status created without them is neither — which reads as "still open" for as long as nobody notices. Those flags are what reporting uses to decide whether an order counts as won, lost or still open.

When creating a status, set them deliberately — an unmarked terminal status leaves orders that never appear to finish. When changing them on an existing status, remember that every historical order carrying it is reinterpreted at once.

## Basic and advanced modes

`statusMode` on the storage record takes `basic` or `advanced`, and a storage is created `basic`. Which one it is changes what the status operations expect, so read it before building any status payload.

Under `basic` the storage keeps one status that stands for both the order and its payment: every status it will accept belongs to the `payment` axis, the `order` and `fulfillment` maps are not applied, and a storage trigger of its own has nothing to drive. Under `advanced` the three axes are configured and move independently.

A payload built for one mode is rejected or, worse, partially accepted in the other — see the payment axis above for the case that answers `200` and writes a different field than you named. The same record carries `priceExpiration` — how long a price quoted on an order stays good, written as a duration such as `10m` rather than a number of seconds.

## Changing the status of an order

1. Read the storage and note its `statusMode`.
2. Read the order and note **every** axis, not just the one you care about.
3. Read the storage's statuses and find the target by identifier and axis.
4. Read the map for that axis and see whether a trigger already pairs with the target.
5. Dry run, and show the human the current and target states.
6. Send, then read the order back and confirm the axis you named is the one that moved and the others are unchanged.

## Common mistakes

- **Treating the status as one field.** There are several axes.
- **Looking for a transition.** There is no such entity; use a trigger or `change-status`.
- **Assuming a status with no axis lands on order.** Only on an `advanced` storage.
- **Trying to move a status between axes.** The update answers `200` and changes nothing.
- **Reading `200 true` from `change-status` as proof the payment axis moved.** On a `basic` storage it writes the order axis.
- **Reading `201 true` from a fired trigger as proof a status moved.** An unmapped trigger reports success.
- **Reusing an identifier from another storage.** They must be unique instance-wide.
- **Renaming a status to fix a typo.** Everything referencing it follows the identifier.

→ `mcp/docs/api/order-status-triggers` · `mcp/docs/api/verification-recipes#orders`
