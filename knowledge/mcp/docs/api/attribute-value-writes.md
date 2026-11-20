# Attribute value writes

`PUT /api/admin/system/update-attribute-values` changes the value of one or more attributes on one record without sending the rest of it, and `PUT /api/admin/system/update-multiple-attribute-values` does the same for several records in one call. Both are permanently confirm-gated by this server: dry-run, show the target, then repeat with the token.

Read this before changing a single value on a product, an order, a user or a form submission, or when a value write answers `400`.

→ `mcp/docs/server/confirm-and-dry-run` · `mcp/operating-rules#attribute-values-are-two-level-and-locale-keyed`

## What one value write looks like

```json
{ "type": "products", "id": 101, "attributeSetIdentifier": "dish",
  "attributeValues": { "en_US": [{ "marker": "weight", "type": "integer", "value": "280" }] } }
```

`type` names the kind of record, `id` the record, and `attributeSetIdentifier` the marker of the attribute set the attribute belongs to. `attributeValues` is keyed by locale and lists each changed attribute with its `marker`, `type` and `value`. Attributes you leave out keep their values. The call answers `true`.

## Writing form values of orders users and submissions

`type` also accepts `orders`, `users` and `form_data`. For those the value is written into the record's `formData`, and `attributeSetIdentifier` is the set of the form the record was made with — not a set you choose.

Read that marker before writing: orders, users and submissions return it as `attributeSetIdentifier`, both in the user list and in a related-data plan.

→ `mcp/docs/api/related-data#attribute-values-in-a-plan`

## A user login cannot be written here

The attribute a user signs in with is refused: the write answers `400` saying the login field is changed by `PUT /users/{id}`. Change it through the user update, where the sign-in rules apply.

→ `mcp/docs/api/users-and-groups#the-islogin-field-is-what-a-user-signs-in-with`

## A batch write is all or nothing

`update-multiple-attribute-values` takes `{ "items": [ … ] }`, each item shaped like a single write. Every item is checked before anything is written: when one is refused the call answers `400` naming it, and no item is written — including the valid ones. Fix the item and send the whole batch again.

On success it answers one entry per item, echoing it with `updated: true`.

## A value write moves the product revision

Writing a value of a product counts as a change to it, so the product's `revision` grows. A product update sent afterwards with the `X-Product-Revision` you read before the value write answers `409` and writes nothing. Re-read the product and send its new `revision`.

→ `mcp/docs/api/products#update-a-product`

## When the new value can be read

Once the write has answered, a related-data plan over the record already returns the new value. Confirm a write that way rather than repeating it.

## Common mistakes

- **Choosing the attribute set for an order or a user.** It is the set of the record's form; read `attributeSetIdentifier`.
- **Changing a sign-in login here.** It answers `400`; use the user update.
- **Retrying the valid half of a refused batch.** Nothing was written; resend the whole batch.
- **Updating a product with the revision read before a value write.** It answers `409`.

→ `mcp/docs/api/attribute-sets#two-levels-always` · `mcp/docs/api/forms-and-form-data#fields-are-attributes`
