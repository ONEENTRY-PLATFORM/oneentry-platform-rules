# Verification recipes

How to prove a change actually worked, through the public Admin API, without a test suite. Each recipe is a sequence of calls against a throwaway entity, the fields worth asserting, and a cleanup.

The general shape: create something disposable, exercise the behaviour, assert, delete it. Never verify against real customer content.

→ `mcp/docs/server/agent-workflows` · `mcp/docs/api/index-attributes`

## The general pattern

1. **Read first.** Know what exists before you add to it.
2. **Create something obviously disposable** — an identifier like `scratch-…` that no human will mistake for real content.
3. **Assert by id.** Read the entity back and check the fields you set, especially attribute values.
4. **Assert through the read the consumer uses** — the projection a site or the admin panel receives, which is often a different endpoint from the one you wrote to and does not always agree with it.
5. **Then assert through the surface that lags** — the listing, the filter, the search — allowing a few seconds.
6. **Delete it**, and confirm it is gone.

Skipping step 3 is how a silently-empty attribute map survives verification. Skipping step 4 is how a validator that no site will ever see gets reported as saved. Skipping step 6 leaves quota-consuming rubbish behind.

→ `mcp/docs/api/silent-no-ops`

## Attribute sets

Create a set, add one attribute of each type you care about, then create an entity using the set and write a value for each attribute.

Assert: reading the entity by id returns every value, in the right shape for its type — a number as a number, a choice as an option id, a file as an object or a list depending on the count.

Then give one attribute a `requiredValidator` and assert it **twice**: the set read by id shows it, and the set's attributes read **by marker** show it too. The second read is the one that matters — a validator written flat rather than under a locale key passes the first and comes back empty from the second, which is exactly what a site would get.

The failures this catches are the one-level attribute map, which reports success and stores nothing, and the flat validator map, which reports success and is never enforced.

→ `mcp/docs/api/attribute-sets#two-reads-two-answers`

## Pages

Create a root page, then a child under it, then reorder the children with the position operation.

Assert: the child appears under the right parent; the ordering reflects what you set; the page reads correctly by page URL as well as by id; content exists in every active locale you wrote.

Then delete the parent and confirm what happened to the child — that is the behaviour worth knowing before you do it to real content.

## Products

Create a product against a known attribute set and parent page, with a price attribute and a status. Read `GET /product-statuses` first — on a fresh instance it is empty and the status has to be created before the product can have one.

Assert: the product reads back by id with its attributes populated **and `statusId` not null**; it appears in the listing operation, allowing a few seconds; a filter on an indexed attribute finds it; a filter on a non-indexed attribute does **not** — that asymmetry is worth seeing once.

Call the listing the way it wants to be called — paging and `langCode` in the query, an array body — and see the `400` once by sending them in the body, so the misleading `langCode` message is familiar rather than surprising.

Update it, remembering to include `blocks` and to omit `forms`, and assert the change landed.

## Blocks

Create a block, attach it to a scratch page, and set its position.

Assert: the block reads back by marker; the page shows it attached in the expected order; for a dynamic block type, the preview operation returns a result and reports which audience rule applied.

An empty dynamic block is not automatically a failure — read the preview's warnings before concluding anything.

→ `mcp/docs/api/block-types#previewing-a-block`

## Menus

Create the menu with an empty `pagesIds`, then add a page item and a custom item by update, nest one under the other, and reorder.

Assert: the tree reads back with the parent relationships intact — this is the check that catches a parent reference dropped during an update; positions are consistent; deleting a parent item removes its branch.

→ `mcp/docs/api/menus#parent-references-are-polymorphic`

## Forms

Create a form with the body wrapped under `newForm`, with an explicit `type` and two fields of different types, one of them carrying a `requiredValidator`.

Assert: the form reads back by identifier with both fields and their types; **`type` is what you sent and not `null`**; **`validators` on the required field is present in the form read by marker**, which is the projection a site consumes — an empty map there means the validator was written flat and no visitor will ever be stopped by it; a submission recorded against it appears in the form data listing; a numeric field left empty reads as `null`, not zero.

The validator assertion is the one worth being pedantic about. It is the difference between a real check and one that only proves you can echo your own input back.

Delete the form last, and note that submissions go with it.

## Orders

Only against a scratch order storage, never a real one.

Assert: the order reads back with every status axis, not just the one you set; changing a status through a transition moves the axis you expected and leaves the others alone; the payment axis is not something your write changed.

Never verify refunds against anything real. There is no undo.

→ `mcp/docs/api/order-statuses`

## Templates

Create a template with a scratch identifier, point a scratch block at it, then re-point the block back.

Assert: the reference reads back on the block; creating a template whose identifier already exists fails — confirming the uniqueness you are relying on elsewhere.

Separately, and once per instance rather than as a scratch exercise: read `GET /template-previews`. If it is empty, create one and upload a test image with its numeric id as `template`, then assert the returned record carries a `previewLink`. That is the only way to find out that images are being stored without thumbnails, because the upload reports no error either way.

Never verify by editing a provisioned default template.

## Permissions

Read a group and its permissions. Adjust the rules on one existing permission record, and assert the change reads back.

Assert also the negative: creating a permission for a path the group already has **fails**. That failure is the guarantee that stops duplicate permissions accumulating.

Restore the original rules afterwards — permission changes affect real visitors.

→ `mcp/docs/api/users-and-groups`

## Cleaning up

Delete every scratch entity you created, in reverse order of creation, and confirm each is gone.

Anything left behind consumes the instance's record quota and will eventually confuse someone. If a delete is refused or gated and you cannot complete it, tell the human exactly what remains and where.
