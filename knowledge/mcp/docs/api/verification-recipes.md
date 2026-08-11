# Verification recipes

How to prove a change actually worked, through the public Admin API, without a test suite. Each recipe is a sequence of calls against a throwaway entity, the fields worth asserting, and a cleanup.

The general shape: create something disposable, exercise the behaviour, assert, delete it. Never verify against real customer content.

→ `mcp/docs/server/agent-workflows` · `mcp/docs/api/index-attributes`

## The general pattern

1. **Read first.** Know what exists before you add to it.
2. **Create something obviously disposable** — an identifier like `scratch-…` that no human will mistake for real content.
3. **Assert by id.** Read the entity back and check the fields you set, especially attribute values.
4. **Then assert through the surface that lags** — the listing, the filter, the search — allowing a few seconds.
5. **Delete it**, and confirm it is gone.

Skipping step 3 is how a silently-empty attribute map survives verification. Skipping step 5 leaves quota-consuming rubbish behind.

## Attribute sets

Create a set, add one attribute of each type you care about, then create an entity using the set and write a value for each attribute.

Assert: reading the entity by id returns every value, in the right shape for its type — a number as a number, a choice as an option id, a file as an object or a list depending on the count.

The failure this catches is the one-level attribute map, which reports success and stores nothing.

→ `mcp/docs/api/attribute-sets`

## Pages

Create a root page, then a child under it, then reorder the children with the position operation.

Assert: the child appears under the right parent; the ordering reflects what you set; the page reads correctly by page URL as well as by id; content exists in every active locale you wrote.

Then delete the parent and confirm what happened to the child — that is the behaviour worth knowing before you do it to real content.

## Products

Create a product against a known attribute set and parent page, with a price attribute and a status.

Assert: the product reads back by id with its attributes populated; it appears in the listing operation, allowing a few seconds; a filter on an indexed attribute finds it; a filter on a non-indexed attribute does **not** — that asymmetry is worth seeing once.

Update it, remembering to include `blocks` and to omit `forms`, and assert the change landed.

## Blocks

Create a block, attach it to a scratch page, and set its position.

Assert: the block reads back by marker; the page shows it attached in the expected order; for a dynamic block type, the preview operation returns a result and reports which audience rule applied.

An empty dynamic block is not automatically a failure — read the preview's warnings before concluding anything.

→ `mcp/docs/api/block-types#previewing-a-block`

## Menus

Create a menu, add a page item and a custom item, nest one under the other, then reorder.

Assert: the tree reads back with the parent relationships intact — this is the check that catches a parent reference dropped during an update; positions are consistent; deleting a parent item removes its branch.

→ `mcp/docs/api/menus#parent-references-are-polymorphic`

## Forms

Create a form with the body wrapped under `newForm`, with two fields of different types.

Assert: the form reads back by identifier with both fields and their types; a submission recorded against it appears in the form data listing; a numeric field left empty reads as `null`, not zero.

Delete the form last, and note that submissions go with it.

## Orders

Only against a scratch order storage, never a real one.

Assert: the order reads back with every status axis, not just the one you set; changing a status through a transition moves the axis you expected and leaves the others alone; the payment axis is not something your write changed.

Never verify refunds against anything real. There is no undo.

→ `mcp/docs/api/order-statuses`

## Templates

Create a template with a scratch identifier, point a scratch block at it, then re-point the block back.

Assert: the reference reads back on the block; creating a template whose identifier already exists fails — confirming the uniqueness you are relying on elsewhere.

Never verify by editing a provisioned default template.

## Permissions

Read a group and its permissions. Adjust the rules on one existing permission record, and assert the change reads back.

Assert also the negative: creating a permission for a path the group already has **fails**. That failure is the guarantee that stops duplicate permissions accumulating.

Restore the original rules afterwards — permission changes affect real visitors.

→ `mcp/docs/api/users-and-groups`

## Cleaning up

Delete every scratch entity you created, in reverse order of creation, and confirm each is gone.

Anything left behind consumes the instance's record quota and will eventually confuse someone. If a delete is refused or gated and you cannot complete it, tell the human exactly what remains and where.
