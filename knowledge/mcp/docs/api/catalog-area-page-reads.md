# Reading the catalogue tree with the catalog area only

An admin who is given the `catalog` area in `admins.modules`, but not `pages.get`, can still read the pages that make up the catalogue tree. Every other page read keeps requiring `pages.get`.

Read it before setting up a catalogue editor who must not see the rest of the site, and when such an account gets `403` on a page read.

→ `mcp/docs/api/admins-and-permissions#reading-admin-data-needs-a-permission-too` · `mcp/docs/api/pages#why-a-catalog-page-filter-returns-plain-pages`

## Which page reads the catalog area opens

With `"admins.modules": ["catalog"]` and no `pages.get`, two reads answer `200`:

| Read | Allowed when |
|---|---|
| `POST /pages/paginated` | the body filters on `type: "catalog_page"`, or on `types` that list only `catalog_page` |
| `GET /pages/{id}` | the page is a catalog page, or a plain page that has a catalog page somewhere below it |

These are exactly the rows a `catalog_page` listing returns, so a catalogue editor can walk the tree level by level with `parentId` and open any row it was shown.

The area grants reading only. Writing products and categories still needs the matching `catalog.*` keys.

## What stays behind pages.get

Without `pages.get`, the account gets `403 Forbidden resource` on:

- a paginated listing with no type filter, or filtered on any other type;
- reading a page by id that is outside the catalogue tree;
- the root listing, the children listing, a page's blocks, and page search.

Holding `pages.get` opens all of these as before, whatever `admins.modules` contains. Removing `catalog` from `admins.modules` closes the catalogue reads on the next request; the account does not need to sign in again.

## Setting up a catalogue editor

Grant the area and the write keys the editor needs, and leave `pages.get` out:

```json
{
  "admins.modules": ["catalog"],
  "catalog.products.create": true,
  "catalog.products.update": true
}
```

Adding `pages.get` to make the catalogue appear is not needed, and it exposes every page of the site to that account.
