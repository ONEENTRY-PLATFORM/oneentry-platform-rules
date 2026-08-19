# Forms and form data

A form is a definition — its fields, their types, how submissions are handled. Form data is what visitors submitted against it. Two entities, two areas of the API.

The create payload has a shape the API document does not show, so read the first section before writing one.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/attribute-types`

## Create a form with a wrapped body

Send the payload nested under `newForm`, and include `type` even though the schema omits it:

```json
{ "newForm": { "identifier": "contact-us",
               "type": "data",
               "processingType": "script",
               "localizeInfos": { "en_US": { "title": "Contact us" } } } }
```

The API document shows the fields unwrapped. The wrapped body is the supported route; a flat one has been observed to succeed on some instances, and that is undefined behaviour rather than an alternative.

→ `mcp/operating-rules#operations-with-a-single-supported-path`

## What a form is made of

- an **identifier** — the marker a site uses to fetch the form;
- **`localizeInfos`** — title and description per locale;
- **fields**, defined as attributes drawn from the same closed type list as everywhere else;
- a **form type** and a **processing type** — what kind of form it is, what happens to a submission;
- **module configs** — the bindings naming which module and entities the form belongs to, who may submit and how ratings behave. A form without one accepts nothing; see below.

## The form type is missing from the schema

`type` has a closed set of values and is **absent from the create schema**. It is nonetheless accepted and stored, so a body built from the schema alone creates a form with `type: null` and nothing says so.

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

A form's fields are attributes, with the same nineteen types and value shapes as elsewhere, so everything in `mcp/docs/api/attribute-types` applies to submissions: numbers may be `null`, choice fields store option ids, file fields change shape with the count. A `text` field takes a list of objects carrying the markup value — a bare string, a list of strings and a list of `{ "value": … }` objects are all refused. A captcha field holds no author-supplied value; it exists so a site renders a widget there.

## Forms are dynamic never hardcoded

A site renders a form from its definition — field markers, types and labels read from the API — not from hand-written markup, which stops matching the moment someone edits the form. So "add a field to the contact form" is a change to the definition, not to any code.

## A form must be bound before it accepts submissions

A form on its own is a definition. What lets a submission exist is a **module config** — the object binding the form to a module (pages, products, users…) and to entities inside it. Every submission names the config it was made against, so a form with no config accepts nothing and the failure is a flat `400`.

Creating a form does not create one: the create operation has no field for it, so a fresh form has none and the first submission fails. Say that in your plan rather than discovering it later. The config comes from the **form update**, which carries a `formModuleConfigs` array the create body does not have:

```json
{ "formModuleConfigs": [ { "formId": 5,
                           "moduleId": 4,
                           "isGlobal": false,
                           "entityIdentifiers": [ { "id": "contacts-page", "isNumeric": false, "childrenOn": false } ],
                           "isClosed": false,
                           "isModerate": false,
                           "viewOnlyUserData": true,
                           "commentOnlyUserData": false } ] }
```

- `formId` is the form being updated, `moduleId` the module — read the modules listing for the id.
- `isGlobal: true` with an empty `entityIdentifiers` binds the form to every entity of the module; otherwise list them, `isNumeric` saying which kind of `id` each is and `childrenOn` including its children.
- `(formId, moduleId)` is unique: the same pair again updates that config instead of adding one.

So the path from nothing to a stored submission is four calls: **create** the form wrapped under `newForm` with an explicit `type`; **update** it with `formModuleConfigs`, the step with no operation of its own that people miss; **read it back** and take `formModuleConfigs[].id`, which is the only way to learn the `formModuleConfigId`; **create the submission**, naming both the form and that config.

A submission body needs `formIdentifier` (the form's marker), `formModuleConfigId`, `moduleEntityIdentifier` (the entity the submission is *for* — a textual identifier, a numeric id for products, a login for users and admins) and the locale-keyed `formData`.

**`formIdentifier` and `formModuleConfigId` must describe the same form.** The config is joined back to its form and the identifiers compared, so a mismatch — or a config id matching nothing, which is what an unbound form gives you — answers `400 Incorrect formIdentifier for provided config`. The message names `formIdentifier` because that is the field compared; the wrong one is usually the config id. Read the form's configs before re-sending anything.

## An update drops every config you do not send back

`formModuleConfigs` is a full replacement list, not a patch. An update that omits it is read as "this form has no configs": every binding is deleted, **and the submissions recorded against them go too**, and the call answers `200 true`. That makes an ordinary edit — a retitle, a processing-type change — destructive by omission. Read the form first and send its current `formModuleConfigs` back unless changing them is the point.

→ `mcp/docs/api/silent-no-ops`

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

## Reading submissions

Form data operations list what was submitted, filter it, and fetch one submission. A submission carries the field values plus metadata about who submitted it and when.

Three things to expect:

- Values follow the field's type — read them as you would an attribute value.
- Responses are capped by this server; page with the operation's `limit` and `offset`.
- There is no flat listing of every submission: that route answers `405` and names the per-form route to use instead. Always read submissions **by the form's marker**, and use the count route for the form's marker when you only need a total.

→ `mcp/docs/server/response-shaping`

## Submission status

Some submissions carry a status an operator moves through as they process it, with an operation of its own. Do not edit a submission's values to record that it was handled — and after an import, check the statuses: unapproved records do not count towards a rating.

## Delete a form

Deleting a form removes its submissions with it — definition and data are one chain — and this is a case where the dry run's target understates what is lost. Before confirming, count the submissions and tell the human the number: "delete the old contact form" often means "and keep the enquiries".

## Common mistakes

- **Sending the create body unwrapped**, or building it from the schema alone — the form ends up typeless.
- **Submitting to a form that was never bound.** No config means no `formModuleConfigId` that works.
- **Taking `formIdentifier` and `formModuleConfigId` from different forms.** They are compared.
- **Reading a field error as proof the config is optional.** Field checks simply run first.
- **Updating a form without echoing `formModuleConfigs`.** Bindings and submissions are deleted, answering `200`.
- **Filtering the submission listing by form identifier.** It comes back empty; use the count route.
- **Deleting a form to "clean up".** The submissions go too.
- **Building a review form by hand.** Read `mcp/docs/api/rating-forms-and-reviews` first.

→ `mcp/docs/api/verification-recipes#forms`
