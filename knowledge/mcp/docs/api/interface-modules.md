# Interface modules

An interface module is a saved setup of the admin panel: a simplified copy of one section — catalogue, pages, orders, payments or users — shown in the panel menu as its own top-level item, and only to the admins it lists.

Read this before creating or changing one, or when an admin asks why a module shows them less than the section it copies. What each base keeps in `config.scope` and how to read a module's data live in their own documents.

→ `mcp/docs/api/interface-module-scopes#what-the-scope-of-each-base-holds` · `mcp/docs/api/interface-module-data#reading-what-a-module-shows`

## What an interface module is and what it is not

A module runs nothing, holds no content and restricts nothing. Every operation returns the same data whether or not the admin reached it through a module, and the usual permissions apply. An admin who should not see an order must be kept from it by permissions, never by leaving it out of a module.

Interface modules are not rows of the module listing. That listing carries one section, `interface-modules`, and the modules themselves are managed through their own operations.

→ `mcp/docs/api/modules#the-modules-you-will-find`

## Which permission keys manage interface modules

| Operation | Permission |
|---|---|
| `GET /api/admin/interface-modules`, `GET /api/admin/interface-modules/{id}` | `interfaceModules.get` |
| `POST /api/admin/interface-modules` | `interfaceModules.create` |
| `PUT /api/admin/interface-modules/{id}` | `interfaceModules.update` |
| `DELETE /api/admin/interface-modules/{id}` | `interfaceModules.delete` |
| `GET /api/admin/interface-modules/my` | none |

`/my` returns only the modules whose `adminIds` include the caller, without `adminIds`, and `[]` when there are none. Every admin sees only their own modules there, the first administrator included; the full list needs `interfaceModules.get`.

## Rules for the identifier and the title

- `identifier` matches `^[A-Za-z0-9-]+$`. An underscore answers `400`.
- It must not collide, ignoring case, with another module or with an admin section identifier such as `catalog`, `content` or `orders`. A collision answers `400` saying a module or section with that identifier exists.
- It cannot change: sending a different `identifier` to the update answers `400`. To rename, create a new module and delete the old one.
- `localizeInfos` needs a non-empty `title` in at least one language, or the create answers `400`.

```json
{ "identifier": "warehouse", "localizeInfos": { "en_US": { "title": "Warehouse" } },
  "adminIds": [12], "config": { "base": "orders", "scope": { "storageIds": [2] } } }
```

## An update replaces config whole

`PUT` takes `config` as a whole object, never a patch. Keys you leave out of the new `config` are gone after the write — a module that had a `scope.filter` loses it when you send a `scope` without one. Read the module, change the field, send the whole `config` back, then read it again.

`config` is checked strictly:

- `base` is one of `catalog`, `pages`, `orders`, `payments`, `users`; anything else answers `400` listing them.
- An unknown key answers `400` naming it, and so does a key that belongs to another base — `pageDefaults` on a catalogue module, for instance.
- The whole `config` is limited to 256 KB.
- `screens` is optional on every base: an object of objects that the panel owns. Leave it as you read it.

## Settings that only change what the panel offers

These keys never restrict an operation; the product, page, order, payment and user operations do not read them. They decide what the panel pre-fills or offers inside the module.

| Base | Keys |
|---|---|
| `catalog` | `productCategories`, `productDefaults`, `productStatusIds` |
| `pages` | `pageDefaults`, `pageCreateTypes`, `pageDeleteFlags` |
| `orders` | `orderStatusIdentifiers` |
| `payments` | `orderLinkModuleId` |
| `users` | `newUserGroupIds` |

So an agent creating a product for a module's admin must set the status, template and attribute set itself; the module's `productDefaults` will not be applied by the create.

→ `mcp/docs/api/interface-module-scopes#panel-settings-of-each-base`

## Screen templates for a module

A template or an assignment for a module's screen uses the key `im:<module identifier>:<base screen>`, for example `im:warehouse:orders-table`, with the usual template and assignment operations. The template belongs to that module only, and `configId` is stored and returned as sent.

The config is checked against the base screen (`orders-table` here): the recipes whose scope lists that screen, its navigation routes, and whether attribute columns may be `editable` there. A config is accepted or refused exactly as it would be on the base screen, with the same `400` message.

Whether a module with that identifier exists is not checked. A template saved for a misspelled or deleted module is accepted and never shown, so read the module list before saving one.

→ `mcp/docs/api/ui-component-configs#screens-templates-and-recipes`

## Common mistakes

- **Treating a module as an access boundary.** It hides nothing from the API; use permissions.
- **Sending a partial `config` to the update.** Everything you leave out is removed.
- **Trying to rename through the update.** `identifier` is fixed; the update answers `400`.
- **Expecting `productDefaults` or `newUserGroupIds` to apply to an API write.** Only the panel reads them.
- **Looking for interface modules in the module listing.** Only the `interface-modules` section is there.
- **Reading `/my` as the full list.** It returns only the caller's modules.

→ `mcp/docs/api/interface-module-data#an-empty-filter-answers-400` · `mcp/docs/api/admins-and-permissions`
