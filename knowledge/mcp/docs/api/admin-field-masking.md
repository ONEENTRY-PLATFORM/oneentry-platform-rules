# Admin field masking

A field rule can show an admin part of a value instead of the whole value: the last digits of a phone, the start of an e-mail, the end of a document number. The value is masked in every Admin API response, so the full value never reaches that admin's browser. A separate permission lets an admin view masked values in full, and every such view is journaled.

Read this when you set a masked rule in `fields.access`, give an attribute a mask template, or see `*` characters in a value an admin reads.

→ `mcp/docs/api/admin-record-scope-and-field-access#making-a-field-or-attribute-read-only` · `mcp/docs/api/admin-permission-catalog#which-permissions-count-as-sensitive`

## How to mask a field or attribute

`masked` is a third level of a `fields.access` rule, next to `read` and `hidden`. It works for area fields and for attributes, including `*` for a whole attribute set.

```json
{
  "fields.access": {
    "users": { "email": "masked:email" },
    "attributes": { "contacts": { "phone": "masked", "passport": "hidden" } }
  }
}
```

- `masked` alone uses the attribute's own mask template, or the default when it has none.
- `masked:phone`, `masked:email` and `masked:document` pick a template in the rule itself. Any other suffix is refused with `400`.
- A masked value cannot be changed, as with `read`.
- `masked` is stricter than `read` and looser than `hidden`. An author with a masked rule can pass on `masked` or `hidden` only.

## What a masked value looks like

| Template | Example |
|---|---|
| `phone` | `+7 *** ***-12-34`: the first and the last four digits stay, separators stay |
| `email` | `iv***@mail.com`: two characters of the name and the domain stay |
| `document` | `**** **3456`: the last four characters stay |
| default | the last four characters stay |

A value no longer than the part that stays is masked whole. A compound value is masked in every text part. Items of `attributeValues` and `formData` keep their marker and type, and only `value` is masked.

## Giving an attribute a mask template

An attribute in an attribute set schema can carry `mask`:

```json
{ "mask": { "preset": "phone", "keepStart": 1, "keepEnd": 4, "char": "*" } }
```

Every key is optional. `keepStart` and `keepEnd` are whole numbers from 0 to 64, and `char` is one character. Any other key or value makes the schema write answer `400`. A template in the rule wins over the attribute's template.

## Sending back a masked value

Send back what you read. A masked value that comes back unchanged keeps the stored value, and a masked value left out of the body stays as stored. Any other value is a change and answers `403` with `FIELD_DENIED`.

A version rollback is judged against the stored values, not against what the admin saw. A rollback that would change a masked value answers `403` with `FIELD_DENIED`.

## Filtering and searching by a masked value

A masked value counts as hidden for filters, sorting and search. A list filtered or sorted by it answers `403` with `FIELD_DENIED`, and search never matches on it. Matching on it would reveal the value one guess at a time. Reading a single masked attribute value through the attribute value operation also answers `403`.

## Viewing masked values in full

An admin holding `fields.viewUnmasked` sends the request header `X-Unmask-Fields: true` to receive masked values in full. Hidden values stay hidden.

- Without the permission the header changes nothing, and values stay masked.
- With the permission and no header, values are masked as usual.
- Each response that revealed something adds an action journal entry with `journalingEvent: fieldsUnmasked`. Its `delta` holds the method, the path and the names of the revealed fields, never the values.
- Exports are always masked.
- An admin with the permission may send back the revealed values unchanged in a write.

`fields.viewUnmasked` is a sensitive permission, so full rights do not include it by default.

## Common mistakes

- **Writing the `*` characters back as a new value.** It is read as unchanged only when it equals the masked value you received. Edit other fields and leave the masked one as it came.
- **Expecting `X-Unmask-Fields` to work alone.** It needs `fields.viewUnmasked`.
- **Filtering by a masked attribute.** Filter by a field the account may read in full.

→ `mcp/docs/api/journal` · `mcp/docs/api/admin-record-scope-and-field-access#what-the-refusal-codes-tell-you`
