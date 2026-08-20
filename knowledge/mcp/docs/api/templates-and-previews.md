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

A template-previews record is two things at once, and the second one is easy to miss.

It renders a template against sample or real data so a human can see the result before it goes live. Previews are entities of their own, with their own operations, so a preview can be created, read and removed independently of what it previews. Use them when a change is visual and the human's acceptance criterion is "does it look right" — not a question the API alone can answer.

It is **also the recipe by which uploaded images get their thumbnails**. Its `proportions` say what sizes to generate and how to align them, and its **numeric id** is what the `template` query parameter on an upload refers to.

That second role is why this is not an optional nicety: a fresh instance has no template-previews records, uploads then produce no `previewLink`, and nothing anywhere reports it.

## A fresh instance has none of these

The dynamic block types ship with default *templates*. Template **previews** are not provisioned — `GET /template-previews` on a new instance answers `[]`.

Create one before the first image upload:

- an **identifier** and a title, as with every marker-addressed entity;
- **`proportions`**, giving the horizontal, vertical and square variants you want, each with its size and alignment.

Then read the list back and keep the numeric id. `template=<id>` on an upload is what makes previews appear; a marker is rejected on the database type, and an id that does not exist is ignored without an error on older instances.

→ `mcp/docs/api/files-and-uploads#no-preview-template-no-preview-and-no-error` · `mcp/docs/api/baseline-data`

## Changing a preview template does not re-crop old images

A preview is produced when the file is uploaded, from the `proportions` the template held at that moment. Editing those proportions later, or pointing an attribute at a different template, leaves every file already uploaded with the crop it was given. The update answers success and says nothing about the images that already exist.

So "make the thumbnails bigger" is two steps, not one: change the template, then regenerate. Report only the first and the site keeps serving the old crops.

## Regenerate previews for files already uploaded

`POST /template-previews/{id}/regenerate` — `AdminTemplatePreviewsController_regenerate`, permission `settings.templatePreview.update`.

- `200` carrying `enqueued`, `affectedAttributeSets` and `message`.
- `404` when no template has that id.
- `400` when the template carries no usable `proportions`. Give it proportions first; there is nothing to rebuild from without them.

The rebuild is asynchronous. The answer means the work was accepted, not that any image has changed — confirm by re-reading a file's `previewLink` after a pause rather than from the status code.

`affectedAttributeSets` counts the stored attribute sets referencing this template. **Zero is an answer, not a failure**: nothing already uploaded points at the template, so the change reaches new uploads only. Tell the human that instead of retrying.

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
- **Editing `proportions` and reporting the images updated.** Only new uploads follow the change until you regenerate.
- **Assuming a preview is what a site will show.** It is a rendering aid; verify on the real surface too.

→ `mcp/docs/api/verification-recipes#templates`
