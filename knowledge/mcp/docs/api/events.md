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
| one real trigger delivered | placeholders that did not resolve |

→ `mcp/docs/api/verification-recipes` · `mcp/docs/api/subscriptions`
