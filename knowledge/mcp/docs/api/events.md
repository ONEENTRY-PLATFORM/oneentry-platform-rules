# Events

An event watches something in one module and sends a message when it happens: a mail, a push, a notification. It is how "tell the customer when the order ships" is configured without any code.

Read this before promising a human that a notification can be built. Which modules an event can watch is a closed list, and one outside it is refused.

→ `mcp/docs/api/modules` · `mcp/docs/api/forms-and-form-data`

## What an event is made of

- a **module** it watches, given as `moduleId`;
- the condition inside that module — which entity, which change, which status;
- **`localizeInfos`** carrying the event's name and every message it sends, per locale;
- recipients, resolved from the module the event belongs to.

Everything a human reads — the name in the list, the mail subject, the mail body, the push text — is in `localizeInfos`. Nothing about the message is anywhere else.

## Which modules an event can watch

Six: catalogue, forms, orders, users, payments, discounts.

**There is no content module among them.** An event on a page, a block or one of their attributes cannot be built, so "send a mail when this article is published" is not an events task. Say that instead of looking for the right payload — and see the next section for why the API will not say it for you.

## An event on another module is refused

A `moduleId` outside those six answers `400`, and the message names every module that is accepted. The same check runs on update, so an event cannot be moved onto an unsupported module either.

Take the rejection at face value: it means the notification the human asked for cannot be built as an event, not that the body was malformed. Report that, rather than looking for a payload that gets through.

An event created with no `moduleId` at all is a template — see the notification node of a workflow.

## The subject and the body live in localizeInfos

```json
{ "moduleId": 1,
  "localizeInfos": { "en_US": {
    "title": "New product published",
    "subject": "A new product is live",
    "template": "<p>{{ product.title }} is now available.</p>",
    "push": "{{ product.title }} is now available" } } }
```

`subject` is the mail subject, `template` the mail body, `push` the push body. A `mailing` field exists on the operation and belongs to the mailing module — a subject written there is stored and never used.

## The name of an event is title not name

`name` is what the create body declares as required, and it is not what the admin panel shows: the list is built from `localizeInfos.<locale>.title`. An event created with `name` alone appears as untitled and is hard for a human to find again.

Send both, then read the event back and check `title` is present for every active locale.

## Configuring a form submitted email event

An event on the forms module is configured by three fields the create and update schemas do carry but that nothing else points you at. Together they decide whether a submitted form sends anything.

```json
{ "name": "Enquiry received", "moduleId": 2,
  "formType": "submit_data",
  "formIdentifier": "contact-us",
  "actions": { "isEmail": true },
  "forms": { "mode": "any_data",
             "emails": [ { "attr": "email" },
                         { "plain": "sales@your-instance.example" } ] },
  "localizeInfos": { "en_US": {
    "title": "Enquiry received",
    "subject": "A new enquiry",
    "template": "<p>Someone filled in the contact form.</p>",
    "push": "A new enquiry" } } }
```

- **`formType`** is a closed set: `registration`, `send_code`, `change_password`, `submit_data`. A value outside it answers `400 formType must be a valid enum value` — the message does **not** list what is allowed, so read the four from here or from the operation schema.
- **`forms.emails` is what decides where the mail goes.** Each entry is either `{"attr":"<field marker>"}`, taking the address from the submitted field, or `{"plain":"<address>"}`, a fixed recipient. Objects, not strings, even though the schema types the list loosely.
- **`forms.mode`** is `any_data`, `reply` or `status`; **`forms.status`** applies when the mode is `status`.
- **`formIdentifier`** binds the event to a form. `formEmailFieldIdentifier` is legacy and is not read — the recipient comes from `forms.emails`.
- Subject, body and push text live in `localizeInfos`, exactly as for any other event.

**Every event bound to a form fires.** Two events naming the same `formIdentifier` both send on one submission, so a duplicate left over from an experiment doubles the mail rather than being ignored.

## An event with no recipient looks configured

While `forms.emails` is empty the event is complete in every visible way — it has a subject, a body, `isEmail` set — and sends to nobody.

That is now visible rather than silent: the submission writes a `skipped` row into the event's email log, reading `no-recipient-configured`. Read the log after the first real trigger rather than trusting the `201`.

→ `mcp/docs/api/forms-and-form-data#a-form-must-be-bound-before-it-accepts-submissions`

## Which placeholders are available

Placeholders depend on the module the event watches. For the catalogue, the entity's own fields — its title and any attribute marker. For recipients, the user's fields by marker.

There is no operation that lists them, so build one message, send it to yourself, and read what arrived before writing the rest. A placeholder that does not resolve is delivered as its own text.

## Creating an event

1. Read the modules listing and take the `moduleId` of one of the six.
2. Read the active locales.
3. Create the event with both `name` and `localizeInfos.<locale>.title`, plus `subject`, `template` and `push` for the channels you want.
4. Read it back and confirm `moduleId` and the localized fields.
5. Trigger the condition once, on a scratch entity, and confirm a message arrives.

Step 5 is the only real check of delivery. Step 1 matters: taking the `moduleId` from the listing is what keeps step 3 from being refused.

## Checking that an event is configured

| check | what it catches |
|---|---|
| the create answered `201`, not `400` | a module that has no event settings |
| `localizeInfos.<locale>.title` is set | an event a human cannot find in the panel |
| `subject` and `template` are non-empty per locale | a message written under `mailing` |
| on a form event, `forms.emails` is non-empty | an event that is complete and sends to nobody |
| one real trigger delivered | placeholders that did not resolve |
| the event's mail log carries an entry | a message nobody was resolved to receive |

## Checking whether an event actually sent mail

A configured event tells you nothing about what it sent. `GET /events/{id}/email-logs` — `AdminEventsController_findEmailLogs` — does: it answers a paged list, `limit` and `offset` in the query, newest first, with a total beside the items. One entry per attempted recipient, carrying `email`, `subject`, `status`, `error` and `createdAt`.

`status` is one of three values:

- `sent` — the platform accepted the message for sending. It is **not** a delivery receipt, so do not report a mail as received on the strength of it.
- `failed` — sending was attempted and did not work; `error` says what happened.
- `skipped` — it was deliberately not attempted, and the entry says why.

An event that asked for mail and resolved nobody now leaves a `skipped` row saying so, so **no entries at all** after a trigger you believe fired means the trigger did not reach the event — a question about its module and its condition, not about the message. Re-saving `subject` and `template` changes nothing either way.

→ `mcp/docs/api/verification-recipes` · `mcp/docs/api/subscriptions`
