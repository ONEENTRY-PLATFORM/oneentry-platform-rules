# Menu write checks

What a menu create, update or delete refuses, and with which status. Read it before writing a menu identifier, or when a menu write answers 400, 404 or 409.

The composition and the tree itself are described in the main menus document.

→ `mcp/docs/api/menus` · `mcp/docs/server/payload-conventions#marker-is-portable-id-is-not`

## Which identifier a menu accepts

A menu identifier is the marker a site reads the menu by, so it must be one the public read accepts:

- a letter first, then letters, digits, `_` or `-` — no spaces, dots, slashes or non-Latin letters;
- a string, not a number;
- at most 255 characters.

Anything else answers `400` naming `identifier`, and no menu is created. Use the marker validation operation of the menus area to check a candidate before writing it.

## Why a menu create or rename answers 409

The identifier is unique across menus. A create with an identifier another menu already holds, or an update that renames a menu to one, answers `409` and changes nothing. Read the list of menus first and pick a free identifier; do not retry the same body.

## Renaming a menu that has an older identifier

An update checks the identifier format only when the identifier **changes**. A menu stored earlier with an identifier outside the format stays editable: send its identifier back unchanged with the new title and the update succeeds. Renaming it is the only way to make it readable by a site, and it breaks whatever site code reads the old marker — treat that as the owner's decision.

## Null is not the same as leaving a field out

On an update, a field left out of the body is left as it is. `identifier: null` or `localizeInfos: null` is not "leave it": it answers `400`. Omit the key to keep the stored value.

## Why a menu update or delete answers 404

- An update or delete of a menu id that does not exist answers `404`, the same as reading it.
- An update whose `pagesIds` names a page that does not exist answers `404` naming that page, and the menu keeps its previous composition. This is the same check a create makes.

## Repeated page ids in pagesIds

`pagesIds` is a set: a page listed twice becomes one item, in the place of its first mention. To show one page in two places, add a custom item with the same target for the second one.

→ `mcp/docs/api/menus#building-a-whole-menu`
