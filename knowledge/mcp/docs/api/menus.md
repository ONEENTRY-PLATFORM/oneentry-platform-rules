# Menus

A menu is a navigation tree whose items point at pages or at arbitrary targets. Both kinds of item live in the same tree and are ordered together.

The two things that catch people: parent references are polymorphic, and ordering is lexorank on the admin side but numeric on the public side.

→ `mcp/docs/api/pages` · `mcp/docs/server/payload-conventions`

## What a menu is made of

- a **menu**, identified by a unique identifier;
- **page items** — entries pointing at pages in the content tree;
- **custom items** — entries pointing at anything else, with their own titles and targets;
- a **parent** and a **position** per item, forming the tree.

The identifier is unique, so creating a menu that already exists fails. That is the good failure — read the list first.

## Finding the operations

```text
cms_api_search { "query": "menus", "mutating": false }
cms_api_search { "query": "menus", "mutating": true }
```

There is a fetch by marker as well as by id, and separate operations for adding and removing items.

## Parent references are polymorphic

An item's parent may be another page item or a custom item — the two kinds mix freely in one tree. So a parent reference is meaningful only together with the kind of thing it refers to.

Page items and custom items are numbered separately, so the same number is a valid reference to two different items of one menu. A parent reference therefore carries a kind alongside the id:

- `parentId` — the number, `null` at the top level;
- `parentType` — `page` or `custom`, `null` at the top level;
- `itemType` on every item of a public menu tree — `page` or `custom`. This is the value a child of that item must send as its `parentType`.

The position operations accept `newParentType` next to `newParentId`, using the same two values as the `leftObjectType` and `rightObjectType` of the same body.

Practical consequences:

- When you move or update an item, **carry its parent reference through unchanged** unless you are deliberately re-parenting. Dropping it flattens the item to the top level, which looks like the menu reorganising itself.
- Carry `parentType` with `parentId`. An id on its own is ambiguous whenever a page item and a custom item of that menu share a number.
- Omitting `newParentType` is accepted: the kind is then worked out from the menu itself, preferring a page item when both match. Send it explicitly when you mean a custom item and the number is also a page item's.

This is the single most common way a menu gets quietly broken.

## Positions are lexorank here and numbers there

On the admin operations, an item's position is a **lexorank string**. On the public side the same ordering is exposed as a **number**.

That is not an inconsistency to correct: they are two representations of one order. Never sort the string form numerically, and never write a number where the string form is expected.

Reorder with the dedicated position operation, which renumbers the siblings. Patching the field directly produces an order that looks right in one place and wrong in another.

## Add a page to a menu

1. Resolve the page — by page URL, not by an id you were handed.
2. Read the menu and find the parent item the new entry should sit under.
3. Add the item, carrying the parent reference explicitly.
4. Read the menu back and check the tree looks as intended.

## Pinned items

An item can be marked so that it is added to the menu automatically rather than by hand. When that is on, entries appear in the tree that you did not create.

Two implications: do not treat an unexpected item as corruption, and do not remove one without checking whether it will simply come back.

## Create the menu empty

**Create a menu with `pagesIds: []`, always.** A create carrying a non-empty `pagesIds` answers `500 null value in column "page_id"` — the create path does not build the join rows the pages need, so the insert fails on a not-null column.

Attach the pages with an **update** afterwards. The update path builds those rows correctly, which is why the two calls behave differently on what looks like the same field.

This 500 is known and expected, not an instance fault. The general rule for a 5xx is to stop and report it; this is the documented exception, so route around it instead of escalating.

## Building a whole menu

Top-down, one level at a time:

1. Create the menu with an empty `pagesIds`.
2. Attach the pages with an update. `pagesIds` is a **flat set**: nesting is not taken from the page tree, so even pages whose parents are also in the menu come back at the top level.
3. Add custom items for everything that is not a page — products, external addresses, column headings. An empty `value` is refused, so a heading with no link needs a placeholder target such as `#`.
4. Nest each level with the position operations, against the parents you read back.

Two things that bite here:

- **The label is the page's `menuTitle`, not its title.** A menu built without setting `menuTitle` arrives carrying page headings instead of the wording the navigation is supposed to show.
- **One page is one item.** `pagesIds` is a set, so a page that has to appear in two places gets a custom item with the same target for the second appearance. Listing it twice loses one of the two silently.

Trying to create a nested structure in one payload, or bottom-up, produces items whose parents do not exist yet.

## Ordering items

The position operations take a required `position` object even when the order does not matter — `{ "leftObjectId": null, "rightObjectId": null }`. A body without `position` answers `400` naming the field; an empty object is not enough.

Both re-parenting and sibling order apply, and a public read returns the items in that order. The order also survives a later update of the menu's `pagesIds`: attaching the same set again does not reshuffle what is already there.

Neighbours are addressed by the id of the item, not of its position: a page item's neighbour is a page id, a custom item's neighbour is a custom item id. In a mixed row, name each neighbour's kind with `leftObjectType` and `rightObjectType` — without them a neighbour of the other kind is not found, and the moved item lands at the edge of the list instead of between the two.

An update that attaches pages assigns order in the order you list `pagesIds`.

→ `mcp/docs/api/silent-no-ops`

## Delete a menu or an item

Removing an item removes the branch beneath it. Deleting the menu removes everything.

Both are confirm-gated where they are destructive. Read the tree before confirming, and tell the human how many items are below the one they asked to remove.

## Common mistakes

- **Creating a menu with its pages in one call.** It answers 500; create empty and attach with an update.
- **Losing the parent reference on an update.** The item jumps to the top level.
- **Sorting lexorank positions numerically.** The order comes out wrong.
- **Patching a position field.** Use the position operation.
- **Referring to a page by id.** Use the page URL.
- **Removing a pinned item.** It returns; change the setting instead.
- **Expecting `pagesIds` to carry nesting.** It does not; nest afterwards.
- **Using a page's title as its menu label.** The label is `menuTitle`.
- **Listing one page twice in `pagesIds`.** Use a custom item for the second place.
- **Sending `parentId` without `parentType`.** Ambiguous whenever the two kinds share a number.
- **Sending a position body without `position`.** It answers 400; the minimal body is `{ "position": { "leftObjectId": null, "rightObjectId": null } }`.

→ `mcp/docs/api/verification-recipes#menus`
