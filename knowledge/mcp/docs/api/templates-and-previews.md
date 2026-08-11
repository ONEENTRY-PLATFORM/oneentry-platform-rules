# Templates and template previews

A template defines how something renders. A template preview shows what it produces without publishing it.

Identifiers here are unique and widely referenced, so renaming is a breaking change and re-creating a provisioned template fails.

→ `mcp/docs/api/block-types` · `mcp/docs/api/pages`

## What a template is

A named rendering definition with a type saying what it renders — a page, a block, and so on — plus localized labels so humans can recognise it in the admin panel.

Entities reference a template by its identifier. That reference is what makes the identifier a contract: changing it detaches everything pointing at the old name.

## Finding the operations

```text
cms_api_search { "query": "templates", "mutating": false }
cms_api_search { "query": "template previews" }
```

## Identifiers are unique

Creating a template whose identifier already exists **fails**. That is the useful case — it tells you the template is there.

The nine dynamic block types each ship a default template named after the type with `_default` appended, for example `trending_block_default`. Those are provisioned, existing blocks already point at them, and re-creating one is neither possible nor desirable.

→ `mcp/docs/api/baseline-data#block-types-and-their-default-templates`

## Making a variant

When a human wants something to render differently, do not edit a default template — every entity of that type follows it, so the change is global and usually wider than intended.

Instead:

1. Create a **new** template with a new identifier and its own labels.
2. Point the specific entity at it.
3. Read the entity back and confirm the reference took.

That keeps the default intact for everything else and makes the change reversible by re-pointing.

## Template previews

A preview renders a template against sample or real data so a human can see the result before it goes live. Previews are entities of their own, with their own operations, so a preview can be created, read and removed independently of what it previews.

Use them when a change is visual and the human's acceptance criterion is "does it look right" — that is not a question the API alone can answer.

## Changing a template that is in use

Read which entities reference it first. A template with one consumer is safe to iterate on; one with fifty is a change to fifty things at once.

If you cannot tell how many reference it, say so before changing it rather than after.

## Delete a template

Deleting a template that entities still reference leaves them pointing at nothing, and they then render with whatever fallback exists — which is usually not what anyone wants and is not obviously connected to the delete.

Re-point the consumers first, then delete. Deletion is confirm-gated; use the dry run's target to confirm you have the right template, and count the consumers before confirming.

## Common mistakes

- **Editing a default template to change one block.** Create a variant instead.
- **Re-creating a provisioned default.** The identifier is taken.
- **Renaming an identifier.** References follow the name, not the record.
- **Deleting before re-pointing.** Consumers are left dangling.
- **Assuming a preview is what a site will show.** It is a rendering aid; verify on the real surface too.

→ `mcp/docs/api/verification-recipes#templates`
