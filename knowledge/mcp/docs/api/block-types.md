# Dynamic block types

Nine block types compute their contents when a site asks, rather than holding a stored list. Each has its own Content API endpoint and its own default template.

They are provisioned with the instance. You configure them; you do not create the types.

→ `mcp/docs/api/blocks` · `mcp/docs/api/baseline-data#block-types-and-their-default-templates`

## The nine types

| Type | What it shows |
|---|---|
| Frequently ordered | Products commonly bought together with a given product |
| Slider | A configured sequence of slides |
| Trending | Products with the most current activity |
| Recently viewed | Products this visitor looked at |
| Repeat purchase | Products this customer has bought before |
| Personal recommendations | Products chosen for this visitor |
| Cart complement | Products that go with what is in the cart |
| Cart similar | Products similar to what is in the cart |
| Wishlist similar | Products similar to what is on the wishlist |

Each has a general type of its own, so a block of one of these kinds is created like any other block — with that type — and then configured.

## Each type has its own endpoint

A site does not read a dynamic block's contents from the block record; it calls the Content API endpoint belonging to that block type, passing the block's marker and whatever context the visitor has.

That is why the marker matters more here than anywhere else: it is the parameter in the URL a site calls.

## Every type ships a default template

Each type is provisioned with a default template whose identifier is the type name followed by `_default` — for example `trending_block_default`. Existing blocks of that type already point at it.

Template identifiers are unique, so re-creating one fails. And creating a replacement by hand produces a block that renders differently from every other block of its type on the instance — usually not what was wanted.

If a human wants a different rendering, create a **new** template with a new identifier and point the specific block at it.

→ `mcp/docs/api/templates-and-previews`

## Configuration lives in the block settings

How many items, which source, which filters, what to do when there is nothing to show — all of it is in the block's own settings object. `cms_api_describe` gives the schema for the create and update operations, and it is normally a loose field.

The reliable approach: read a working block of the same type on the same instance and change one field at a time.

## Context dependent output

Several of these types depend on the visitor: recently viewed, repeat purchase, personal recommendations, and the cart and wishlist companions all produce different results for different people, and nothing at all for a visitor with no history.

So "the block is empty" from an admin's own browser is usually correct behaviour rather than a fault. To check the configuration is right, use the preview facilities rather than looking at a live page.

## Previewing a block

There is a preview operation that returns what a block would produce, and it accepts a simulated context — a target product, a simulated visitor, cart or wishlist contents, and audience overrides.

Two things to know:

- It requires a specific admin permission. Without it the preview answers 403 while the rest of the block operations work.
- Its response reports which audience rule was applied, whether a fallback was used, and any warnings. Those warnings are the fastest route to why a block is empty.

```text
cms_api_search { "query": "blocks preview" }
```

## Audience filtering

A block can be restricted to visitors matching a rule over user attributes, so the same page shows different blocks to different people. The rule lives in the block's settings.

When a block is unexpectedly invisible, check the audience rule before anything else, and use the preview's simulated visitor to test it rather than guessing.

## Common mistakes

- **Editing content on a dynamic block.** Its contents are computed; change the settings.
- **Re-creating a default template.** It exists, the identifier is unique, and a hand-made replacement diverges from every other block of the type.
- **Concluding a block is broken because it is empty for you.** Use the preview with a simulated context.
- **Forgetting the marker is the site-facing handle.** Renaming it breaks whatever calls it.

→ `mcp/docs/api/verification-recipes#blocks`
