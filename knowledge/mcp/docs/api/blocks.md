# Blocks

A block is a reusable content unit attached to pages: a banner, a text section, a product carousel, a slider. One block can appear on many pages, which is what makes it worth being a separate entity.

Blocks are addressed by marker, and their behaviour is decided by their general type.

→ `mcp/docs/api/block-types` · `mcp/docs/api/pages`

## What a block is made of

- a **marker** — the stable string identifier, portable between instances;
- a **general type** — which kind of block this is, and therefore which settings it carries;
- **`localizeInfos`** — title and content per locale;
- **`attributesSets`** — attribute values, keyed by locale and attribute;
- **settings** specific to the block type;
- attachments to pages, each with a position.

## Finding the operations

```text
cms_api_search { "query": "blocks", "mutating": false }
cms_api_search { "query": "blocks", "mutating": true }
```

There is a fetch by marker as well as by id. Prefer the marker: it is the identifier a site uses and the one that survives being carried to another instance.

## Create a block

1. Resolve the general type by name — read `mcp/docs/api/general-types` if you are not sure which.
2. Pick a marker. It appears in site code and in other configuration, so choose one that will not need renaming.
3. Read the active locales and the block attribute set.
4. Dry run, send, read back by marker.

A block created with no `localizeInfos` shows as untitled in the admin panel and is hard for a human to find again.

## Attaching a block to a page

Attachment lives with the page, not with the block, and carries a position within that page. So "add this block to the home page" is a page update, not a block update.

Create or locate the block first, then attach it. Attaching a marker that does not exist yet leaves a reference resolving to nothing, and the page renders without it and without an error.

**Then read the page through the public route a site uses.** A page carrying an attached block has been observed to answer `5xx` on public reads while the admin read of both the page and the block stays perfectly healthy — the block itself is unreadable publicly too, and an empty block is enough to produce it. If that happens, detach the block so the site keeps working, keep the content in page attributes for now, and report it with the page URL and the block marker. Do not conclude the block was written wrongly: it was not.

→ `mcp/docs/api/content-modelling#attribute-first-block-second-new-set-last`

## Updating a block does not leave its pages alone

A block update whose body has no `blockPages` is applied as "this block is on no pages" — every attachment it had is removed, nesting included, and the call answers `200`. This is how a block disappears from a site after an edit that had nothing to do with placement.

Read the block, keep its current `blockPages`, change what you meant to change, send it back whole. Then read it again and look at `blockPages`.

`attributeSetId` is not affected — omitting it leaves it alone. The danger is specific to the page relationship.

→ `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it`

## Ordering blocks on a page

Blocks on a page are ordered. As everywhere in parent-scoped ordering, use the dedicated position operation rather than patching a field, so the surrounding blocks are renumbered correctly.

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## Block settings

Each block type carries its own settings object: how many items to show, which source to draw them from, how to filter them. `cms_api_describe` gives you the schema, and it is usually a loose field — copy the shape from the example, or from an existing block of the same type.

The most reliable way to configure an unfamiliar block type is to read a working block of that type on the same instance and change one field at a time.

## Blocks that fill themselves

The dynamic block types — trending, recently viewed, recommendations, cart and wishlist companions, and the rest — do not hold a product list. They compute their contents when a site asks, from the block's settings and the visitor's context.

Two consequences when one looks empty:

- There may be nothing wrong with the block. The rule behind it may simply match nothing for that visitor.
- Editing the block's own content will not change what it shows. Change the settings or the underlying relation.

→ `mcp/docs/api/block-types` · `mcp/docs/api/product-relations`

## Delete a block

Deletion is confirm-gated. Before confirming, check which pages the block is attached to — the dry run's `target` shows the block, not its attachments, so that is a separate read.

A deleted block leaves those pages without it, silently.

## Common mistakes

- **Addressing a block by id across instances.** Use the marker.
- **Attaching a block that does not exist yet.** Create first, attach second.
- **Editing content on a dynamic block.** Its contents come from settings, not from its own body.
- **Patching a position field.** Use the position operation.
- **A one-level attribute map.** Accepted, stored empty. Read back.
- **Creating a block for a single page.** That is what an attribute is for.
- **Checking only the admin read after attaching.** Read the page publicly too.

→ `mcp/docs/api/verification-recipes#blocks`
