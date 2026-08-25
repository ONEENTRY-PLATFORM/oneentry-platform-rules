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

## What the rating aggregate carries

The entity's `rating` object holds more than the star. In the numeric modes — `average`, `weighted`, `median` — it carries four fields:

| field | what it is |
|---|---|
| `value` | the star, following `allowHalfRatings` |
| `averageValue` | the exact mean to two decimals, on the same scale, **not** rounded to a star |
| `distribution` | `{"1":n, …, "N":n}` — reviews per star, `N` being the scale |
| `votes` | how many submissions carried a score |

`averageValue` is the number a shop prints as "4.4 based on 38 reviews". Reading `value` for that shows 4, and the difference is a whole half-star of accuracy thrown away.

`method` says which mode produced it. Under `like_dislike` the object carries `like` and `dislike` instead, and neither `averageValue` nor `distribution` exists — branch on `method` rather than assuming the numeric shape.

## Only a submission with a score counts as a rating

The one-rating-per-visitor rule weighs **scored** submissions alone, and that is what makes follow-ups possible:

```text
score 5, new fingerprint, unrated entity   → 201
score 3, new fingerprint, same entity      → 400 "You have already rated this entity"
no score, new fingerprint, same entity     → 201
```

So a shop reply, a helpfulness note, or any submission that carries no score is accepted against an entity that is already rated. Name the review in `replayTo` and it is stored as that review's child — the extended read returns it with `parentId` set and `depth: 1`, which is what a threaded review list is built from.

## A rating submission can be edited

Updating one is accepted and rewrites only its content. The original author and the fingerprint the rerating rule keys on are both preserved, so an edit neither reassigns the review nor frees the visitor to score again.

That matters for moderation: correcting a typo in a review does not quietly turn it into the moderator's own review, and does not open a second vote.

## Importing reviews from another system

The refusal above still shapes a bulk import: one scored submission per entity per fingerprint is all the route takes, so imported reviews need a fingerprint each and this server cannot make the visitor request for you.

What the visitor route needs, beyond the submission body:

- a per-request `x-device-metadata` header carrying a fingerprint and basic client information, shaped `{"fingerprint":"…","deviceInfo":{"os":"…","browser":"…","location":"…"}}`. It is required for a form that carries a rating, and omitting it answers `400` naming the header;
- a fingerprint that is **distinct per review and derived from the review** — a random one per run turns the second run into a duplicate set;
- `formData` values in the field's own shape: a `text` field takes a list of objects carrying the markup value, not a bare string.

Tell the human that this part runs outside MCP, so the confirmations and the audit line do not cover it.

## Which review statuses the aggregate counts

The aggregate counts the submissions a visitor can see, and no others. A submission held for moderation, one blocked by a moderator, and one deleted are all left out; a submission that is simply sent, or one already approved, is counted.

Whether a new review waits for approval is the binding's own setting, `isModerate`. With it off there is no approval step at all, so a review counts from the moment it is accepted — waiting for an "approved" status that never arrives is the usual reason an import looks like it failed to calculate. With it on, a review counts only once a moderator approves it, so after an import read the statuses before concluding anything.

Because the rule is "visible or not", the aggregate also moves **downwards** with no new review: blocking or deleting one takes it out of the average and out of the counts. An entity left with no visible reviews reads as zero — zero votes and a zero score, not the number it carried before.

Delete the submissions you made while working out the shape. Test reviews count towards the average as readily as real ones, and they are invisible on a page while distorting the number next to it.

## The aggregate is calculated after a delay

Immediately after the last submission the aggregate can still be empty, or counted for a fraction of the entities. That is normal and not evidence of a failure.

A change of status carries the same delay: approving, blocking or deleting a review moves the average no faster than writing a new one does. A read taken straight after approving looks exactly like an approval that did nothing.

Check again after a wait before reporting anything. A first check taken as the truth turns a working platform into a defect report — and a report an agent has to withdraw costs more than the wait.

→ `mcp/operating-rules#a-read-straight-after-a-write-can-lag`

## Checking a review import

1. Read the form by identifier: `type` is `rating`, the score field carries `isRatingValue`, and `formModuleConfigs` has an entry with an `id`, `isRating` and the scale you meant.
2. Count the submissions with the count route rather than filtering the listing by form identifier.
3. Read the statuses and confirm the imported reviews are approved.
4. Wait, then read the aggregate on a sample of entities, then on all of them.
5. Search for the entities you tested against and confirm no debug submission survives.

→ `mcp/docs/api/verification-recipes#forms` · `mcp/docs/api/bulk-content-migration`
