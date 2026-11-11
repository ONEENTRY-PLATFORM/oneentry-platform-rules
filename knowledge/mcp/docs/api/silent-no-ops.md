# Operations that answer success and do not do the work

The most expensive class of mistake on this platform is not a 400. It is a `200` or a `201` for a write that did not happen, or that happened to the wrong field. An agent checks the status, sees success, and reports it to the human — and the report is false.

This document lists the known cases with the route that actually works. Read it before a session that creates a lot of entities, and read the row before you use one of these operations.

→ `mcp/docs/server/payload-conventions#verify-with-the-read-your-consumer-uses`

## The table

| operation | what it answers | what does not happen | the route that works |
|---|---|---|---|
| attribute set create, update or schema replace with `schema` keyed by marker instead of `attribute<id>` | `201` / `200` | the admin panel cannot open any attribute of the set; its page lands on the not-found page | key each attribute `attribute<id>` with the marker in `identifier`, then read the set back and compare every key with its `id` |
| file upload with no valid `template` | `201` | no `previewLink` is generated | create a template-previews record first and pass its numeric id |
| block update with `blockPages: []` | `200` | every page attachment is removed, nesting included | omit the field when placement is not what you are changing; an empty array is an instruction |
| form create with no `type` | `201` | the form is stored with `type: null`, and nothing in the response says so | send `type` — it is declared and validated, so a wrong value is refused rather than stored |
| form update with no `formModuleConfigs` | `200 true` | every module binding is deleted, and the submissions recorded against them go too | read the form and send its current `formModuleConfigs` back unchanged. An instance carrying the confirmation gate answers `409` when submissions would go — see `mcp/docs/api/forms-and-form-data#an-update-drops-every-config-you-do-not-send-back` |
| an option extra value written under a locale key | `200` | it is stored and the admin panel shows the field as empty | put `type` and `value` flat in `extended`, with no locale level |
| general settings update carrying a `searchVocabulary` section | `200 true` | the section is kept nowhere, so the read afterwards is exactly what it was and the tuning you meant to apply is not in effect | send the block as `fileContentSearch.vocabulary` — see `mcp/docs/api/search-query-correction#tuning-correction-for-a-project` |
| attribute set schema with `inSearchVocabulary` on an attribute | `200 true` | the key is stored like any unrecognised one and the attribute's values stay correctable in public search | spell it `isInSearchVocabulary`; read the set back and check the leading `is` — see `mcp/docs/api/attribute-sets#changing-an-attribute-afterwards` |
| product create or update with a top-level `sku` | `201` / `200` | the article number is not stored, and every later read shows the one the product's attribute values give it | write the value on the attribute the set uses for it, inside `attributesSets` |
| environment marker update carrying a `marker` value | `200` | the marker keeps the value it had; only `environmentId` is read | remove the marker and bind the new value — see `mcp/docs/api/environment-markers#why-renaming-a-marker-does-nothing` |
| page update carrying a `version` lower than the stored one | `200` with a body of `false` | nothing is written, and no field says which version was expected | read the page, send back the `version` you read, and treat a `false` body as "someone wrote this between your read and your write" — see `mcp/docs/api/pages#update-a-page` |
| order status update carrying a different `axis` | `200 true` | the status keeps the axis it was created on; an axis cannot be moved | create a status on the axis you want — see `mcp/docs/api/order-statuses#which-axis-a-status-belongs-to` |
| `change-status` with `axis: "payment"` on a `basic` storage | `200 true` | the payment axis is untouched; the **order** axis is written instead | read both axes back. The refusal you might expect arrives only on an `advanced` storage — see `mcp/docs/api/order-statuses#the-payment-axis-is-managed-for-you` |
| firing a catalogue trigger the storage's status map has no entry for | `201 true` | no status moves on any axis | read the map for the axis first; an unmapped trigger reports success |
| storage status trigger update carrying a new `identifier` | `200 true` | the identifier is fixed at creation; only the label changes | delete the trigger and create it again, then rewrite every map that named it |
| custom menu item delete with no body | `200 true` | only that item goes; its children stay, pointing at an id that no longer exists | pass `{ "flag": "with_children" }` when the subtree should go — see `mcp/docs/api/menus#delete-a-menu-or-an-item` |
| public API error listing filtered with `createdAt[from]` / `createdAt[to]` | `200` with the period **unfiltered** | no window is applied, and the default period answers instead | this listing takes the flat `from` and `to` — see `mcp/docs/api/content-api-journal#this-listing-takes-a-flat-window-not-a-nested-one` |
| public API error listing with an unrecognised `statusCodes` token | `200` with **more** rows than asked for | the token is dropped and the filter widens rather than narrowing | use `4xx`, `5xx` or exact codes, and compare `total` with the unfiltered one — see `mcp/docs/api/content-api-journal#narrowing-by-status-code-and-by-path` |
| refund preview naming a product id the order does not contain | `200` with `refundAmount: 0` | nothing is selected, and no field says the id was unknown | key `products` by the catalog product id and check it against the order — see `mcp/docs/api/orders#what-a-refund-body-ignores-without-saying-so` |
| refund preview with a `quantity` above the number ordered | `200` | the quantity is clamped to what the order holds, so the amount is lower than requested | read `refundAmount` and `remainingProducts` from the preview rather than computing the amount |
| collection row bulk delete with `deleteAll: true` and an `ids` list | `200 true` | `ids` is ignored and **every** row in the collection is removed | send `deleteAll: false` with the ids — see `mcp/docs/api/collections#deleteall-overrides-the-ids-you-send-with-it` |
| collection row bulk delete naming ids the collection does not hold | `200 true` | nothing is deleted, and the body is identical to a successful call | read the rows back and compare the count |
| `DELETE /products/delete/multi` with `ids` in the body | `200 false` | no product is deleted | send the same body to `DELETE /products` — see `mcp/docs/api/products#deleting-several-products-at-once` |
| a batch write over many entities | `200` per call | one entity can be missed with nothing in the response to say which | re-read every entity you wrote and retry the mismatches |
| any request once the admin session lapses | `200` with the login page as the body | nothing at all was written | re-authenticate; through this server that retry is automatic, and a second login page is reported |

