# Rating forms and reviews

Reviews and star ratings are a **form of type `rating`** bound to the entity being rated. The platform stores the reviews and calculates the average; a product carries no rating field of its own.

Read this before writing a rating anywhere, and before importing reviews from another system. The settings that make a rating work live in a place searching for "rating" does not obviously lead to.

→ `mcp/docs/api/forms-and-form-data` · `mcp/docs/api/products`

## Never store a rating as an entity attribute

A `rating` attribute and a `reviews_count` attribute on a product are two numbers that go stale the moment a visitor writes a review, and they make the platform's own calculation disagree with the catalogue.

The rule is the general one: **if the platform can calculate it, it calculates it.** The site reads the aggregate; nothing multiplies, averages or counts on the way to a page. If such attributes exist from an earlier import, remove them — and remove the values they left on the entities, which outlive the attribute.

→ `mcp/docs/api/attribute-sets#values-outlive-the-attribute-you-removed`

## What a review form is made of

- a form with `type: "rating"`, created wrapped under `newForm`;
- a **score** field, marked as the rating value;
- the fields a review shows — author, title, body, and any flags such as a verified purchase;
- a **module config** binding the form to the entity kind being rated, carrying the rating settings.

Nothing about a review is stored on the entity. A review is a submission against the binding.

## The rating settings live in the module config

They have no operation of their own. They are fields of an entry in `formModuleConfigs`, written with the form update:

```json
{ "formModuleConfigs": [ { "formId": 5, "moduleId": 3, "isGlobal": true,
                           "entityIdentifiers": [],
                           "isRating": true,
                           "ratingCalculation": "average",
                           "maxRatingScale": 5,
                           "allowHalfRatings": false,
                           "allowRerating": false,
                           "isAnonymous": true } ] }
```

`isRating` turns the binding into a rating binding, `ratingCalculation` decides how scores combine, `maxRatingScale` and `allowHalfRatings` define the scale, `allowRerating` says whether one author may score twice, `isAnonymous` whether an unauthenticated visitor may submit.

Read the form first and send the whole list back: an omitted `formModuleConfigs` deletes every binding **and its submissions**, and answers `200 true`.

→ `mcp/docs/api/forms-and-form-data#a-form-must-be-bound-before-it-accepts-submissions`

## The score field must be marked as the rating value

One field of the form carries the score, and it must be marked with `isRatingValue`. Without that mark the form is rejected as having no rating marker, and the message names the marker rather than the field you forgot to flag.

Set it when you create the field, and confirm it by reading the form back before submitting anything.

## Verified buyer and similar flags are separate fields

A flag a shop shows next to a review — verified buyer, verified by the shop — is **its own field**, as a `list`, one per flag. Not one combined flag, and not free text.

Separate list fields are what makes them filterable and what lets a human edit one in the admin panel without touching the other.

→ `mcp/docs/api/list-options-and-extra-values`

## Importing reviews from another system

The Admin API route treats one authenticated author as one review per entity: the second submission for the same product is refused as already rated. So a bulk import of visitor reviews cannot go through it, and this server cannot make the visitor request for you.

What the visitor route needs, beyond the submission body:

- a per-request `x-device-metadata` header carrying a fingerprint and basic client information, shaped `{"fingerprint":"…","deviceInfo":{"os":"…","browser":"…","location":"…"}}`. It is required for a form that carries a rating, and omitting it answers `400` naming the header;
- a fingerprint that is **distinct per review and derived from the review** — a random one per run turns the second run into a duplicate set;
- `formData` values in the field's own shape: a `text` field takes a list of objects carrying the markup value, not a bare string.

Tell the human that this part runs outside MCP, so the confirmations and the audit line do not cover it.

## An unapproved review does not count

Submissions carry a status, and the aggregate is built from approved ones. After an import, read the statuses: a whole import sitting unapproved reads exactly like a rating that failed to calculate.

Delete the submissions you made while working out the shape. Test reviews count towards the average as readily as real ones, and they are invisible on a page while distorting the number next to it.

## The aggregate is calculated after a delay

Immediately after the last submission the aggregate can still be empty, or counted for a fraction of the entities. That is normal and not evidence of a failure.

Check again after a wait before reporting anything. A first check taken as the truth turns a working platform into a defect report — and a report an agent has to withdraw costs more than the wait.

→ `mcp/operating-rules#a-read-straight-after-a-write-can-lag`

## Checking a review import

1. Read the form by identifier: `type` is `rating`, the score field carries `isRatingValue`, and `formModuleConfigs` has an entry with an `id`, `isRating` and the scale you meant.
2. Count the submissions with the count route rather than filtering the listing by form identifier.
3. Read the statuses and confirm the imported reviews are approved.
4. Wait, then read the aggregate on a sample of entities, then on all of them.
5. Search for the entities you tested against and confirm no debug submission survives.

→ `mcp/docs/api/verification-recipes#forms` · `mcp/docs/api/bulk-content-migration`
