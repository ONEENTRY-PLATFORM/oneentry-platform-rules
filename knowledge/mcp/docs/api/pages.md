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

## Positions inside a parent

Ordering among siblings uses a **lexorank string** on the parent-scoped operations, not a number. Never sort those values numerically, and never reorder by patching the field.

Find the dedicated position operation and use it — it is what renumbers the surrounding siblings correctly:

```text
cms_api_search { "query": "pages position" }
```

→ `mcp/docs/server/payload-conventions#position-is-a-lexorank-string-or-a-number`

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

A public read of a page's blocks is governed by the reading group's rules, and a read restriction caps the answer at a fixed number of records. That ceiling applies to the blocks of a single page exactly as it applies to a listing, so a page carrying more blocks than the ceiling comes back trimmed.

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

→ `mcp/docs/api/verification-recipes#pages`
