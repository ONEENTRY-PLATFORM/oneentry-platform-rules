# Events

An event watches something in one module and sends a message when it happens: a mail, a push, a notification. It is how "tell the customer when the order ships" is configured without any code.

Read this before promising a human that a notification can be built. Which modules an event can watch is a closed list, and an event outside it is created without complaint and never fires.

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

## An event on another module is created and never fires

A `moduleId` outside those six is accepted. The event is stored, reads back fully populated, appears in listings — and nothing is ever sent, because the module has no event settings at all.

Nothing in the response distinguishes it from a working event, which makes this the most expensive shape of mistake here: the API reports success and the human reports that the notification is not configured. Choose the module from the six, and if the task needs another one, report it.

→ `mcp/docs/api/silent-no-ops`

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

Step 5 is the only real check. Everything before it passes for an event on an unsupported module too.

## Checking that an event is configured

| check | what it catches |
|---|---|
| `moduleId` is one of the six | an event that is stored and inert |
| `localizeInfos.<locale>.title` is set | an event a human cannot find in the panel |
| `subject` and `template` are non-empty per locale | a message written under `mailing` |
| one real trigger delivered | placeholders that did not resolve |

→ `mcp/docs/api/verification-recipes` · `mcp/docs/api/subscriptions`
