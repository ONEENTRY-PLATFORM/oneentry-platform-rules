# Pages

Pages are the content tree: a hierarchy of nodes, each with a general type, localized content, attribute values, and optional blocks and forms attached.

Most site structure work is page work, so this is usually the first entity area an agent touches.

→ `mcp/docs/api/general-types` · `mcp/docs/api/attribute-sets`

## What a page is made of

- a **general type** — `common_page`, `catalog_page`, `error_page` or `external_page`, which decides how the page behaves;
- a **page URL** — the stable string identifier a site uses to fetch it, the page's marker in everything but name;
- **`localizeInfos`** — title and content per locale;
- **`attributesSets`** — attribute values, keyed by locale and attribute;
- a **parent** and a **position** within it;
- optionally **blocks** and **forms** attached to it.

## Finding the operations

```text
cms_api_search { "query": "pages", "mutating": false }
cms_api_search { "query": "pages", "mutating": true }
```

Reads split into the root listing, the children of a node, a fetch by id, and a fetch by page URL. Prefer the URL-based read when you have one: it is the identifier that survives a move between instances.

## The page tree

Pages form a tree through a parent reference. A page with no parent is a root page; the root listing returns those and is where an exploration of an unfamiliar instance starts.

Two habits keep tree work correct:

- **Read the parent before creating a child.** You need its id, and you want to see what its existing children look like.
- **Do not build a tree bottom-up.** Create the parent, read back its id, then create children against it.

## Why a catalog page filter returns plain pages

A paginated page listing takes a page-type filter in its body and a `parentId` in the query, and that is how a catalogue tree is walked one level at a time. Filtering on `catalog_page` deliberately returns more than catalog pages: a plain page is included whenever a catalog page sits **anywhere below it**, at any depth, not only as a direct child. That is what lets a tree render plain pages as folders all the way down to a nested catalogue.

So read each row's own general type before you treat it as a catalogue. A row that came back under a `catalog_page` filter may still be a `common_page` acting as a folder; descend into it with another listing call on its id.

The same rule drives the boolean `hasNestedCatalogPages` on a page read. It is `true` for **every** ancestor of a catalog page, not just its immediate parent, so you can expand a branch from the root without probing each level first.

## Positions inside a parent

Ordering among siblings uses a **lexorank string** on the parent-scoped operations, not a number. Never sort those values numerically, and never reorder by patching the field.

A children read returns siblings ordered by that string, highest first. The Content API returns the same children in the same order, but numbers them instead: the first child carries the **highest** number and the numbers count down to `1` for the last one. Sort that number descending to reproduce the order the panel shows — sorting it ascending reverses the list.

Find the dedicated position operation and use it — it is what renumbers the surrounding siblings correctly:

```text
cms_api_search { "query": "pages position" }
```

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

## Ordering products on a page

A product's place in a page's catalogue is a lexorank position on the product-and-page pair, and it has its own operation:

```text
PUT /api/admin/pages/{pageId}/products/{productId}/position
  { "position": { "leftObjectId": 19, "rightObjectId": null } }
```

Neighbours are named by product id. A neighbour left out — omitted or `null` — means "no neighbour on that side", which is how a product goes to either end of the row. A body carrying no `position` object answers `400` naming that field; `destinationId` on its own is not a substitute for it. In a row of one kind leave `leftObjectType` and `rightObjectType` out: a kind borrowed from another list answers `400` naming the side, the neighbour id and the kind that was looked for, and the product keeps the place it had.

Read the new order back with a product list sorted by position, and make that request address **one** page — either the per-page listing, or the general listing whose body is a single filter carrying one page URL. A position sort on a request that addresses no single page comes back in creation order instead, and the move reads as if it did nothing.

The public read of the same page by URL returns that order in the same direction, so `sortOrder` means the same thing on both sides. Sort descending to reproduce the order the panel shows.

## Create a page

1. Resolve the general type by name.
2. Read the active locale codes.
3. Read the attribute set the page kind uses and build the attribute keys from it.
4. Choose the parent and read it.
5. Dry run the create, then send it.
6. **Read the page back by id** and confirm the attributes are populated.

A page created without `localizeInfos` may be accepted and is unusable — it appears untitled in the admin panel and has nothing to render.

## Update a page

An update merges: `parentId`, `attributesSets` and `generalTypeId` are all left as they are when the body does not carry them, so an edit to a page's content leaves it where it is in the tree.

`parentId: null` is a different instruction. It means "move this page to the root", and it also decrements the former parent's `childrenCount` and recomputes the position — so never send `null` as a stand-in for "no opinion".

Read the page, change what you meant to change, send it back, then read it again and check `parentId` — not the fields you set, the ones you did not.

→ `mcp/docs/server/payload-conventions#an-omitted-field-can-mean-clear-it`

## Attaching blocks

Blocks are attached to a page rather than owned by it: the same block can appear on several pages. The page payload carries the attachment, and the block itself lives in the blocks area.

Create or find the block first, then attach it by its marker. Attaching a block that does not exist yet leaves a reference that resolves to nothing.

→ `mcp/docs/api/blocks`

## Why a page returns fewer blocks than you attached

A public read of a page's blocks is governed by the reading group's rules, and a read restriction caps the answer at a fixed number of records. This read is one the ceiling is applied to, so a page carrying more blocks than the ceiling comes back trimmed — even though other public listings on the same instance answer in full.

Nothing in the answer says so. These reads take no `limit` or `offset` and carry no total, so a trimmed response is indistinguishable from a page that genuinely has that many blocks, and the blocks past the ceiling cannot be reached from the site at all.

The comparison is the tell: read the same page's blocks through the Admin API, which the restriction does not touch. Two different counts mean the restriction, not a lost attachment.

Two ways to open it, both on the instance rather than in the request:

- raise `restrictedDataLength` under `user` in the general settings;
- grant the group unrestricted read on the page routes.

Either takes a few minutes to take effect.

→ `mcp/docs/api/content-api-permission-rules#a-restricted-read-caps-the-list-at-ten`

## Delete a page

Deletion is confirm-gated by this server because it is irreversible, and a page rarely stands alone: it may have children, attached blocks, forms, or products pointing at it as their catalogue parent.

Before confirming:

- read the page's children — deleting a parent affects them;
- check whether anything references its page URL;
- show the human the `target` from the dry run, quoting the page URL and title.

The delete operation may accept flags controlling what happens to dependent content. Read them in `cms_api_describe` and choose deliberately rather than accepting defaults.

## Common mistakes

- **Hardcoding a locale.** A page written only in `en_US` is blank in every other active language.
- **A one-level attribute map.** Accepted, stored empty, no warning. Read back.
- **Assuming a page URL is a route.** It is an identifier for the API, not a path on a website.
- **Using an id from another instance.** Page ids are local; the page URL is the portable handle.
- **Reordering by patching position.** Use the position operation.
- **Reading a trimmed block list as the whole set.** A read restriction caps it, and the response gives no sign.
- **Sending `parentId: null` to mean "leave it".** That is the instruction to move the page to the root.
- **Treating every row of a catalog-filtered listing as a catalogue.** Ancestors of a nested catalog page come back too. Check each row's general type.
- **Reading a `403` on a page write as a missing permission.** The account may hold the key and be confined to another branch of the tree.

→ `mcp/docs/api/verification-recipes#pages` · `mcp/docs/api/admin-page-scope#which-page-operations-the-scope-governs`
