# Sensitive form fields

A form field can be marked sensitive, for a passport number, a diagnosis or a bank detail. The value is still accepted and kept with the submission. But no response returns it, except a submissions listing read by an admin who holds a dedicated permission, and every such read is recorded in the instance journal.

Read this when a submission comes back with `value: null` for a field the visitor filled in, or before you mark a field sensitive.

→ `mcp/docs/api/form-submissions` · `mcp/docs/api/attribute-sets` · `mcp/docs/api/journal`

## How to mark a field sensitive

Set `isSensitive: true` on the attribute in the schema of an attribute set whose type is `forForms`, the set a form takes its fields from. Read the set back afterwards: on sets of any other type the flag is dropped from the answer and has no effect.

```json
{ "attribute1": { "id": 1, "type": "string", "identifier": "passport",
  "position": 1, "isSensitive": true } }
```

The flag applies to submissions saved **after** it is set. A submission saved before keeps returning its value in every read. Switching the flag off later does not reveal submissions saved while it was on: they keep answering `value: null` as described below.

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

## Updating a submission without losing the value

An update replaces the field list of a submission. For a sensitive field, sending `value: null`, or leaving `value` out, keeps the value already stored. So a client can send back a submission exactly as it read it. Send a non-null value to replace the stored one.

## Sensitive fields are left out of search and filters

The value of a sensitive field is not used by semantic search over submissions. A text that only that field contains does not match. Submissions are not filtered or sorted by field values, so there is no filter to switch off.

## Common mistakes

- Marking the field on a set of a type other than `forForms` and expecting values to be hidden.
- Treating `value: null` as an empty answer and asking the visitor to submit again.
- Granting `forms.data.readSensitive` to every admin because a listing showed `null`.
- Expecting submissions saved before the flag was set to be hidden too.
