# Operations that answer success and do not do the work

The most expensive class of mistake on this platform is not a 400. It is a `200` or a `201` for a write that did not happen, or that happened to the wrong field. An agent checks the status, sees success, and reports it to the human — and the report is false.

This document lists the known cases with the route that actually works. Read it before a session that creates a lot of entities, and read the row before you use one of these operations.

→ `mcp/docs/server/payload-conventions#verify-with-the-read-your-consumer-uses`

## The table

| operation | what it answers | what does not happen | the route that works |
|---|---|---|---|
| attribute set schema replace with flat `validators`, `listTitles`, `localizeInfos` or `additionalFields` | `200 true` | the entry is stored outside any locale key, so the projection a site reads returns `{}` | key it by locale. Create and update of the set itself answer `400` for the same body — only the schema-replace operation accepts it |
| attribute set schema replace wrapped as `{ "schema": … }` | `200 true` | the wrapper is stored as the schema; the previous attributes are gone | send the schema object itself |
| file upload with no valid `template` | `201` | no `previewLink` is generated | create a template-previews record first and pass its numeric id |
| block update with `blockPages: []` | `200` | every page attachment is removed, nesting included | omit the field when placement is not what you are changing; an empty array is an instruction |
| form create with no `type` | `201` | the form is stored with `type: null`, and nothing in the response says so | send `type` — it is declared and validated, so a wrong value is refused rather than stored |
| form update with no `formModuleConfigs` | `200 true` | every module binding is deleted, and the submissions recorded against them go too | read the form and send its current `formModuleConfigs` back unchanged |
| an option extra value written under a locale key | `200` | it is stored and the admin panel shows the field as empty | put `type` and `value` flat in `extended`, with no locale level |
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
