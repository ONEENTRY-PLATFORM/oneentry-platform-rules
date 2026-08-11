# Discounts

A discount reduces what a customer pays, according to conditions evaluated when an order is processed. Coupons are discounts a customer has to present.

Discounts affect money, so treat every change here as a business decision that needs a human's agreement.

→ `mcp/docs/api/orders` · `mcp/docs/api/products`

## The three enums

A discount payload is largely shaped by three closed vocabularies:

| Field | Values |
|---|---|
| Discount type | `NONE`, `PERCENTAGE`, `FIXED_AMOUNT`, `FIXED_PRICE` |
| Applicability | `TO_PRODUCT`, `TO_ORDER` |
| Kind | `DISCOUNT`, `BONUS`, `PERSONAL_DISCOUNT` |

`PERCENTAGE` takes a percentage off. `FIXED_AMOUNT` subtracts a sum. `FIXED_PRICE` replaces the price outright — the one most often chosen by mistake, because a "fixed" discount usually means the second.

`TO_PRODUCT` applies per matching item; `TO_ORDER` applies once to the order total. The same numbers under the two settings produce very different amounts.

## Finding the operations

```text
cms_api_search { "query": "discounts", "mutating": false }
cms_api_search { "query": "coupons" }
```

Read a working discount on the instance before creating one. It is the fastest way to see how the enums, the conditions and the localized labels fit together.

## Conditions decide what a discount applies to

A discount carries conditions narrowing which products or orders it touches — selected products, selected categories, attribute comparisons, thresholds.

Two things follow:

- Conditions over attributes work only on **indexed** attributes. A condition on a non-indexed attribute matches nothing, silently, and the discount appears not to work.
- A discount with no conditions applies broadly. That is occasionally intended and usually not.

→ `mcp/docs/api/index-attributes`

## Stacking

Whether several discounts can apply to one order at the same time is an instance-level setting, not a property of an individual discount.

So the effect of adding a discount depends on the setting as well as on the discount. Check it before promising a human what the total will be.

→ `mcp/docs/api/settings`

## Bonuses and personal discounts

`BONUS` and `PERSONAL_DISCOUNT` are the same machinery aimed differently: one accrues value to a customer, the other targets a specific customer or segment. Both use the same conditions and the same type enum.

A personal discount that is broader than intended is a discount for everyone. Read the conditions back after creating one, and check them against a customer who should **not** qualify.

## Discounts and existing orders

A discount is applied when an order is processed. Creating or changing one does not alter orders that already exist, and it should not — an order is a historical record of what was charged.

If a human wants an existing order adjusted, that is a refund or a manual change to that order, not a discount change.

## Create a discount

1. Read an existing discount to see the shape.
2. Choose the three enum values deliberately, and say in plain words what they mean before sending — "20% off each matching item" or "£5 off the order once".
3. Add conditions, and check them against one product that should match and one that should not.
4. Provide localized labels for every active locale; customers see them.
5. Dry run, show the human the resulting configuration, then send.

## Coupons

A coupon is a discount gated behind a code the customer supplies. The code is the identifier customers type, so it needs to be unambiguous — and once it is distributed, it cannot be changed without invalidating what was distributed.

## Common mistakes

- **Choosing `FIXED_PRICE` when `FIXED_AMOUNT` was meant.** One sets the price, the other reduces it.
- **Confusing per-item and per-order applicability.** Very different totals.
- **Conditioning on a non-indexed attribute.** Silently matches nothing.
- **Creating a discount with no conditions.** It applies to everything.
- **Expecting a discount to change existing orders.** It does not, by design.
