# Admin record scope and field access

Two kinds of permission narrow what an admin account may touch. A record scope confines it to chosen records of one area. A field rule makes chosen fields or attributes read-only or invisible. Both live in the same permission map as every other key.

Read this before you delegate part of a catalogue, a set of forms or a few discounts to someone, and when a write answers `403` with `SCOPE_DENIED` or `FIELD_DENIED`.

→ `mcp/docs/api/admin-page-scope#the-value-is-a-list-of-page-ids-not-a-boolean` · `mcp/docs/api/admin-permission-catalog#a-403-names-the-missing-permission`

## Which records an admin may touch

Each scope key takes a list of record ids. An account without the key reaches every record of that area, as before.

| Key | Records | Shape |
|---|---|---|
| `pages.scope` | pages | each listed page and everything below it |
| `catalog.scope` | catalogue categories | each listed category and everything below it |
| `forms.scope` | forms and their submissions | the listed forms |
| `blocks.scope`, `discounts.scope`, `menus.scope`, `templates.scope`, `collections.scope` | records of that area | the listed records |
| `orders.storage.scope` | order storages and their orders | the listed storages |
| `userGroups.scope` | user groups | the listed groups |

The value follows the `pages.scope` rules: a non-empty list of whole positive ids, or `false` for no restriction. `[]` and `true` are refused with `400`.

A product is in scope when at least one of its categories is inside `catalog.scope`. A product with no category is outside it.

## What a scope changes for reads and writes

A scope governs reads as well as writes.

- Lists return only records in scope, and their totals count only those.
- Reading, changing or deleting one record outside the scope answers `403`.
- A record that does not exist answers the same `403`, so a refusal is not proof the record exists.
- A bulk operation with one id outside the scope is refused whole, and nothing is changed.
- A scoped account cannot create a new record in an area with a flat list, because the new record cannot be in its list. Pages and categories can be created under a parent inside the scope.
- Creating a product, or changing its categories, needs every category it is placed in to be inside the scope.

For `pages.scope`, the list of root pages becomes the pages named in the scope, so the tree still opens at the branches the account owns.

## Making a field or attribute read-only

`fields.access` is a list of restrictions. Anything it does not name stays fully editable.

```json
{
  "fields.access": {
    "discounts": { "startDate": "read", "endDate": "read" },
    "attributes": {
      "price_set": { "*": "read", "purchase_price": "hidden" }
    }
  }
}
```

- `read` shows the value but refuses a change to it. `hidden` leaves the value out of responses and refuses a change to it.
- Area keys and field names come from `fields` in the permission catalog. A name not listed there is refused with `400`.
- Attributes are named by the marker of their attribute set and the marker of the attribute. `*` stands for every attribute of the set, and a rule on one attribute wins over `*`.
- An attribute set or attribute marker that does not exist is refused with `400`, so a typo is never stored as a rule that does nothing.
- Attribute rules also cover submitted form values: the `formData` items of users, form submissions, orders and integration collection rows. Each item is judged by its `marker` against the attribute set of the record's form (the auth provider's form for a user, the storage form for an order, the collection form for a row). A hidden item is left out of every response; changing a read-only item answers `403` with `FIELD_DENIED`; a hidden item you never received stays as stored when you update the record without it.
- Send `false`, or leave the key out, to lift every field rule.

→ `mcp/docs/api/admin-permission-catalog#what-the-permission-catalog-returns`

## Sending back a value you may only read

A write is judged by what it would change, not by what it carries.

- A read-only field sent back with the value it already has is accepted. Numbers sent as strings, dates in another format and objects with keys in another order count as the same value.
- A read-only field that is left out of the body is accepted, and the stored value stays.
- A hidden attribute you never received stays as stored when you update the record without it. You do not have to, and cannot, send it back.
- Any real change to a restricted field refuses the whole write, and nothing is saved.

So the safe way to update a record under field rules is to send back what you read, changing only the fields you may edit.

## Filtering by an attribute you cannot see

A list filtered or sorted by a hidden attribute answers `403` with `FIELD_DENIED`. Matching on a value you cannot read would reveal it.

Search results never match on a hidden attribute value either: a record found only through such a value is left out.

## Searching by a field you cannot see

A hidden field is not searchable, because a match would confirm its value.

- The quick search of products, pages and blocks matches on the title only. With `localizeInfos` hidden for that area it answers `403` with `FIELD_DENIED`; use the paginated list instead.
- The global search does not match on a hidden title, admin e-mail or page URL, and leaves that value out of the item. An item whose title is hidden carries the record's marker as `title`.
- Filter items whose record has a hidden title come back with `localizeInfos: null`.

→ `mcp/docs/api/global-search`

## Reading the action journal under field rules

`GET /api/admin/journal` (`JournalController_list`) applies the reader's field rules to the `delta` of every entry, the same way reading the record itself does.

- The area of an entry follows its `journalingEvent`: a `pageUpdated` entry drops the fields hidden for pages, an `adminUpdated` entry drops those hidden for admins. Entries of an area without field rules keep every field.
- Hidden attributes and hidden `formData` items are left out of `delta`.
- When the reader has attribute rules and a `delta` carries attribute values without naming their attribute set, or `formData` without naming its form, those values are left out whole.

So a missing field in a journal `delta` can mean the reader may not see it, not that the request omitted it. Read the record by id with an account that may see the field to confirm.

## What the refusal codes tell you

| `code` | Read it as | What to do |
|---|---|---|
| `SCOPE_DENIED` | The record, or one of `ids`, is outside the scope named in `permission` | Work on a record inside the scope, or ask for it to be added |
| `FIELD_DENIED` | The request changes, filters or searches by one of `fields`, which the account may not use that way | Leave those fields out of the change or the query, or ask for the rule to be lifted |

`fields` names attributes as `attributes.<set marker>.<attribute marker>`.

Neither code means a key is missing, so asking for the key in `permission` changes nothing.

## Who can grant a scope or a field rule

An account can only pass on restrictions at least as tight as its own.

- An account with a scope can give another account only records inside that scope. It cannot remove the key from someone else's map, because no key means every record.
- An account with field rules can give another account only the same rules or stricter ones. `hidden` is stricter than `read`.
- A new account created by a restricted author must carry those restrictions from the start.

A refusal here is a `403` naming the key, without a `code`.

## Scopes inside an environment

Scope ids are ids from the main data set, as for `pages.scope`. In a request that names an environment, each id stands for the environment's copy of that record.

→ `mcp/docs/api/admin-page-scope#the-scope-inside-an-environment` · `mcp/docs/api/environments#copying-content-from-one-environment-to-another`

## Common mistakes

- **Granting a scope with `true` or `[]`.** Both are `400`. Send the list, or `false` to lift it.
- **Expecting a scoped account to create flat records.** It cannot; create them with an unscoped account and add the ids.
- **Sending a hidden attribute as empty to clear it.** That is a change, and it is refused.
- **Retrying a `SCOPE_DENIED` bulk call id by id.** Nothing was changed; remove the ids outside the scope.

→ `mcp/docs/api/admins-and-permissions#never-replace-an-admins-permission-map` · `mcp/docs/api/admin-permission-catalog#which-permissions-count-as-sensitive`
