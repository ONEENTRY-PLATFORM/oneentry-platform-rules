# Forms and form data

A form is a definition — its fields, their types, how submissions are handled. Form data is what visitors submitted against it. Two entities, two areas of the API.

The create payload has a shape the API document does not show, so read the first section before writing one.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/attribute-types`

## Create a form with a wrapped body

Send the payload nested under `newForm`, and always include `type`:

```json
{ "newForm": { "identifier": "contact-us",
               "type": "data",
               "processingType": "script",
               "localizeInfos": { "en_US": { "title": "Contact us" } } } }
```

The API document shows the fields unwrapped. Every instance accepts the wrapped body. Some also accept the flat one, and others answer 5xx to it, so the wrapped body is the one to send.

→ `mcp/operating-rules#operations-with-a-single-supported-path`

## What a form is made of

- an **identifier** — the marker a site uses to fetch the form;
- **`localizeInfos`** — title and description per locale;
- **fields**, defined as attributes drawn from the same closed type list as everywhere else;
- a **form type** and a **processing type** — what kind of form it is, what happens to a submission;
- **module configs** — the bindings naming which module and entities the form belongs to, who may submit and how ratings behave. A form without one accepts nothing; see below.

## Always send the form type

`type` has a closed set of values. Some instances list it in the create schema and refuse an unknown value with 400. Others leave it out of the schema, accept it anyway, and create a form with `type: null` when it is omitted — and nothing says so. Send it on every create.

| value | what it is for |
|---|---|
| `data` | a plain data-collection form — a contact form is this one |
| `order` | a checkout or ordering form |
| `sign_in_up` | authentication: sign-in and registration |
| `collection` | a form that gathers entries into a collection |
| `rating` | a rating or review form, with rating behaviour attached — see `mcp/docs/api/rating-forms-and-reviews` |

Copy the values exactly. Where `sign_in_up` is rejected as unknown, the instance still carries the historical misspelling `sing_in_up` — report that rather than guessing.

`type` is accepted on update, so a typeless form is corrected without recreating it. Read the form back by identifier and check the field: a wrong or absent `type` is never reported.

## Fields are attributes

A form's fields are attributes, with the same nineteen types and value shapes as elsewhere, so everything in `mcp/docs/api/attribute-types` applies to submissions: numbers may be `null`, choice fields store option ids, file fields change shape with the count. A captcha field does carry a value, and a submission that omits it is refused — see `mcp/docs/api/captcha-fields`.

The one type written differently here than on an entity is `text`: a submitted value is a list of **one object carrying exactly one** of `plainValue`, `htmlValue` or `mdValue`, with no other key beside it. A bare string, a list of strings, `[{ "value": … }]`, two of the three keys together and the four-key shape an entity attribute uses are each refused.

```json
{ "formData": { "en_US": [ { "marker": "message",
                             "value": [ { "plainValue": "Please call me back." } ] } ] } }
```

→ `mcp/docs/api/attribute-types#text-is-written-one-way-on-an-entity-and-another-in-a-form`

## Forms are dynamic never hardcoded

A site renders a form from its definition — field markers, types and labels read from the API — not from hand-written markup, which stops matching the moment someone edits the form. So "add a field to the contact form" is a change to the definition, not to any code.

## A form must be bound before it accepts submissions

A form on its own is a definition. What lets a submission exist is a **module config** — the object binding the form to a module (pages, products, users…) and to entities inside it. Every submission names the config it was made against, so a form with no config accepts nothing and the failure is a flat `400`.

Creating a form does not create one: the create operation has no field for it, so a fresh form has none and the first submission fails. Say that in your plan rather than discovering it later. The config comes from the **form update**, which carries a `formModuleConfigs` array the create body does not have:

```json
{ "formModuleConfigs": [ { "formId": 5,
                           "moduleId": 4,
                           "isGlobal": false,
                           "entityIdentifiers": [ { "id": "contacts-page", "isNumeric": false, "childrenOn": false, "isNested": false } ],
                           "isClosed": false,
                           "isModerate": false,
                           "viewOnlyUserData": true,
                           "commentOnlyUserData": false } ] }
```

