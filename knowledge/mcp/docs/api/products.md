# Products

Products are catalogue entities: a general type, localized content, attribute values, a status, a place in the page tree, and optionally blocks.

Two things about this area catch everyone: there is no plain list operation, and product updates have required and forbidden fields that are not obvious from the schema.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/product-statuses`

## Listing products

There is **no `GET /products`**. The listing operation is a `POST`, and the split between its query and its body is the part that catches everyone:

```text
cms_api_call { "opId": "AdminProductsController_findAll",
               "query": { "limit": 30, "offset": 0, "langCode": "en_US" },
               "body": [] }
```

**Paging and locale are query parameters. The body is an array of filter objects, and an empty array means no filter.** Sending `{ "limit": 30, "offset": 0, "langCode": "en_US" }` as the body answers `400 langCode must match /^[a-zA-Z_]{2,5}$/` — a message about the query parameter you did not send, describing a value that would have satisfied it. The body is bound to the filter array and never looked at for paging.

A `POST` for a read looks wrong and is correct here: the filter array is too large for a query string. `cms_api_search` will show you the operation; do not go looking for a `GET` variant. The catalog classifies this one as a **read**, so it works on a read-only server.

Do not wrap this call in a catch that turns a failure into an empty list. "The catalogue is empty" and "the request was malformed" then look the same, and the next run creates a second copy of everything.

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

That advice is safe **for products**, whose update merges what you send. It is not safe everywhere: on pages and blocks an omitted field is applied as "clear it". Check the entity's own document before you shrink a body.

→ `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it` · `mcp/operating-rules#operations-with-a-single-supported-path`

## One article number is one product

Variants that carry their own article number in the source — colours, sizes, finishes — are **separate products**, not one product with a variant field. The article number is what decides it: different numbers in the source, different products in the catalogue.

Two things follow that are easy to get half right:

- **A product holds only its own images.** Copying the whole gallery into every colour is the same mistake one step later. A source site that shows one shared gallery for every colour is not a model to copy.
- **The link between variants is a relation template**, not an attribute listing the other colours. The attribute the rule compares must be **indexed**, or the rule matches nothing and nothing says so.

Anything with a closed set of values — colour, badge, label, product kind — is a `list`, never free text. A filter can be built from a list and not from a string, and two spellings of one value become two filter entries.

→ `mcp/docs/api/product-relations` · `mcp/docs/api/list-options-and-extra-values`

## There is no stock field

A product carries a status — available, unavailable — and no quantity of its own. A stock number is an ordinary numeric attribute.

Where the source does not publish quantities, agree a placeholder with the human and **record that it is a placeholder** in the handover. Statuses stay real regardless: a product unavailable on the source site is unavailable in the catalogue.

→ `mcp/docs/api/product-statuses` · `mcp/docs/api/bulk-content-migration#report-the-result-as-numbers`

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

A product carries a status resolved by marker, which is what a site uses to decide whether it can be bought. Statuses are instance data — read them, do not assume markers like "in stock". **A fresh instance has none**, so the first status is something you create.

Set the status by including `statusId` in the ordinary product update. There is also a bulk `set-status` operation, and it is a trap worth knowing about: the status id goes in a field called **`id`**, not `statusId`. Given `statusId` the handler reads nothing, writes `NULL` over the status, and still answers `201 true`. It does not re-index either, so even a correct bulk write stays invisible to the listing operation until something else triggers a reindex.

Whichever route you take, read the product back by id and look at `statusId`. The response status is not evidence.

→ `mcp/docs/api/product-statuses` · `mcp/docs/api/silent-no-ops`

## Relations between products

Related, similar and recommended products come from relation templates and conditions rather than from a list stored on the product.

→ `mcp/docs/api/product-relations`

## Filtering a product list

Filters are the body of the list operation, and the body is an **array**: each element is one filter object, and `[]` asks for everything. Paging and locale stay in the query.

Only attributes that are indexed can be filtered on. A filter over a non-indexed attribute returns nothing rather than erroring, which reads as "no such products".

→ `mcp/docs/api/filters` · `mcp/docs/api/index-attributes`

## Common mistakes

- **Looking for `GET /products`.** It does not exist; the list is a `POST`.
- **Paging in the body of the list call.** It belongs in the query, and the error message blames `langCode` for it.
- **Swallowing a failed list into an empty result.** The next run creates duplicates of everything.
- **Bulk `set-status` with a `statusId` field.** It nulls the status and answers success.
- **Omitting `blocks` on an update, or sending `forms`.** Both are avoidable failures.
- **A one-level attribute map.** Silently empty. Read back.
- **Putting every colour in one product.** One article number, one product.
- **A free-text field where the values are a closed set.** Use a `list`.
- **Storing a rating or a review count as an attribute.** The platform calculates both.
- **Treating a `null` price as zero.**
- **Re-creating a product because the list did not show it yet.** Lists lag by seconds; read by id and never repeat a create.

→ `mcp/docs/api/verification-recipes#products`
