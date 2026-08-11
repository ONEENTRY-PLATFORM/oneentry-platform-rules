# General types

A general type classifies an entity: which kind of page this is, which kind of block, whether something is a product or a form or an order. Almost every create payload needs one.

They are provisioned with the instance and their names are unique, so this is a read-and-pick exercise, never a create.

→ `mcp/docs/api/baseline-data#general-types` · `mcp/docs/api/pages`

## What you will find

Expect these names on a provisioned instance:

- Pages — `common_page`, `catalog_page`, `error_page`, `external_page`
- Products — `product`, `product_preview`
- Blocks — `common_block`, `product_block`, `similar_products_block`, plus one per dynamic block type
- Other — `form`, `order`, `service`, `discount`

Read the list rather than trusting this one:

```text
cms_api_search { "query": "general types", "mutating": false }
```

## Resolve by name never by id

Ids here are not contiguous and not consistent between instances: types have been added and removed over the platform's life, and the vacated numbers were never reused.

An id you remember, or copied from another instance's data, points at a different type or at nothing. Look the type up by name, every time, and keep the id only for the duration of the task.

## The name is unique

Creating a general type that already exists **fails**. That is the good case — you learn it is there and move on.

There is essentially no reason to create one. The set of types is what the admin panel knows how to render, so a type nobody wrote a UI for is not useful.

## Which type to pick

For pages, the type decides how the page behaves rather than only how it is labelled:

- `common_page` — an ordinary content page.
- `catalog_page` — a page that lists products; the usual parent for products in a catalogue tree.
- `error_page` — used for error responses on the site.
- `external_page` — a navigation entry pointing somewhere outside the instance.

For blocks, the type decides which settings the block carries and, for the dynamic types, which Content API endpoint serves its contents.

If you are unsure which type an existing entity uses, read a comparable entity and copy its type.

## Types are attached to modules

Each general type belongs to an admin module, which is what makes entities of that type appear in the right section of the admin panel. That wiring is provisioned; you do not set it up and should not try to change it.

→ `mcp/docs/api/modules`

## Before you create an entity

1. Read the general types and find the one you need, by name.
2. Note its id for this payload only.
3. Check what an existing entity of that type looks like — it tells you which other fields the type expects.
4. Build the payload, dry run, send, read back.

→ `mcp/docs/api/blocks` · `mcp/docs/api/products`
