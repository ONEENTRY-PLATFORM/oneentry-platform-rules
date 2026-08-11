# Locales

Every piece of human-readable content on this platform is keyed by a locale code, so reading the active codes is the first call of almost any content task.

There is nothing to create here. The full set of codes is provisioned with the instance; what varies is which ones are active.

→ `mcp/docs/server/payload-conventions#localizeinfos-is-keyed-by-locale-code` · `mcp/docs/api/baseline-data#locales`

## What a locale code looks like

A language and a region joined by an underscore: `en_US`, `ru_RU`, `fr_FR`, `de_DE`. That exact string is the key you use in `localizeInfos` and in `attributesSets`, and the value of the `langCode` parameter where an operation accepts one.

Codes are unique across the instance, so a create always fails. The operation you want, when a new language is needed, is an **activation** of a code that already exists.

## Read the active codes first

```text
cms_api_call { "opId": "AdminLocalesController_findAllActive" }
```

This is the supported listing. Use it rather than a general locales listing, and rather than assuming.

Do this once at the start of a content task and keep the result. Everything you write afterwards depends on it.

## Never hardcode a locale

A fresh instance has exactly one active locale, usually `en_US`, which makes hardcoding look harmless. It is not: the moment an operator activates a second language, every payload written against a fixed key produces content that is blank in the new locale, with no error and no fallback.

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

Pass it explicitly when you are checking whether a translation exists. An entity that looks complete without the parameter can be empty in the language you care about.

## What to check when content looks missing

1. Is the locale actually active? An inactive code is not written and not returned.
2. Did the write include that locale key? Content written for one language does not fall back to another.
3. Are you reading with the right `langCode`?
4. Was the entity created seconds ago? Lists lag briefly; read by id to confirm.

→ `mcp/docs/server/payload-conventions#writes-are-eventually-consistent`
