# Forms and form data

A form is a definition — its fields, their types, how submissions are handled. Form data is what visitors submitted against it. Two entities, two areas of the API.

The create payload for a form has a shape the API document does not show, so read the first section before writing one.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/attribute-types`

## Create a form with a wrapped body

Send the payload nested under `newForm`:

```json
{ "newForm": { "identifier": "contact-us",
               "type": "data",
               "processingType": "script",
               "localizeInfos": { "en_US": { "title": "Contact us" } } } }
```

The API document shows the fields unwrapped. The wrapped body is the supported route and the one to use. A flat body has been observed to succeed on some instances; treat that as undefined rather than as an alternative — it is not what the endpoint documents, and it is not what the admin panel sends.

Include `type` even though the schema omits it — see below.

→ `mcp/operating-rules#operations-with-a-single-supported-path`

## What a form is made of

- an **identifier** — the marker a site uses to fetch the form;
- **`localizeInfos`** — title and description per locale;
- **fields**, defined as attributes with types, drawn from the same closed type list as everywhere else;
- a **form type** and a **processing type**, which say what kind of form it is and what happens to a submission;
- **module configs** — the bindings that say which module and which entities the form belongs to, who may submit, and how ratings behave. A form without one accepts nothing; see below.

## The form type is missing from the create schema

`type` is a real column with a closed set of values, and it is **absent from the create schema**. It is nonetheless accepted and stored, so an agent that builds its body from the schema alone creates a form with `type: null` and is told nothing.

| value | what it is for |
|---|---|
| `data` | a plain data-collection form — a contact form is this one |
| `order` | a checkout or ordering form |
| `sign_in_up` | authentication: sign-in and registration |
| `collection` | a form that gathers entries into a collection |
| `rating` | a rating or review form, with rating behaviour attached |

Copy the values exactly. On instances that predate the fix for it, the database enum still carries the historical misspelling `sing_in_up`; if `sign_in_up` is rejected with an enum error, that instance has not been migrated — report it rather than guessing at spellings.

`type` is also accepted on update, so a typeless form can be corrected without recreating it. Read the form back by identifier and check the field: a wrong or absent `type` is not reported at any point.

## Fields are attributes

A form's fields are attributes, with the same nineteen types and the same value shapes as elsewhere. A field marked as a captcha field holds no author-supplied value — it exists so a site renders a captcha widget there.

That means the rules from `mcp/docs/api/attribute-types` apply to submissions too: numbers may be `null`, choice fields store option ids rather than labels, and file fields change shape with the number of files.

## Forms are dynamic never hardcoded

A site should render a form from its definition — field markers, types and labels read from the API — rather than from hand-written markup. A hardcoded field list silently stops matching the moment someone edits the form in the admin panel.

When you are asked to "add a field to the contact form", that is a change to the form definition, not to any code.

## A form must be bound to a module before it accepts submissions

A form on its own is a definition and nothing else. What lets a submission exist is a **module config** — the object that binds the form to a module (pages, products, users…) and to entities inside it. Every submission names the config it was made against, so a form with no config accepts nothing at all, and the failure is a flat `400`.

Creating a form does not create a config. The create operation has no field for one, so a fresh form has zero, and the first submission against it fails. Say that in your plan; it is not an error you discover later.

The config is created through the **form update**, which carries a `formModuleConfigs` array the create body does not have:

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

- `formId` is the form you are updating, `moduleId` the module it is bound to — read the modules listing for the id, do not guess it.
- `isGlobal: true` with an empty `entityIdentifiers` binds the form to every entity of that module. Otherwise list the entities: `id` numeric or textual with `isNumeric` saying which, and `childrenOn` to include an entity's children.
- The pair `(formId, moduleId)` is unique. Sending the same pair again updates that config rather than adding a second one.

So the working path from nothing to a stored submission is four calls:

1. **create the form** — wrapped under `newForm`, with an explicit `type`;
2. **update the form** with `formModuleConfigs` — this is the step that has no operation of its own and is the one people miss;
3. **read the form by id** and take `formModuleConfigs[].id`. That number is the `formModuleConfigId` a submission needs, and reading it back is the only way to learn it;
4. **create the form data**, naming both the form and the config.

