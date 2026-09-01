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

The position operations accept `newParentType` next to `newParentId`, with those same two values. Do not carry them over to `leftObjectType` and `rightObjectType` of the same body: a neighbour's kind uses the other vocabulary, `menu-page` or `menu-custom-item`. Two dictionaries in one body — ordering, below, spells it out.

Practical consequences:

- When you move or update an item, **carry its parent reference through unchanged** unless you are deliberately re-parenting. Dropping it flattens the item to the top level, which looks like the menu reorganising itself.
- Carry `parentType` with `parentId`. An id on its own is ambiguous whenever a page item and a custom item of that menu share a number.
- Omitting `newParentType` is accepted: the kind is then worked out from the menu itself, preferring a page item when both match. Send it explicitly when you mean a custom item and the number is also a page item's.

This is the single most common way a menu gets quietly broken.

## Reading a menu from a site

The public route takes the menu's marker under a `marker` segment:

```text
GET /api/content/menus/marker/main   → 200
GET /api/content/menus/main          → 404 Cannot GET
```

The second is the address people try first, and its `404` is a wrong path rather than a missing menu — nothing about the menu needs fixing when you see it.

One kind of item is only ever seen here: a page whose general type is `external_page` has no public address of its own, and the address it points at arrives in `localizeInfos.<locale>.htmlContent` of its item. That makes such a page the way to put an outside link inside a section of the tree.

→ `mcp/docs/api/content-api-reads#a-page-of-type-external-page-has-no-public-address`

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

## Creating a menu with its pages

A create may carry `pagesIds`. The ids are checked first: one that does not exist answers `404` naming it, and no menu is created — so a typo costs you nothing to clean up.

What the create does **not** do is nest anything. `pagesIds` is a flat set wherever it appears, so the pages arrive as siblings at the top level whatever their relationship in the page tree.

Creating empty and attaching with an update afterwards is equally valid and is the better shape when you are building the tree level by level.

## Building a whole menu

Top-down, one level at a time:

1. Create the menu, with or without `pagesIds`.
2. Attach any remaining pages with an update. `pagesIds` is a **flat set**: nesting is not taken from the page tree, so even pages whose parents are also in the menu come back at the top level.
3. Add custom items for everything that is not a page — products, external addresses, column headings. An empty `value` is refused, so a heading with no link needs a placeholder target such as `#`.
4. Nest each level with the position operations, against the parents you read back.

Two things that bite here:

- **The label is the page's `menuTitle`, not its title.** A menu built without setting `menuTitle` arrives carrying page headings instead of the wording the navigation is supposed to show.
- **One page is one item.** `pagesIds` is a set, so a page that has to appear in two places gets a custom item with the same target for the second appearance. Listing it twice loses one of the two silently.

Trying to create a nested structure in one payload, or bottom-up, produces items whose parents do not exist yet.

## Ordering items

The position operations take a required `position` object even when the order does not matter — `{ "leftObjectId": null, "rightObjectId": null }`. A body without `position` answers `400` naming the field; an empty object is not enough.

Both re-parenting and sibling order apply, and a public read returns the items in that order. The order also survives a later update of the menu's `pagesIds`: attaching the same set again does not reshuffle what is already there.

Neighbours are addressed by the id of the item, not of its position: a page item's neighbour is a page id, a custom item's neighbour is a custom item id. In a row of one kind that is all it takes — an unnamed neighbour is read as the same kind as the item being moved.

In a **mixed** row, name each neighbour's kind with `leftObjectType` and `rightObjectType`, and name it in the vocabulary those two fields use:

| field | values | means |
|---|---|---|
| `newParentType` | `page`, `custom` | the kind of the new parent |
| `leftObjectType`, `rightObjectType` | `menu-page`, `menu-custom-item` | the kind of a neighbour |

Both directions work once the kind is spelled this way — a custom item in between two pages, a page in beside a custom item.

```text
PUT /api/admin/menus/8/custom-items/459/position     custom item 459 between pages 3 and 4
  { "newParentId": null,
    "position": { "leftObjectId": 3, "leftObjectType": "menu-page",
                  "rightObjectId": 4, "rightObjectType": "menu-page" } }
```

The parent words `page` and `custom` are understood here too and mean the same two kinds.

**A kind the instance does not know is refused.** A near-miss such as `menu_page`, or any other value, answers `400` naming the side, the neighbour id and the kind that was looked for, and the moved item keeps the rank it had. The same `400` comes back when the kind is right but that neighbour is not in this menu. A neighbour id left out — omitted or `null` — still means "no neighbour on that side" and is accepted: that is how an item goes to either end of a row.

An update that attaches pages assigns order in the order you list `pagesIds`. Omitting `position` while changing the parent leaves the order alone.

## Delete a menu or an item

Removing an item removes the branch beneath it. Deleting the menu removes everything.

Both are confirm-gated where they are destructive. Read the tree before confirming, and tell the human how many items are below the one they asked to remove.

## Common mistakes

- **Losing the parent reference on an update.** The item jumps to the top level.
- **Sorting lexorank positions numerically.** The order comes out wrong.
- **Patching a position field.** Use the position operation.
- **Referring to a page by id.** Use the page URL.
- **Removing a pinned item.** It returns; change the setting instead.
- **Expecting `pagesIds` to carry nesting.** It does not; nest afterwards.
- **Using a page's title as its menu label.** The label is `menuTitle`.
- **Listing one page twice in `pagesIds`.** Use a custom item for the second place.
- **Sending `parentId` without `parentType`.** Ambiguous whenever the two kinds share a number.
- **Writing `page` or `custom` as a neighbour's kind.** That is the `newParentType` vocabulary. A neighbour is `menu-page` or `menu-custom-item`; the wrong word answers 200 and resets the rank.
- **Judging a reorder by the order alone.** Read the lexorank: an unresolved neighbour can leave a plausible order behind.
- **Sending a position body without `position`.** It answers 400; the minimal body is `{ "position": { "leftObjectId": null, "rightObjectId": null } }`.
- **Reading a menu publicly without the `marker` segment.** That path answers 404.

→ `mcp/docs/api/verification-recipes#menus`
