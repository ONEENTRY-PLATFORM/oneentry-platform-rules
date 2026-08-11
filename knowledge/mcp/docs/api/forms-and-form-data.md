# Forms and form data

A form is a definition — its fields, their types, how submissions are handled. Form data is what visitors submitted against it. Two entities, two areas of the API.

The create payload for a form has a shape the API document does not show, so read the first section before writing one.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/api/attribute-types`

## Create a form with a wrapped body

Send the payload nested under `newForm`:

```json
{ "newForm": { "identifier": "contact-us",
               "localizeInfos": { "en_US": { "title": "Contact us" } } } }
```

The API document shows the fields unwrapped. The wrapped form is the one the endpoint accepts; sending the fields flat fails.

This is the supported route — there is no alternative body that works.

→ `mcp/operating-rules#operations-with-a-single-supported-path`

## What a form is made of

- an **identifier** — the marker a site uses to fetch the form;
- **`localizeInfos`** — title and description per locale;
- **fields**, defined as attributes with types, drawn from the same closed type list as everywhere else;
- a **form type** and a **processing type**, which say what kind of form it is and what happens to a submission;
- configuration: which entities it relates to, who may submit, and rating behaviour where relevant.

## Fields are attributes

A form's fields are attributes, with the same nineteen types and the same value shapes as elsewhere. A field marked as a captcha field holds no author-supplied value — it exists so a site renders a captcha widget there.

That means the rules from `mcp/docs/api/attribute-types` apply to submissions too: numbers may be `null`, choice fields store option ids rather than labels, and file fields change shape with the number of files.

## Forms are dynamic never hardcoded

A site should render a form from its definition — field markers, types and labels read from the API — rather than from hand-written markup. A hardcoded field list silently stops matching the moment someone edits the form in the admin panel.

When you are asked to "add a field to the contact form", that is a change to the form definition, not to any code.

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

- **Sending the create body unwrapped.** It must be nested under `newForm`.
- **Hardcoding a form's fields.** Render from the definition.
- **Treating an empty numeric field as zero.** It is `null`.
- **Writing labels instead of option ids into a choice field.**
- **Deleting a form to "clean up".** The submissions go too.

→ `mcp/docs/api/verification-recipes#forms`