A submission body needs four things: `formIdentifier` (the form's marker), `formModuleConfigId` (the id from step 3), `moduleEntityIdentifier` (the entity the submission is *for* — a textual identifier, a numeric id for products, a login for users and admins), and the locale-keyed `formData`.

**`formIdentifier` and `formModuleConfigId` must describe the same form.** The server loads the config, joins it back to its form, and compares identifiers. A mismatch — and equally a `formModuleConfigId` that matches no config at all, which is what you get when the form was never bound — answers:

```
400 Incorrect formIdentifier for provided config
```

The message names `formIdentifier` because that is the field it compares, but the field that is usually wrong is the config id. Read the form and check what configs it actually has before re-sending anything.

### An update drops every config you do not send back

`formModuleConfigs` is a full replacement list, not a patch. A form update that omits it is read as "this form has no configs": every existing binding is deleted, **and the submissions recorded against those bindings are deleted with them**. The call answers `200 true`.

This makes an ordinary edit — retitling a form, changing its processing type — destructive by omission. Read the form first and send its current `formModuleConfigs` back unchanged unless changing them is the point of the update.

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

Steps 1–3 are field validation and steps 4–5 are the form itself, so **field errors arrive before the config error every time**. Fill in the missing field and the same request fails again on the config. That sequence reads like the config became a problem only afterwards; it did not. It was always required, and it was simply behind a check that failed earlier.

Two consequences worth planning around:

- **A typeless form can never accept a submission.** Step 4 rejects anything that is not `data` or `rating`, and a form created without `type` is `null`. That is the same trap as the create schema above, surfacing here as a submission failure rather than a create failure.
- **On the admin API, steps 1–3 do not run.** Field validation is wired into the content API only. Submitting through the admin API means a missing required field is stored rather than rejected, and the config check is the first thing you meet. A `400` there is about the binding, not about the fields — and a submission that the admin API accepts is not evidence that a visitor's submission would pass.

## Reading submissions

Form data operations list what was submitted, filter it, and fetch individual submissions. A submission carries the field values plus metadata about who submitted it and when.

Two things to expect:

- Values follow the field's type, so read them the way you would read an attribute value.
- Submissions can be numerous. Responses are capped by this server; page with the operation's own `limit` and `offset` rather than asking for everything.

→ `mcp/docs/server/response-shaping`

## Submission status

Some submissions carry a status that an operator moves through as they process it. There is an operation to update it, separate from the submission itself.

Do not edit a submission's values to record that it was handled. Use the status.

## Delete a form

Deleting a form removes its submissions with it — the definition and the data are one chain. Deleting is confirm-gated, and this is one of the cases where the target shown in the dry run understates what is lost.

Before confirming, count the submissions and tell the human the number. "Delete the old contact form" often means "and keep the enquiries".

## Common mistakes

- **Sending the create body unwrapped.** Nest it under `newForm`.
- **Building the body from the create schema alone.** `type` is not in it and the form ends up typeless.
- **Submitting to a form that was never bound to a module.** There is no config, so there is no `formModuleConfigId` that works. Bind it through the form update first.
- **Taking `formIdentifier` and `formModuleConfigId` from different forms.** They are compared; read the config id off the form you are actually submitting to.
- **Reading a field error as proof the config is optional.** Field checks simply run first.
- **Updating a form without echoing `formModuleConfigs`.** The bindings and their submissions are deleted, and the call answers `200`.
- **"Correcting" `sing_in_up` on an unmigrated instance.** The current value is `sign_in_up`; if that is rejected, the instance is behind — say so.
- **Building field validation from a form read by marker without checking it.** Validators written flat are absent there.
- **Hardcoding a form's fields.** Render from the definition.
- **Treating an empty numeric field as zero.** It is `null`.
- **Writing labels instead of option ids into a choice field.**
- **Deleting a form to "clean up".** The submissions go too.

→ `mcp/docs/api/verification-recipes#forms`
