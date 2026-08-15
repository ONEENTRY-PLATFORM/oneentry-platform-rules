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

Practical consequences:

- When you move or update an item, **carry its parent reference through unchanged** unless you are deliberately re-parenting. Dropping it flattens the item to the top level, which looks like the menu reorganising itself.
- When you read an item to modify it, keep the whole parent information, not just an id.

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
2. Attach the top-level items with an update, and read them back to learn their ids.
3. Add each level of children against the parents you just read.
4. Set the order last, using the position operation.

Trying to create a nested structure in one payload, or bottom-up, produces items whose parents do not exist yet.

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

→ `mcp/docs/api/verification-recipes#menus`
