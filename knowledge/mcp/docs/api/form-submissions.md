# Reading form submissions

What visitors submitted against a form: how the listing is called, how to narrow it to one page or one product, and what comes back. The listing has one supported method and two filters that are easy to confuse with each other.

Read this when a listing returns everything, nothing, or a shape you did not expect.

→ `mcp/docs/api/forms-and-form-data` · `mcp/docs/api/rating-forms-and-reviews`

## The listing is a POST with the filter in the body

Submissions are always read **by the form's marker**, and that route is a `POST` because the filter travels in the body rather than in the query string. There is no `GET` form of it. The flat listing of every submission answers `405` and names the per-form route instead — take the name and change the method too, not just the path.

When you only need a total, use the count route for the same marker rather than paging a listing to the end.

Values follow each field's type, so read them as you would any attribute value. Responses are capped by this server; page with the operation's `limit` and `offset`.

→ `mcp/docs/server/response-shaping`

## Reading the submissions of one page or product

`entityIdentifier` in the filter body is what separates one entity's submissions from another's. Send the same value the submission was made with as `moduleEntityIdentifier`: a page's textual identifier, a product's numeric id, a login for a user or an admin.

An integer value is matched against the entity's internal id as well as against that text, so an id copied out of a listing keeps working. A value too long to be an id — a phone number used as a login, say — is compared as text only, which is the correct reading of it.

```json
{ "entityIdentifier": "contacts-page", "status": ["SENT"] }
```

The rest of the filter body is `parentId` (`null` for root-level entries only), `dateFrom` and `dateTo` over the submission time, `status`, `userIdentifier` and `isAdmin`. A `parentId` that is not an integer answers `400`.

## What formModuleConfigId narrows and what it cannot

`formModuleConfigId` narrows the listing to a single module binding, in both the plain and the extended mode. A config id belonging to another form returns no rows rather than every submission of the form — an empty answer there means the id is foreign, not that nothing was submitted.

It **cannot** separate the entities of one form. A form has one config per module, so every page or product bound through that module shares the same config id; `entityIdentifier` is what tells them apart. Reach for `formModuleConfigId` when a form is bound to several modules and you want one of them.

## What a submission carries

Both modes return the same identity keys: `id`, `formIdentifier`, `time`, `formData`, the attribute set and module identifiers, and `entityIdentifier`, `entityId` and `formModuleConfigId` beside them. So the plain listing is enough to tell which entity a submission belongs to.

The extended mode adds the comment fields — `depth`, `path`, `parentId` — and the submitter metadata. Ask for it when you are building a thread, not by default.

## Submission status

Some submissions carry a status an operator moves through as they process it, with an operation of its own. Do not edit a submission's values to record that it was handled — and after an import, check the statuses: unapproved records do not count towards a rating.

## Common mistakes

- **Looking for a `GET` listing.** It is a `POST` with the filter in the body.
- **Using `formModuleConfigId` to separate pages.** One config per module; that is what `entityIdentifier` is for.
- **Reading an empty answer as "no submissions".** A foreign config id returns nothing.
- **Filtering the listing by form identifier.** It comes back empty; use the count route.
- **Asking for the extended mode by habit.** The plain one already names the entity.

→ `mcp/docs/api/forms-and-form-data#a-form-must-be-bound-before-it-accepts-submissions` · `mcp/docs/api/verification-recipes#forms`
