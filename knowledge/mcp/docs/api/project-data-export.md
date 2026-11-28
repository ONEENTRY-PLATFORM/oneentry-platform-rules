# Exporting all project data to a ZIP archive

How an admin takes every piece of project data out of the instance in one archive of CSV files plus the uploaded files, in a form another system can import. The archive carries business fields and links between rows, never storage details.

Read this when someone asks to download, back up or hand over all the data of a project, or when a project export answers `409`, `410` or `429`.

→ `mcp/docs/api/admins-and-permissions#exporting-data-has-its-own-permissions` · `mcp/docs/api/admin-permission-catalog#which-permissions-count-as-sensitive`

## Which permission starts a project export

Every project export operation needs `project.export`. The key is sensitive: a map of "every non-sensitive key" does not include it, and the project owner holds it out of the box. Without it each call answers `403` and names the key. Grant it only when a person explicitly asks: the archive contains every section, users and orders included.

## How to run a project export from start to download

The archive is built in the background, so the export is three calls, never one.

1. `AdminProjectExportController_request` (`POST /api/admin/export/project`, no body) answers `202` with the export: `id`, `status: "waiting"`, `requestedAt`.
2. Poll `AdminProjectExportController_get` (`GET /api/admin/export/project/{id}`) every few seconds until `status` is `completed` or `failed`. `status` takes `waiting`, `active`, `completed`, `failed` or `expired`. A completed export also reports `finishedAt`, `size` in bytes, `expiresAt`, and `files` (each file name with its row count).
3. `AdminProjectExportController_download` (`GET /api/admin/export/project/{id}/download`) returns the ZIP as `application/zip` with the file name in `Content-Disposition`.

`AdminProjectExportController_list` (`GET /api/admin/export/project`) lists the calling admin's exports, newest first. Use it to find an export you started earlier instead of starting a new one.

The admin panel socket may also send a `projectExport` event with `action` `completed` or `failed`. Do not wait for it alone: polling is the reliable signal.

## Why a project export answers 409, 410, 429 or 404

| Status | When | What to do |
|---|---|---|
| `409` on request | another export of this project is still running | wait for it, poll it, then download it |
| `429` on request | less than an hour since the last export started; `retryAfter` gives the seconds left | download the previous archive, or retry after `retryAfter` |
| `409` on download | the archive is not ready yet | keep polling until `completed` |
| `410` on download | the archive is older than 24 hours and was deleted | start a new export |
| `404` | the export belongs to another admin or another data environment, or does not exist | only the admin who requested an export can read or download it |

## What the project export archive contains

- One CSV file per kind of data: pages, products, blocks, forms and their submissions, users, orders and their items, payments, discounts, menus and menu items, collections and their rows, templates, attribute sets and attributes, languages, settings.
- `README.csv` lists every file and every column with its meaning, the export date and the format version.
- `files/` holds the uploaded files and images; a CSV cell that refers to a file holds its path inside the archive, such as `files/<name>`.

Files are UTF-8 with a byte-order mark, so spreadsheet programs keep non-Latin text. A cell that starts with `=`, `+`, `-` or `@` is prefixed with `'`, so it opens as text, not as a formula.

## How rows of a project export link to each other

Every row has a `ref` column, an opaque id made for the export. Other files point at it: `parent` of a page or menu item, `categories` of a product, `pages` and `products` of a block, `form` and `user` of a submission, `user` of an order, `order` and `product` of an order item, `order` of a payment, `menu` and `page` of a menu item. A list of refs is separated by `;`.

A `ref` stays the same between exports of an unchanged project, so two archives can be compared row by row. Internal ids never appear.

Values in several languages are separate columns suffixed with the language code: `title_en_US`, `title_de_DE`. Attribute values are `attr_<marker>_<language>`, form fields `field_<marker>_<language>`. Only active languages get columns.

## What a project export leaves out

Password hashes, tokens, API keys, payment and sign-in provider secrets, payment session links, and attributes or form fields marked as sensitive or as passwords are never exported. Fields and attributes hidden from the requesting admin by field rules are left out of that admin's archive.
