# Product relations

Related, similar and recommended products are produced by rules, not by lists stored on each product. You configure a template of conditions; the platform evaluates it when a site asks.

That distinction is the whole document: if you are looking for a field on the product holding its related products, you will not find one.

→ `mcp/docs/api/products` · `mcp/docs/api/blocks`

## How relations are expressed

A **relation template** holds a set of **conditions**. Each condition narrows the candidate products by comparing an attribute against a value — equal, not equal, greater, less, in a set, not in a set, exists, does not exist.

Conditions combine into a narrowing chain: candidates that satisfy all of them are the related products. An empty template matches broadly and is rarely what anyone wants.

## Finding the operations

```text
cms_api_search { "query": "product relations" }
cms_api_search { "query": "related products" }
```

There is a read that returns the related products of a given product, and admin operations to manage templates and their conditions.

## Global and local relations

A relation can be evaluated globally — the template applied across the catalogue — or locally, scoped to the context the product sits in. The read operation takes a parameter selecting which.

Ask which one the human means before configuring anything. The two produce visibly different results on the same data, and "the related products are wrong" is usually this parameter rather than a broken condition.

## Conditions compare attributes

A condition names an attribute and a comparison. Consequences:

- The attribute must exist in the products' attribute set, and you need its identity from that set — the same read you do before writing any attribute value.
- Only **indexed** attributes are usable: the markers that `GET /api/admin/index-attributes?type=products` returns, plus `price`. Creating or updating a template whose condition names any other marker answers `400` naming it, and nothing is written.
- `conditions` is an array of objects. Each one either pairs `attributeMarker` with a `conditionMarker`, or narrows by `pageUrls`, `statusMarker`, `title` or `ids`. An unknown `conditionMarker`, half of the attribute pair, or an object that narrows by nothing answers `400` naming the element. Empty-string fields count as absent.
- An attribute can drop out of the index after a template was saved. That template then matches nothing rather than erroring, which reads as "no related products", and its next update answers `400` until the condition is changed.

→ `mcp/docs/api/index-attributes` · `mcp/docs/api/attribute-sets`

## Building a template

1. Read the products' attribute set and pick the attributes to narrow on.
2. Confirm those attributes are indexed.
3. Create the template, then add conditions one at a time.
4. After each condition, read the relations of a known product and check the result set shrank the way you expected.

Adding five conditions and then debugging the empty result is much slower than adding them one at a time.

## Previewing before publishing

The admin panel can preview what a template produces. Through the API, the equivalent is to read the related products of a representative product after each change.

Pick two products deliberately: one that should match and one that should not. A template that returns nothing for both is usually a non-indexed attribute; one that returns everything for both is usually a condition that never narrows.

## Relations and blocks

Several block types display related or similar products, and they take their contents from this mechanism. So a block that looks empty on a site may have nothing wrong with it — the template behind it may be matching nothing.

Check the relation before touching the block.

→ `mcp/docs/api/block-types`

## Common mistakes

- **Looking for a list field on the product.** Relations are computed, not stored.
- **Conditioning on a non-indexed attribute.** Refused with `400` on save; a template saved before the attribute left the index returns nothing.
- **Confusing global and local evaluation.** Different results, same template.
- **Deleting a template still referenced by a block.** The block then displays nothing.
- **Building the whole template before testing it.** Add and verify one condition at a time.
