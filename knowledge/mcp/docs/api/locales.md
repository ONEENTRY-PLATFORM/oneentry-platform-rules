# Locales

Every piece of human-readable content on this platform is keyed by a locale code, so reading the active codes is the first call of almost any content task.

A broad set of codes is provisioned with the instance and what usually varies is which of them are active — but a code the instance does not have can be created.

→ `mcp/docs/server/payload-conventions#localizeinfos-is-keyed-by-locale-code` · `mcp/docs/api/baseline-data#locales`

## What a locale code looks like

A language and a region joined by an underscore: `en_US`, `ru_RU`, `fr_FR`, `de_DE`. That exact string is the key you use in `localizeInfos` and in `attributesSets`, and the value of the `langCode` parameter where an operation accepts one.

Codes are unique across the instance. Almost always the operation you want is an **activation** of a code the instance already holds, because almost every language is already there — check before reaching for a create.

## Creating a locale the instance does not have

Where the code is genuinely absent, `POST /locales` creates it under `settings.locales.create`:

```json
{ "code": "zu_ZA", "shortCode": "zu", "name": "Zulu", "nativeName": "isiZulu" }
```

`code` is the one the rest of the API uses; `shortCode` is the language half on its own; `name` is the English name and `nativeName` the language's own. A code already present answers `400 This locale already exists`, and a body with no `code` answers `400 Locale's code is not present`.

Check first with `GET /locales/check-existence/{code}`, and read its polarity the right way round: it answers `true` when the locale **exists**. That is the opposite of the `marker-validation` operations elsewhere in this API, which answer `valid: true` when a marker is **free**.

A locale created this way arrives **inactive** — `isActive` is `false`. Creating it does not make it usable for content; activation is the separate step below, and it is the one with consequences.

## Read the active codes first

```text
cms_api_call { "opId": "AdminLocalesController_findAllActive" }
```

This is the supported listing. Use it rather than a general locales listing, and rather than assuming.

Do this once at the start of a content task and keep the result. Everything you write afterwards depends on it.

## Never hardcode a locale

A fresh instance has exactly one active locale, usually `en_US`, which makes hardcoding look harmless. It is not: the moment an operator activates a second language, every payload written against a fixed key produces content that is blank in the new locale, with no error.

```json
{ "localizeInfos": { "en_US": { "title": "Summer sale" },
                     "ru_RU": { "title": "…" } } }
```

Build the object by iterating the active codes. If you only have content for one language, say so to the human and ask what should appear in the others — do not quietly leave them empty.

## The same keying applies to attribute values

`attributesSets` is keyed by locale first and by attribute second:

```json
{ "attributesSets": { "en_US": { "string_id42": "SKU-1" } } }
```

Attributes whose value is genuinely language-independent — a number, a date, a reference — still sit under a locale key. Write them for every active locale unless the entity's document says otherwise.

→ `mcp/docs/api/attribute-sets`

## Activating a language

Activation is an operator decision with consequences: every existing entity immediately has an empty version of itself in the new language, and someone has to fill it.

If a human asks you to add a language, find the activation operation, dry run it, and tell them what will follow — not just that the code will be switched on.

```text
cms_api_search { "query": "locales", "mutating": true }
```

## Reading content in one language

Many read operations accept a `langCode` query parameter and return content for that language. When it is omitted the instance chooses a default, which may not be the one you meant.

Pass it explicitly when you are checking whether a translation exists. An entity that looks complete without the parameter can be empty in the language you care about — and on the public reads listed below it can look complete when the translation is missing.

## Which content reads fall back to another language

Writes never fall back: a locale key you did not write stays empty wherever the entity is read. A handful of **public** reads are the exception, and they matter because either answer can mislead an agent checking for a translation.

These public reads return the language the value was stored in when the requested one holds nothing:

- an order, including the form data submitted with it, and the status names of an order storage
- a discount: its names and its attribute values
- a row of an integration collection, including the form data submitted with it
- the signed-in user's own profile, including the form data submitted with it

The key in the response is the real language of the value, not the one you asked for. Read the key, not the parameter: `langCode=ru_RU` answering `{ "en_US": … }` means there is no Russian translation and you are looking at the English value.

The one case where the key does match the parameter is an entity that holds nothing in any language: there is no real language to name, so the answer carries the code you asked for with an empty value — `langCode=ru_RU` answering `{ "ru_RU": [] }`. So a matching key alone does not prove a translation exists; the value does. The shape stays the same whether anything was found or not, which is what makes the check a safe one to write.

Admin reads are not affected. They return what is stored — for most entities the whole locale map, for a collection row the single language you asked for — which makes them the surface to use when the question is whether a translation exists at all.

## What to check when content looks missing

1. Is the locale actually active? An inactive code is not written and not returned.
2. Did the write include that locale key? A write for one language does not fill the others, and only the public reads listed above substitute anything.
3. Are you reading with the right `langCode`?
4. Was the entity created seconds ago? Lists lag briefly; read by id to confirm.

→ `mcp/docs/server/payload-conventions#writes-are-eventually-consistent`
