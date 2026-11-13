# Subscriptions

Two related things share the word: **plans**, which an operator defines, and **user subscriptions**, which are individual customers' enrolments in a plan.

Plans are configuration. User subscriptions are records of a paying relationship, so they carry the same care as orders.

→ `mcp/docs/api/payments` · `mcp/docs/api/orders`

## Plans and enrolments

A **plan** describes what is being sold on a recurring basis: its price, its period, what it grants. It is content you create and edit.

A **user subscription** links one customer to one plan, with a `status` of its own. It is created when a customer subscribes, not by hand.

`status` takes one of seven values, and spelling them from memory is how a comparison silently never matches:

```text
pending      active       canceled     unpaid
past_due     incomplete   incomplete_expired
```

Two traps in that list. It is `canceled` with one `l`, not `cancelled`. And there is no `expired` and no `recovering` — a subscription that has lapsed is `past_due` or `unpaid`, and one being put right stays `active` while the customer fixes their payment method.

A fresh enrolment reads `pending`; it becomes `active` when the first payment completes, or when an operator renews it.

## Finding the operations

```text
cms_api_search { "query": "subscriptions", "mutating": false }
cms_api_search { "query": "user subscriptions" }
```

Read the paths carefully — plan operations and user-subscription operations both match the word.

## One live enrolment per customer and plan

A customer has at most one live subscription to a given plan. That invariant is what makes "is this customer subscribed" answerable.

Consequences:

- Creating a second enrolment for a customer who already has one is a mistake, not a renewal.
- A cancelled subscription that is later recovered is the *same* record moving state, not a new one.

Read the customer's existing subscriptions before writing anything.

## States move on their own

Subscriptions expire on schedule and change state when payments succeed or fail. So a state you read may be different a minute later, and a state you write may be overwritten by the platform's own processing.

Do not hand-write a state to reflect a payment outcome. Read the payment side and let the platform reconcile.

→ `mcp/docs/api/payments`

## Plans are tied to the payment provider

A plan sold through a payment provider has a counterpart on that provider's side. Changing the plan's price is therefore not a local edit — it has to be reflected where the money is taken.

Treat a price change on an active plan as a business decision with existing subscribers attached. Say how many are enrolled before making it, and ask what should happen to them.

## Content API surface

Customers see subscriptions through the Content API: listing what is available, subscribing, cancelling, and checking what is currently active. Those routes need the corresponding permissions granted to the relevant group.

If a customer-facing subscribe call returns a permission error, the fix is granting the existing permission to the group — not creating a permission.

→ `mcp/docs/api/users-and-groups`

## Subscribing to a plan with no payment page

Subscribing answers with a payment session: its id, amount, status and a `paymentUrl` to send the customer to.

When the plan's payment account is settled manually rather than through a payment provider, the session comes back with status `waiting` and `paymentUrl` set to `null`. Do not redirect. Tell the customer that payment happens outside the storefront.

The enrolment stays pending and is not listed among the customer's active subscriptions until an operator renews it. Renewing completes the waiting session; cancelling the enrolment cancels it. Subscribing again replaces the waiting session with a new one.

## The subscribe language must carry the plan price

Subscribing takes a `langCode`, and the plan's price is read from its products in that language. Pick a language the products are priced in.

- A language that is not active on the instance answers `400 Language "<code>" is not active`.
- On a plan sold through a payment provider, a product with no price or no currency in that language answers `400 Product's price is not defined (productId: <id>)` or `400 Product's currency is not defined (productId: <id>)`.
- On a manually settled plan, a language in which none of the plan's products has a price answers `400 Subscription price is not defined`. A plan that is meant to cost nothing needs an explicit price of `0` on its products.

Each of these refusals happens before anything is written: the customer's existing enrolment keeps its status and its provider link, so a lapsed subscription can still be recovered. Retry in a priced language rather than changing the enrolment.

## Delete a plan

A plan any enrolment still points at cannot be deleted. The call answers `400`, names how many subscribers it has and how many of those are active or pending, and changes nothing:

```text
Cannot delete subscription #41: it has 1 subscriber(s) (0 active/pending).
Cancel them first to preserve history.
```

Read that count carefully: the refusal is on **any** enrolment in any status, not just a live one. The example above is a plan whose single enrolment is already `canceled` — cancelling every subscriber does not make the plan deletable. Nothing is ever detached, so there is no state in which customers are left holding a grant from a plan that is gone.

So "remove this plan" is usually not a delete. Report the subscriber count to the human and ask whether the plan should instead be left in place and withdrawn from sale.

## Changing a plan while someone is enrolled

A plan carries `isUsed`, and it is `true` while at least one enrolment on it is **active** — a `pending` one does not set it. While it is `true`, three fields are frozen, each with its own refusal:

| Changing | Answer |
|---|---|
| `identifier` | `400 You cannot change marker for active subscriptions` |
| `periodInDays` | `400 You cannot change period for active subscriptions` |
| `paymentAccountId` | `400` on the same grounds |

Sending the fields back unchanged answers `200`, so a read-modify-write that touches nothing else still works. Read `isUsed` before promising a human that a plan can be edited.

## Reading a customer's enrolments

`GET /subscriptions/user/{userId}` lists every enrolment of one customer, finished ones included, nearest `dueDate` first. Each row carries its plan, and the plan carries its payment account — identified by `id`, `identifier`, `type` and `localizeInfos`. The account's `settings` and `testSettings` are not part of this response; read the account itself through the payment account routes when you need its configuration.

The call needs the `users.get` permission — the same one that opens the customer. An administrator who has only `subscriptions.manage` gets `403` here; grant `users.get` rather than retrying.

## Renewing an enrolment by hand

`POST /subscriptions/user/{userId}` with `{ "id": <plan id> }` extends an existing enrolment: it sets `status` to `active`, pushes `dueDate` out by the plan's `periodInDays`, and completes the waiting payment session. When the enrolment already has a `dueDate`, the period is added to it, so renewing early stacks time. When it has none — it was never activated, including one cancelled before its first payment — the period counts from the moment of the call, and the enrolment becomes `active` again. The `id` in the body is the **plan's**, not the enrolment's — the enrolment is found from the two together, and a plan the customer is not enrolled in answers `404 Subscription not found`.

It is not a way to create an enrolment. Only subscribing through the Content API does that, so there is no operator route that puts a customer on a plan.

Renewing by hand only works where the plan's payment account is of type `custom`. On a provider-backed plan it answers `400 Only subscriptions with custom payment account can be renewed manually` — the provider owns the billing cycle there.

## Common mistakes

- **Confusing a plan with an enrolment.** Different entities, different operations.
- **Creating a second enrolment as a renewal.** One live record per customer and plan.
- **Writing a state to reflect a payment.** The platform reconciles it.
- **Changing the price of an active plan casually.** Existing subscribers are affected and the provider side has to agree.
- **Expecting a delete to detach enrolments.** It is refused while any enrolment points at the plan.
- **Reading `cancelled`, `expired` or `recovering` from `status`.** None of the three is a value it takes.
- **Passing an enrolment id to the renew route.** The body takes the plan id.
- **Renewing a provider-backed plan by hand.** Only a `custom` payment account allows it.
- **Redirecting to a null payment link.** A manually paid plan has no payment page.
