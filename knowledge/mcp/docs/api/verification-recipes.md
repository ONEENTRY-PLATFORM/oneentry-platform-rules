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

## Fields that exist for the admin panel

Some structures are there to drive the admin panel: the extra value of a list option, display flags such as `multiselect`, field settings. They are stored as opaque values and any shape is accepted.

For those the chain "wrote it, read it back, it matched" proves **nothing** — the admin read and the public read both hand your own input back, wrong shape included, while the panel shows an empty field or one selection out of four.

So the recipe is different:

1. Write the value.
2. Check it against a **documented** shape rather than against your own memory of it.
3. Ask a human to open that screen in the admin panel and say what they see.
4. If nobody can, report the check as incomplete and name exactly what is unverified.

Step 4 is the one that matters. A confident report on the strength of a read-back is how a field reaches a customer empty.

The recipe for the commonest case of it — an option's extra value and `multiselect` — is with the field itself.

→ `mcp/docs/api/list-options-and-extra-values#checking-an-option-you-wrote` · `mcp/docs/api/bulk-content-migration#panel-facing-fields-cannot-be-verified-by-reading`

## Attribute sets

Create a set, add one attribute of each type you care about, then create an entity using the set and write a value for each attribute.

Assert: reading the entity by id returns every value, in the right shape for its type — a number as a number, a choice as an option id, a file as an object or a list depending on the count.

Then give one attribute a `requiredValidator` and assert it **twice**: the set read by id shows it, and the set's attributes read **by marker** show it too. The second read is the one that matters — a validator written flat rather than under a locale key passes the first and comes back empty from the second, which is exactly what a site would get.

The failures this catches are the one-level attribute map, which reports success and stores nothing, and the flat validator map, which reports success and is never enforced.

→ `mcp/docs/api/attribute-sets#two-reads-two-answers`

## Pages

Create a root page, then a child under it, then reorder the children with the position operation.

Assert: the child sits under the right parent; the ordering reflects what you set; the page reads by page URL as well as by id; content exists in every active locale you wrote. A `position` read from a listing is a lexorank string and cannot be sent back to the update — drop it from the body.

Then delete the parent and see what happens to the child, which is worth knowing before doing it to real content.

## Products

Create a product against a known attribute set and parent page, with a price attribute and a status. Read the statuses first — on a fresh instance there are none and the first one has to be created.

Assert: the product reads back by id with its attributes populated **and `statusId` not null**; it appears in the listing operation, allowing a few seconds; a filter on an indexed attribute finds it and a filter on a non-indexed one does **not** — an asymmetry worth seeing once.

Call the listing the way it wants — paging and `langCode` in the query, an array body — and see the `400` once by sending them in the body, so the misleading `langCode` message is familiar rather than surprising.

Update it, including `blocks` and omitting `forms`, and assert the change landed.

## Blocks

Create a block, attach it to a scratch page, and set its position.

Assert: the block reads back by marker; the page shows it attached in the expected order; **the page still reads through the public route** — a page carrying a block has been observed to answer `5xx` there while both admin reads stay healthy; for a dynamic block type, the preview operation returns a result and names the rule that applied.

An empty dynamic block is not automatically a failure — read the preview's warnings first.

→ `mcp/docs/api/block-types#previewing-a-block`

## Menus

Create the menu with an empty `pagesIds`, then add a page item and a custom item by update, nest one under the other, and reorder.

Assert: the tree reads back with the parent relationships intact — this is the check that catches a parent reference dropped during an update; deleting a parent item removes its branch; the included pages and the custom items **share no id**, which is what catches a branch that would be returned under two parents.

Reordering is the one to try once and stop expecting: the position call answers success, and sibling order comes back as it was.

→ `mcp/docs/api/menus#parent-references-are-polymorphic`

## Forms

Create a form with the body wrapped under `newForm`, with an explicit `type` and two fields of different types, one of them carrying a `requiredValidator`. Then bind it to a module with a form update carrying `formModuleConfigs`, and submit against that binding.

Assert: the form reads back by identifier with both fields and their types; **`type` is what you sent and not `null`**; **`validators` on the required field is present in the form read by marker** — an empty map there means it was written flat and no visitor will ever be stopped by it; **`formModuleConfigs` carries an entry with an `id`**, without which there is nothing a submission can name; the submission appears in the form data listing; an empty numeric field reads as `null`.

Two negatives earn their calls, because both get mistaken for something else: a submission naming a config id from another form is refused with `Incorrect formIdentifier for provided config`, and a form update omitting `formModuleConfigs` answers `200` while deleting the binding and the submission you just made. Run the second on a scratch form only.

The validator assertion is the one to be pedantic about — it separates a real check from echoing your own input back. Delete the form last, and note that submissions go with it.

## Orders

Only against a scratch order storage, never a real one.

Assert: the order reads back with every status axis, not just the one you set; changing a status through a transition moves the axis you expected and leaves the others alone; the payment axis is not something your write changed.

Never verify refunds against anything real. There is no undo.

→ `mcp/docs/api/order-statuses`

## Templates

Create a template with a scratch identifier, point a scratch block at it, then re-point the block back.

Assert: the reference reads back on the block; creating a template whose identifier already exists fails — confirming the uniqueness you rely on elsewhere.

Separately, once per instance: read the preview templates. If there are none, create one and upload a test image with its numeric id as `template`, then assert the returned record carries a preview link. That is the only way to learn that images are being stored without thumbnails, because the upload reports no error either way.

Never verify by editing a provisioned default template.

## Permissions

Read a group and its permissions, adjust the rules on one existing record, and assert the change reads back.

Assert the negative too: creating a permission for a path the group already has **fails**, and that failure is what stops duplicates accumulating. Restore the original rules afterwards — permission changes affect real visitors.

→ `mcp/docs/api/users-and-groups`

## Verifying a batch rather than a sample

After writing the same field across many entities, read them **all** back — for products, with the read that takes a list of ids, so it is one call rather than one per entity.

Assert: every id you wrote is in the response and carries the value. Retry the ones that do not, then read again. A batch write can miss one entity while every response reports success, so the count of successful calls is not the count of stored values.

For anything the platform calculates from what you wrote — a rating above all — wait before asserting, then check again. Zero immediately afterwards is not a result.

→ `mcp/docs/api/bulk-content-migration#read-back-every-object-you-wrote`

## Cleaning up

Delete every scratch entity you created, in reverse order of creation, and confirm each is gone.

Anything left behind consumes the instance's record quota and will eventually confuse someone. If a delete is refused or gated and you cannot complete it, tell the human exactly what remains and where.
