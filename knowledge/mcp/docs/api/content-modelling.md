# Choosing the content model

How to decide where content goes before writing it: an attribute, a block, or a set of its own. The model is what a human reviews when they open the admin panel, so it is the deliverable, not an implementation detail.

Read this at the start of a migration, before the first entity exists. Every rule here is one a customer asked for after seeing the result of the alternative.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/blocks`

## Attribute first block second new set last

In order, and stop at the first that fits:

1. **An attribute** — the content belongs to one entity. This is the default.
2. **A block** — the same content is shown on more than one page. Reuse is the whole justification for a block.
3. **A set of its own** — the attributes have outgrown the shared set and cannot be merged into it.

A block used on a single page is an extra object in the tree: a human opens the blocks section and finds records that are reused nowhere. If a page needs ten fields, that is ten attributes, not a block.

## Where interface labels belong

Text that belongs to the interface rather than to any one entity — a button caption, an empty-cart message, a thank-you line — has no natural home on a page or a block. Put it on an attribute set of the `system` type, and keep the text in each attribute's `initialValue`.

`initialValue` is stored on the **schema** rather than on any entity, and it comes back when the set is read. That is what lets the text exist with no carrier at all: a `system` set attaches to nothing, so the labels share no lifecycle with a page, nobody edits them by opening unrelated content, and deleting content never takes them along.

It is locale-keyed like every other localized field on an attribute, one level deeper than the shape suggests:

```json
{ "initialValue": { "en_US": { "value": "Add to cart" } } }
```

Group the sets the way a person would look for them — one per screen or subsystem, rather than one set holding every string on the instance.

The alternative is worse in a specific way: a `forPages` or `forBlocks` set has to hang off a real entity, so the labels become part of that entity's content and inherit its visibility, its locale coverage and its deletion.

→ `mcp/docs/api/attribute-sets#attribute-set-types-are-provisioned` · `mcp/docs/api/locales`

## A page is many attributes not one editor field

A page such as "about us" is a heading, a lead paragraph, several sections, images and captions. Each part is its own attribute of its own type.

The sign that it was done wrong: one `text` attribute holding the whole layout. Content like that cannot be reordered, translated, rendered differently on a phone, or edited in one place — only replaced whole.

So take the source page apart before writing anything, and create an attribute per part. If the parts repeat from page to page and there are many, that is the case for a set of its own.

## textWithHeader holds several heading and text pairs

The value of a `textWithHeader` attribute is a **list**, and the list is the point of the type: it holds pairs of a heading and its text.

```json
[ { "header": "How to use",
    "htmlValue": "<p>Rinse before first use.</p>",
    "mdValue": "", "plainValue": "",
    "params": { "editorMode": "html" } } ]
```

An accordion of twelve questions is one attribute with twelve items. Three ways of doing it that all come back as complaints: everything in one item with the questions marked up inside, one attribute per question, or the heading in one attribute and the text in a neighbour.

→ `mcp/docs/api/attribute-types#text-has-two-forms`

## An attribute title is text a visitor will read

The title of an attribute is not a note for a developer — it is displayed. If a product page says "Color", the attribute is titled `Color`, not `Variant name` or `Option 1`, and the wording comes from the source rather than being improved on the way.

Read a set's titles in order once it is built. If `attribute13`, `Field 2` or `Other` is among them, the set is not ready to be shown to anyone.

## Choose the content language before the first write

The locale of the content and the language of the names a human sees — attribute titles, set names, form titles — are one decision, and it belongs before the migration, not during it.

Ask which language the content is in. Do not infer it from the language of the conversation: they are regularly different, and changing it afterwards means revisiting every object already written.

→ `mcp/docs/api/locales`

## Name every object after what is inside it

A human reads the admin panel as a tree. "Block 1", "Form 2" and "Page" make that tree unreadable, and an entity created with no `localizeInfos` shows as untitled and is hard to find again.

Same for emptiness: an attribute created "for later" and left blank reads as unfinished work. Either fill it or leave it out.

## Say which fields are deliberately empty

Sometimes the source has no value for a field. That is a decision, and it is only a decision if it is stated: a blank field mentioned in the handover is a choice, and the same blank field found by the customer is a defect.

List them when you report the result, with the reason. The same goes for a placeholder value agreed with a human — record that it is a placeholder.

→ `mcp/docs/api/bulk-content-migration#report-the-result-as-numbers`

## What a customer looks at when reviewing

Not the API responses. They open the admin panel and check whether objects are named clearly, whether content sits in separate fields rather than in one editor, whether a product behaves as a product, and whether the platform calculates what it can calculate.

Which is why the model is worth an hour of thought before the first write, and why a technically successful import can still be rejected on sight.

→ `mcp/docs/api/products` · `mcp/docs/api/rating-forms-and-reviews`
