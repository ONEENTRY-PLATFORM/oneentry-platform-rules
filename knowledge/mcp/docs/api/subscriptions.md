# Subscriptions

Two related things share the word: **plans**, which an operator defines, and **user subscriptions**, which are individual customers' enrolments in a plan.

Plans are configuration. User subscriptions are records of a paying relationship, so they carry the same care as orders.

→ `mcp/docs/api/payments` · `mcp/docs/api/orders`

## Plans and enrolments

A **plan** describes what is being sold on a recurring basis: its price, its period, what it grants. It is content you create and edit.

A **user subscription** links one customer to one plan, with a state of its own: pending, active, expired, cancelled, or recovering. It is created when a customer subscribes, not by hand.

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

## Delete a plan

Deleting a plan that has enrolments detaches those enrolments rather than tidying anything up, and the customers involved keep whatever the plan granted them until their record expires.

Before confirming a delete, count the enrolments and tell the human. Ending a plan properly is usually a state change plus a communication, not a delete.

## Common mistakes

- **Confusing a plan with an enrolment.** Different entities, different operations.
- **Creating a second enrolment as a renewal.** One live record per customer and plan.
- **Writing a state to reflect a payment.** The platform reconciles it.
- **Changing the price of an active plan casually.** Existing subscribers are affected and the provider side has to agree.
- **Deleting a plan with enrolments.** They are detached, not cleaned up.
