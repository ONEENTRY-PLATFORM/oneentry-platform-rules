# Captcha fields on a form

A form can carry one captcha field, and a submission against such a form must bring a verification token with it. The field is not decoration: leave its value out and the submission is refused.

Read this before you build a submission for a form whose field list includes a captcha field, and before you attach one to a form.

→ `mcp/docs/api/forms-and-form-data#fields-are-attributes` · `mcp/docs/api/attribute-types`

## What a captcha field must carry

The field's type is `spam`, and it takes a value like any other field. The value is an object holding the token the visitor's browser produced together with the site key that produced it, and it goes into the locale-keyed `formData` under the field's own marker:

```json
{ "formData": { "en_US": [ { "marker": "captcha",
                             "type": "spam",
                             "value": { "event": { "token": "03AFcWe...", "siteKey": "6Lc000AAAAA" } } } ] } }
```

That object is handed to the verification provider unchanged, so the two keys and their nesting are the whole contract. A value in any other shape is refused exactly as a failed check is — in the answer, a wrong shape and an automated visitor look identical.

## There is no operation that checks a token

Nothing in the API takes a token and answers valid or invalid. A token is checked while the submission carrying it is accepted, and that submission's own status code is the verdict. Build the client around one call, not two.

Only the visitor-facing routes check it: creating form data, and creating or updating an order through a storage marker. On the operator API the check does not run at all, so a submission you make as an operator is no evidence that a visitor's would pass.

## Produce the token when the form is submitted

The token is single-use and expires within roughly two minutes of being issued. Requesting it once as the page loads and reusing it at submit time works while you are testing and fails for real visitors, who take longer. The symptom is a refusal that looks intermittent and unrelated to the key.

Request a fresh token inside the submit handler, every time, including after a refused attempt.

## What the refusals mean

| Status | Message | What to change |
|---|---|---|
| `400` | `formData doesn't have spam attribute` | the form has a visible captcha field and your `formData` carries no entry for its marker |
| `400` | `Captcha Validation Failed, captcha type is: google` | the token was refused or had expired, or the value was not shaped as above. The message names the provider, never the reason |
| `400` | `Your form has more than 1 span attribute whereas the only one is possible` | the form's field list has two visible captcha fields; leave one |

The middle row covers several causes at once, among them a site key that does not belong to the credentials the instance verifies with. When every submission is refused, including one you know came from a person, suspect the configuration before the client.

## Which forms can use a captcha field

Attach it to a data form or an order form. A form used for sign-in and registration cannot carry one: while a visible captcha field is on it, every submission through the registration and profile routes answers `400` whatever the token says. Keep those forms free of it.

→ `mcp/docs/api/forms-and-form-data#the-order-the-errors-arrive-in` · `mcp/docs/api/form-submissions`