## Why a read-back is the only detection

Every row above is indistinguishable from success at the point of the call. There is no error to react to, no warning field, and no difference in the response body. The write is accepted; it is the *meaning* that is lost.

So the only signal is reading the entity again — and reading it **the way its consumer will**. Most of these fail specifically when verified through the endpoint that was written to, because that endpoint echoes the input back, wrong shape included.

Two rows are worse than that: a field that exists to drive the admin panel returns your own input from every read there is, so no request proves anything about it. For those, the proof is a human opening the screen — and saying the check is incomplete is the honest report when nobody can.

→ `mcp/docs/api/bulk-content-migration#panel-facing-fields-cannot-be-verified-by-reading`

`cms_api_describe` names the verifying read under `verifyWith` for the operations where the pairing is known, and repeats the caveat under `silentNoOp`. `cms_api_call` echoes both after a successful mutation.

## What this means for a report to a human

"The statuses are set", "the validators are saved", "the previews are there" are claims about state, and a `201` is not evidence for any of them. Either read the state back and say what you saw, or say that you wrote and did not verify.

A false confident report costs more than an admitted gap: the human stops checking.

## When you find a new one

Two conditions make a case belong here: the call reports success, and a read afterwards shows the work was not done. That is a platform defect, so report it with the operation id and the exact body you sent, and add the working route to this table.

Do not build a silent workaround for one. An agent that quietly retries a different way leaves nobody knowing the endpoint is broken.

→ `mcp/docs/api/verification-recipes` · `mcp/operating-rules#a-200-means-accepted-not-applied`
