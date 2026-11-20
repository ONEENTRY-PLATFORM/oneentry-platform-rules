# Sensitive form fields

A form field can be marked sensitive, for a passport number, a diagnosis or a bank detail. The value is still accepted and kept with the submission. But no response returns it, except a submissions listing or an admin order read by an admin who holds a dedicated permission, and every such read is recorded in the instance journal. Orders whose storage uses the form, and the profiles of users who signed up through a form, are covered too.

Read this when a submission comes back with `value: null` for a field the visitor filled in, or before you mark a field sensitive.

→ `mcp/docs/api/form-submissions` · `mcp/docs/api/orders` · `mcp/docs/api/users-and-groups` · `mcp/docs/api/journal`

## How to mark a field sensitive

Set `isSensitive: true` on the attribute in the schema of an attribute set whose type is `forForms`, the set a form takes its fields from, or `forUsers`, the set of a sign-up form. Read the set back afterwards: on sets of any other type the flag is dropped from the answer and has no effect.

```json
{ "attribute1": { "id": 1, "type": "string", "identifier": "passport",
  "position": 1, "isSensitive": true } }
```

The flag also covers submissions saved **before** it was set. The schema update that switches it on hides that field in every stored submission of the forms built on the set, and answers only once they are all hidden. Switching the flag off later does not reveal anything: submissions saved while it was on keep answering `value: null` as described below.

→ `mcp/docs/api/attribute-sets#the-schema-key-is-attribute-and-the-id`

## Why a submitted value comes back as null

Every read of a sensitive field answers the field with `"value": null` and `"isSensitive": true`, beside the other fields of the submission, which are returned as usual. This covers:

- the answer to the submission itself;
- the public submissions listing and public semantic search;
- the admin submissions listing read without the permission below;
- the answer to an update of a submission.

A `null` here does not mean the visitor left the field empty, and resubmitting does not change it. Event notifications built from a submission, such as e-mails and workflow steps, also receive the field without its value.

## Who can read the value

`AdminFormDataController_findByFormMarker` (`POST /form-data/marker/{marker}`) returns the real value only to an admin holding `forms.data.readSensitive`, in addition to `forms.data.read`, which the route itself requires. Without the extra key the call still answers `200`, and the sensitive values are `null`. There is no `403` to tell you a key is missing.

The key is not granted to anyone by default, including the account that set up the instance. Grant it explicitly, and only to the admins who must see such data.

→ `mcp/docs/api/admins-and-permissions#a-key-can-exist-without-any-admin-holding-it`

## Every read of a value is journaled

Each listing call that returned at least one sensitive value adds one journal entry:

| Field | Value |
|---|---|
| `moduleName` | `FORM_DATA` |
| `action` | `READ` |
| `journalingEvent` | `formDataSensitiveViewed` |
| `delta` | `formIdentifier`, `formDataIds` (the submissions read) and `markers` (the sensitive fields read) |

The entry never contains the values. A listing that revealed nothing, such as a call without the key, adds no entry.

Writes are journaled too. Creating or updating a submission through the admin API adds an entry whose `delta` is the request body, and in it every item whose `marker` is a sensitive field carries `value: "[REDACTED]"`. The other fields keep their values. A marker counts as sensitive when any `forForms` set on the instance marks it, so a plain field that shares its marker with a sensitive field of another form is shown as `[REDACTED]` as well. The same holds for the `formData` of users and orders, and every `authData[].value` — the login and password sent when an admin creates a user — is always `[REDACTED]`.

## Sensitive fields in an order form

An order storage takes its fields from a form, so a sensitive field of that form is hidden in the `formData` of its orders in the same way:

- Creating or updating an order answers the field with `"value": null` and `"isSensitive": true`.
- Every order read answers it the same way: the public order listings and single-order reads, the admin listings of all orders, of one storage and of an interface module scope, `GET /orders-storage/{id}/order/{orderId}` and `GET /orders/{id}`. Event notifications and workflow steps built from an order also get the field without its value.
- The four admin reads return the real value to an admin holding `forms.data.readSensitive`. Each such read adds one journal entry with `moduleName` `ORDERS`, `action` `READ`, `journalingEvent` `formDataSensitiveViewed`, and a `delta` of `orderIds` and `markers`, without the values.
- An order update that sends the field with `value: null` keeps the stored value.
- An orders export has no column for a sensitive field.
- Switching the flag on also hides the field in orders placed earlier.

## Sensitive fields in a user profile

The form of an auth provider takes its fields from a `forUsers` set, so a sensitive field of that set is hidden in the `formData` of the users who signed up through it:

- `GET /users/me` returns the user's own values in full. The sign-up answer returns the field with `value: null`.
- Admin user reads (`GET /users`, `GET /users/{id}`, `POST /users/all`, `POST /users/paginated`, `POST /users/search`) answer `value: null` with `isSensitive: true`. When the field is the notification e-mail or SMS field of the form, `notificationData.email` or `notificationData.phoneSMS` is `null` as well.
- An admin holding `forms.data.readSensitive` gets the values. Each such read adds a journal entry with `moduleName` `USER`, `action` `READ`, `journalingEvent` `formDataSensitiveViewed` and a `delta` of `userIds` and `markers`, without values.
- `PUT /users/{id}` with the field at `value: null` keeps the stored value, so a card can be saved back as read.
- The field may be the login field or a notification field. Sign-in and notifications keep working, but the login itself is the user's `identifier` and is shown and searchable as before.
- `POST /users/all` and interface module scopes answer `400` for a filter on a sensitive field. The users export has no column for it.
- Users imported into the provider get the field hidden in the same way, and switching the flag on hides it for users who signed up earlier.

## Updating a submission without losing the value

An update replaces the field list of a submission, and the two update routes treat a sensitive field differently.

- Admin API, `PUT /api/admin/form-data/{id}`: sending `value: null` keeps the value already stored, so the submission can be sent back exactly as it was read.
- Content API, `PUT /api/content/form-data/{id}`: the visitor's edit is checked like a new submission. `value: null` answers `400` saying the value must be a string, and an entry with no `value` answers `400`, whether or not the field is required. Leaving the entry out answers `400` for a required field, and for an optional one it is accepted and erases the stored value.

So a site that lets a visitor edit a submission must ask for the sensitive value again and send it. Echoing the read back fails, and dropping the field loses it.

## Sensitive fields are left out of search and filters

The value of a sensitive field is not used by semantic search over submissions. A text that only that field contains does not match. Submissions are not filtered or sorted by field values, so there is no filter to switch off.

## Common mistakes

- Marking the field on a set of a type other than `forForms` or `forUsers` and expecting values to be hidden.
- Expecting a sensitive login field to hide the user's `identifier`.
- Treating `value: null` as an empty answer and asking the visitor to submit again.
- Granting `forms.data.readSensitive` to every admin because a listing showed `null`.
- Switching the flag off to read a value back. Use the permission below instead.
- Reading a submission's value from the journal. Entries show `[REDACTED]`; read the submission with the permission.
