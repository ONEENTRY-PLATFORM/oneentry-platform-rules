# Similar product blocks

Three block types show products that resemble another product: the one on a product page, and the two that work from a visitor's cart and wishlist. They share one set of settings, so what you learn configuring one applies to all three.

Read this before writing settings on any of them. Most of the ways these blocks come back empty are configuration the API accepted without complaint.

→ `mcp/docs/api/block-types` · `mcp/docs/api/blocks`

## Three block types share one set of settings

The settings live in the block's own settings object, under keys the server reads the same way whichever of the three types the block has:

- `mode` — only on the product-page type;
- `similarScope` — where to look and who qualifies;
- `autoSimilarAttributes` and `autoSimilarMinMatches` — what must match the source product;
- `similarCollapseBy` — which products count as one thing.

A block with none of them keeps working: every key is optional and the defaults are the behaviour blocks had before the keys existed.

## Manual or automatic decides what a request returns

`mode` takes `manual` or `automatic`, and it — not the request — decides what comes back:

- **`manual`** returns a static shelf resolved from the block's own rules. Every visitor sees the same products, and a `productId` on the request is ignored.
- **`automatic`** computes the result for the product in the request. Without `productId` there is nothing to compute from, so the call answers `400 productId is required for automatic mode` rather than an empty list.

An absent `mode` reads as `manual`.

The cart and wishlist types have no mode: their source is whatever the visitor has, so the matching settings always apply.

Settings belonging to the mode a block is not in are kept, not deleted. Switching back finds them where they were.

## The scope replaces the source section

`similarScope` has four fields:

```json
{
  "pageIds": [12, 34],
  "hard": [{ "attributeMarker": "brand", "conditionMarker": "same" }],
  "soft": ["string_id4"],
  "priceInfluence": 0.2
}
```

- **`pageIds`** is the whole search area. The source product's own section is **not** added to it — send that section too if you want it. An **empty** list means "not configured", and then the area is the source product's own section.
- Descendants are resolved for you. Send the sections you chose, not their subtrees, or the area stops matching the tree as soon as someone adds a subsection.
- **`hard`** conditions must all pass. A condition using the `same` operator resolves against the source product, which is how "same brand as this one" is expressed without naming a brand.
- **`soft`** only orders the result; it never excludes anything.
- **`priceInfluence`** is `0` to `1` and decides how much a close price lifts a product in the order.

Mind which name each field wants: a `hard` condition names an attribute by its **marker**, the way every product filter does, while `soft` and `autoSimilarAttributes` want the **storage key** described below. The two are not interchangeable, and neither reports the other as wrong.

The object is checked on create and on update: a malformed one answers `400` instead of being stored and quietly producing nothing.

## Matching attributes are named by their storage key

`autoSimilarAttributes` holds the keys attribute values are stored under — the `<type>_id<n>` form, such as `string_id4` — not attribute markers and not attribute set identifiers. A key that does not exist matches no product, the block comes back empty, and nothing reports the mistake.

Read the source product's attribute values and copy the keys from there.

`autoSimilarMinMatches` is how many of those attributes must match. It filters first — a product matching fewer is not in the block at all — and among those that pass, more matches rank higher. Absent, one match is enough.

→ `mcp/docs/server/payload-conventions` · `mcp/docs/api/attribute-sets`

## Collapsing variants into one card

`similarCollapseBy` names what makes two products the same thing, so a size run or a colour range takes one place in the block instead of seven.

- A block created as one of these three types is given the title as its collapse key. That happens **once, at creation** — it is never applied on a read, so an existing block whose value is empty means collapsing is off, and it stays off.
- The named keys combine with AND.
- A key list that does not include the title merges products that merely share an attribute value — every black item in the catalogue becoming one card. The diagnostics read below reports this.

## Why a block comes back short

There is an admin read that counts the block's area without running the selection or writing anything:

```text
cms_api_search { "query": "blocks similar diagnostics" }
```

It needs the same permission as the block preview, answers `404` for an unknown block, and returns:

- `productCount` and `modelCount` — how many products are in the area, and how many remain distinct after collapsing. Equal numbers mean the catalogue has no variants and collapsing gains nothing.
- `shortBlockProductCount` — for how many products the block would gather fewer than two cards.
- `warnings` — where a number was skipped, or a setting breaks silently.

A short block is not a fault. Block length is the merchant's setting, and a section that runs out of comparable products is an ordinary catalogue fact. Report the numbers; do not "fix" them by widening the scope on your own initiative.

## Common mistakes

- **Sending `productId` and expecting it to matter.** In `manual` mode it is ignored; the shelf is the same for everybody.
- **Leaving `mode` at `automatic` for a block a site calls without a product.** Every such call answers 400.
- **Naming attributes by marker in `autoSimilarAttributes`.** Use the storage key; a wrong one matches nothing, silently.
- **Expecting `pageIds` to add to the product's own section.** It replaces it.
- **Expanding a section's subtree into `pageIds`.** Descendants resolve on their own, and the expanded list goes stale.
- **Treating a short block as a defect.** Read the diagnostics before concluding anything.

→ `mcp/docs/api/block-types#previewing-a-block` · `mcp/docs/api/verification-recipes#blocks`