- `formId` is the form being updated, `moduleId` the module — read the modules listing for the id.
- `isGlobal: true` with an empty `entityIdentifiers` binds the form to every entity of the module; otherwise list them. An entry is `{ "id": … }` plus two optional keys: `isNested: true` extends it to the entity's children, and `exceptionIds` lists children left out of that. `id` takes a textual identifier or a numeric one, whichever the module addresses its entities by — nothing else describes the kind.
- The read returns the entries in exactly this shape, so the list you read is the list you send back. Keys outside those three are stored and never read: an entry that names children through anything but `isNested` binds the parent alone, and nothing reports it.
- An entry with no `id` is a **new** binding. An existing one is matched by its own `id` and by nothing else — same module, same entities, no `id`, and you have replaced it rather than edited it.

So the path from nothing to a stored submission is four calls: **create** the form wrapped under `newForm` with an explicit `type`; **update** it with `formModuleConfigs`, the step with no operation of its own that people miss; **read it back** and take `formModuleConfigs[].id`, which is the only way to learn the `formModuleConfigId`; **create the submission**, naming both the form and that config.

A submission body needs `formIdentifier` (the form's marker), `formModuleConfigId`, `moduleEntityIdentifier` (the entity the submission is *for* — a textual identifier, a numeric id for products, a login for users and admins) and the locale-keyed `formData`.

**`formIdentifier` and `formModuleConfigId` must describe the same form.** They are compared, so a mismatch — or a config id matching nothing, which is what an unbound form gives you — answers `400 Incorrect formIdentifier for provided config`. The message names `formIdentifier`; the wrong one is usually the config id.

## Send status only as one of its five values

`status` is optional on a submission and it is not free text. Send it only as one of `sent`, `moderation`, `approved`, `banned`, `deleted`, lowercase. Any other value is refused with a `400` naming `status` and listing the five it accepts.

The value that looks safe and is not is the **empty string**: there is no "no status yet" value to send. Leave the key out instead, and the form's own moderation setting decides what the stored submission starts as. A submission sent through a client library that types the field as a plain string will compile with `""` in it; that is the client's typing being wider than the API, not a value the API takes.

→ `mcp/docs/api/form-submissions#submission-status`

## An update drops every config you do not send back

`formModuleConfigs` is a full replacement list, not a patch. An update that omits it is read as "this form has no configs": every binding is deleted, **and the submissions recorded against them go too**, and the call answers `200 true`. That makes an ordinary edit — a retitle, a processing-type change — destructive by omission. Read the form first and send its current `formModuleConfigs` back unless changing them is the point.

Sending them back **without their `id`** costs the same. The entry becomes a new binding, the previous one goes with the submissions counted against it, and the call answers `200` — the form still looks bound:

```text
read the form      → formModuleConfigs[0].id = 9,  count for the marker = 4
update, no id sent → 200
read the form      → formModuleConfigs[0].id = 10, count for the marker = 0
```

A public read of the entity keeps naming the previous binding while its answer is still current, so the site's own submissions answer `400 Incorrect formIdentifier for provided config` for a while afterwards rather than at the moment of the edit — which is what makes it hard to connect to it.

Keep each entry's `id`, and read back: same ids, same count.

Where a shortened list would take submissions with it, the call answers `409` instead, naming how many submissions and which bindings, and changes nothing. Add `confirmFormDataDeletion: true` to the same body to go through — it is the only way to delete a binding that holds submissions, and an edit that echoes every binding back with its `id` never meets it. Dropping a binding with no submissions still answers `200` either way, and so does every call on an instance without the gate: the status tells you the gate is there, never that the submissions were safe.

→ `mcp/docs/api/silent-no-ops` · `mcp/docs/api/content-api-reads#why-a-public-read-still-shows-the-previous-value`

## The order the errors arrive in

The checks on a submission do not run in the order the payload is written, and the first error you see is not the only thing wrong. In order:

| # | check | what it says when it fails |
|---|---|---|
| 1 | payload shape, locales, attribute markers | the marker or locale that does not belong to the form |
| 2 | required fields | `required values are missing or incorrect: <marker>` |
| 3 | field types, lists, validators, captcha | the offending field |
| 4 | the form's `type` | `Form has incorrect type: <type>` |
| 5 | the config, against `formIdentifier` | `Incorrect formIdentifier for provided config` |

Steps 1–3 are field validation, steps 4–5 are the form itself, so **field errors always arrive before the config error**. Fixing the field makes the same request fail on the config, which reads as the config becoming a problem afterwards. It did not: it was always required, behind a check that failed earlier.

Two consequences worth planning around:

- **A typeless form can never accept a submission.** Step 4 rejects anything that is not `data` or `rating`, and a form created without `type` is `null` — the create-schema trap, surfacing as a submission failure.
- **On the admin API, steps 1–3 do not run.** Field validation belongs to the visitor route, so a missing required field is stored rather than rejected and the config check is the first thing you meet. A `400` there is about the binding, and an accepted submission is no evidence that a visitor's would pass.

## An empty value is not the same as no value

A field that carries no `requiredValidator` can be left out of the submission entirely, and that is accepted. Sending the same marker with `""` is treated the same way: the format validator on the field lets an empty value through and the submission is accepted. That holds for `emailInspectionValidator`, `urlInspectionValidator`, `regExpValidator`, `fieldMaskValidator`, `stringInspectionValidator`, `checkForAGivenList` and `checkForNumberValidator`.

Only the empty case is treated this way. A marker sent with a non-empty value the validator does not accept still answers `400` naming the marker, and whether a value has to be there at all is decided by `requiredValidator` alone, with its own message.

A numeric field is no exception, and its configuration does not change the answer: `checkForNumberValidator` accepts an empty value with `integerOnly` either way and with `minValue` or `maxValue` set. A non-numeric value still answers `400` naming the marker, whole-number mode still rejects `3.5`, and a range is still enforced on a value that is actually there.

`null` is not an empty value here. A marker sent with `null` answers `400` whatever the field's type or validator configuration: the submission body is checked against its own shape before any validator runs, and a value there has to be a string or a number. Omit the marker instead of sending it with `null`.

**Omitting the markers you have no value for is still the better shape**, even though sending every control a form rendered is now accepted. The difference shows up in the panel: a submission row then shows only the fields the visitor actually filled, as a missing key rather than an empty string.
A required field reads whitespace as nothing. A marker sent as `"   "` or `"\t\n "` to a field carrying `requiredValidator` is refused in the same words as an omitted one, so a visitor who types a space into a required box meets a `400` instead of leaving a stored record whose value is a space.

`trimValidator` on a field trims its string value before every other validator on that field runs, so the trimmed value is the one the required and format checks see, and the one stored. Paired with `stringInspectionValidator` a padded value is measured after the padding is gone.

**Omit the markers you have no value for.** Sending every control a form rendered is the obvious implementation and the one that meets this; drop the empty ones from `formData` before submitting and an optional field behaves as optional. The consequence to expect in the panel: a submission row then shows only the fields the visitor actually filled, as a missing key rather than an empty string.

## Reading submissions

Submissions are read by the form's marker, with a `POST` whose filter travels in the body. Which filter narrows to one page and which one narrows to a module binding is the part worth reading before you call it.

→ `mcp/docs/api/form-submissions`

## Submission status

A submission can carry a status an operator moves through as they process it.

→ `mcp/docs/api/form-submissions#submission-status`

## Delete a form

Deleting a form removes its submissions with it — definition and data are one chain — and this is a case where the dry run's target understates what is lost. Before confirming, count the submissions and tell the human the number: "delete the old contact form" often means "and keep the enquiries".

## Common mistakes

- **Sending the create body unwrapped**, or building it from the schema alone — the form ends up typeless.
- **Submitting to a form that was never bound.** No config means no `formModuleConfigId` that works.
- **Taking `formIdentifier` and `formModuleConfigId` from different forms.** They are compared.
- **Reading a field error as proof the config is optional.** Field checks simply run first.
- **Updating a form without echoing `formModuleConfigs`.** Bindings and submissions are deleted, answering `200`.
- **Echoing them back without their `id`.** Same loss, and the form still looks bound.
- **Reading a `409` on a form update as a failed edit.** Nothing was written: the body asks to delete bindings holding submissions.
- **A `text` value with more than one of the three keys**, or with `params` beside them.
- **Deleting a form to "clean up".** The submissions go too.
- **Building a review form by hand.** Read `mcp/docs/api/rating-forms-and-reviews` first.

→ `mcp/docs/api/form-submissions` · `mcp/docs/api/verification-recipes#forms`
