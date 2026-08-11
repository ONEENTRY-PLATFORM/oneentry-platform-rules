# Products

Products are catalogue entities: a general type, localized content, attribute values, a status, a place in the page tree, and optionally blocks.

Two things about this area catch everyone: there is no plain list operation, and product updates have required and forbidden fields that are not obvious from the schema.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/product-statuses`

## Listing products

There is **no `GET /products`**. The listing operation is a `POST`:

```text
cms_api_call { "opId": "AdminProductsController_findAll",
               "body": { "limit": 30, "offset": 0 } }
```

A `POST` for a read looks wrong and is correct here — the filter payload is too large for a query string. `cms_api_search` will show you the operation; do not go looking for a `GET` variant.

For a name lookup there is a lighter quick-search operation that returns a minimal projection. Use it to resolve a name to an id, then fetch the full product by id.

## Create a product

The minimum is a general type, an attribute set, localized content and attribute values:

```json
{ "localizeInfos": { "en_US": { "title": "Blue mug" } },
  "attributeSetId": 5,
  "attributesSets": { "en_US": { "string_id42": "SKU-1", "integer_id43": 7 } } }
```

Both `localizeInfos` and `attributesSets` are usually loose fields, so copy the shape from the example in `cms_api_describe` rather than inferring it.

After creating, **read the product back by id**. A one-level attribute map is accepted, answers 201, and stores nothing.

## Update a product

Two rules that are not visible in the schema:

- **Always include `blocks`.** Send `blocks: []` when you have nothing to set. Omitting it makes the update fail.
- **Never include `forms`.** The schema accepts the field and the update rejects it on save.

Read the product first and send the smallest body that expresses your change. A large speculative body fails as one opaque error; a small one tells you what is wrong.

→ `mcp/operating-rules#operations-with-a-single-supported-path`

## Prices and the price attribute

A price is an ordinary numeric attribute that has been marked as the product's price in its attribute set. That marking is what makes it appear as a price in listings, filters and order calculations rather than as a plain number.

Consequences worth knowing:

- One attribute per set carries that role. Marking a second one is a schema change with visible effects.
- The value is a number or `null`. `null` means "no price set", not zero — do not coerce it.
- Changing a price has effects beyond the product record: anything that reacts to prices reacts to the change, and not always immediately.

## Products live in the page tree

A product is attached to a page — typically a `catalog_page` — which is what gives it a place in the catalogue. The page relationship is how a site navigates to it.

Resolve the parent page before creating a product, and prefer the page URL over an id when you are given a target by a human.

→ `mcp/docs/api/pages`

## Statuses

A product carries a status resolved by marker, which is what a site uses to decide whether it can be bought. Statuses are instance data — read them, do not assume markers like "in stock".

→ `mcp/docs/api/product-statuses`

## Relations between products

Related, similar and recommended products come from relation templates and conditions rather than from a list stored on the product.

→ `mcp/docs/api/product-relations`

## Filtering a product list

The list operation accepts filters over attribute values, but only attributes that are indexed can be filtered on. A filter over a non-indexed attribute returns nothing rather than erroring, which reads as "no such products".

→ `mcp/docs/api/filters` · `mcp/docs/api/index-attributes`

## Common mistakes

- **Looking for `GET /products`.** It does not exist; the list is a `POST`.
- **Omitting `blocks` on an update, or sending `forms`.** Both are avoidable failures.
- **A one-level attribute map.** Silently empty. Read back.
- **Treating a `null` price as zero.**
- **Re-creating a product because the list did not show it yet.** Lists lag by seconds; read by id and never repeat a create.

→ `mcp/docs/api/verification-recipes#products`
