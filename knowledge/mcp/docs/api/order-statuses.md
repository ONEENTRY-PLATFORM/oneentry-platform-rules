# Order statuses and status axes

An order's state is not a single field. It moves along several independent axes, and one of them is managed by the platform rather than written by hand.

Get this wrong and you produce orders whose history does not explain itself, which is expensive to unpick.

→ `mcp/docs/api/orders` · `mcp/docs/api/payments`

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

## The payment axis is managed for you

The payment axis reflects what actually happened to the money: a payment session completing, a webhook arriving, a refund settling. The platform maintains it.

Do not write it directly. A hand-set payment status that disagrees with the payment record is worse than no status at all, because everything downstream believes it.

## Statuses carry meaning flags

A status can be marked as terminal, as a cancellation, or as a successful conclusion. Those flags are what reporting uses to decide whether an order counts as won, lost or still open.

When creating a status, set them deliberately — an unmarked terminal status leaves orders that never appear to finish. When changing them on an existing status, remember that every historical order carrying it is reinterpreted at once.

## Transitions and triggers

Moving between statuses can be configured as a transition rather than an arbitrary write, and a transition can fire triggers — side effects that happen when an order reaches a state.

Where a transition exists, use it. Writing the target status directly skips whatever the transition was meant to cause, and the omission is invisible until someone notices the side effect never happened.

```text
cms_api_search { "query": "order status" }
cms_api_search { "query": "status maps" }
```

## Basic and advanced modes

A storage can run with a simple single-axis view or with the full multi-axis model. Which one it uses changes what the status operations expect.

Read the storage's configuration before assuming. A payload built for one mode is rejected or, worse, partially accepted in the other.

## Changing the status of an order

1. Read the order and note **every** axis, not just the one you care about.
2. Read the storage's statuses and find the target by identifier.
3. Check whether a transition exists for the move you want.
4. Dry run, and show the human the current and target states.
5. Send, then read the order back and confirm the other axes are unchanged.

## Common mistakes

- **Treating the status as one field.** There are several axes.
- **Writing the payment axis.** It is derived from the payment record.
- **Skipping a transition.** Its side effects are skipped with it.
- **Reusing an identifier from another storage.** They must be unique instance-wide.
- **Renaming a status to fix a typo.** Everything referencing it follows the identifier.

→ `mcp/docs/api/verification-recipes#orders`
