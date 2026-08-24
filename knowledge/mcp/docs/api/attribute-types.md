# Attribute field types

The closed list of field types an attribute can have, and the value shape each one expects and returns. Read this when a value you wrote came back looking different from what you sent.

There are nineteen types and no way to add a twentieth. Pick from the list.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/server/payload-conventions`

## The nineteen types

| Type | Value in a payload |
|---|---|
| `string` | a plain string |
| `text` | rich text, carrying a markup form alongside a plain form |
| `textWithHeader` | a **list** of heading-and-text pairs |
| `integer` | a whole number |
| `real`, `float` | a number |
| `date` | a date value |
| `dateTime` | a date and time value |
| `time` | a time value |
| `file` | one or more file references |
| `image` | one or more image references |
| `groupOfImages` | always a list of image references |
| `radioButton` | the id of a chosen option, as a bare string |
| `list` | a **list of option objects**; `multiselect` decides whether more than one shows |
| `entity` | references to other entities, by marker |
| `timeInterval` | a definition of repeating intervals |
| `button` | a label and a target |
| `spam` | a captcha field; it holds no author-supplied value |
| `json` | an arbitrary JSON value |

## Numbers are numbers and null is null

`integer`, `real` and `float` come back as a number or as `null`. An empty numeric attribute is **not** zero.

That distinction matters when you branch on a value: treating `null` as `0` turns "no price set" into "free". Check for `null` explicitly.

## Text has two forms

A `text` or `textWithHeader` value carries both a markup form and a plain form. Write the markup form; read whichever the consumer needs.

Do not put markup into a `string`. It is stored verbatim and rendered as characters wherever the admin panel or a site shows it.

## Text is written one way on an entity and another in a form

Same type, two shapes, and neither is accepted where the other belongs.

On an **entity** — a page, a product — the value carries the forms together, the way the admin panel writes them:

```json
[ { "htmlValue": "<p>Rinse before first use.</p>",
    "mdValue": "", "plainValue": "",
    "params": { "editorMode": "html" } } ]
```

In a **form submission** it is a list of one object holding **exactly one** of `plainValue`, `htmlValue` or `mdValue`, and nothing else — `[ { "plainValue": "Please call me back." } ]`.

Everything around that is refused, and only the first two answers name the field:

| sent as the value of a form field | answer |
|---|---|
| `"Please call me back."` | the marker's value must be an array |
| `["Please call me back."]` | the value should be of type text |
| `[{ "value": "…" }]` | value is not allowed |
| `[{ "htmlValue": "…", "mdValue": "", "plainValue": "" }]` | only one of the three may be provided |
| the four-key entity shape above | value is not allowed to be empty |

Reaching for the entity shape in a submission is the usual way to arrive at those last two.

→ `mcp/docs/api/forms-and-form-data#fields-are-attributes`

## textWithHeader is a list of pairs

The value of a `textWithHeader` attribute is a list, and holding several heading-and-text pairs in one attribute is what the type exists for:

```json
[ { "header": "How to use",
    "htmlValue": "<p>Rinse before first use.</p>",
    "mdValue": "", "plainValue": "",
    "params": { "editorMode": "html" } } ]
```

An accordion of a dozen questions is one attribute with a dozen items — not one item with the questions marked up inside it, and not one attribute per question.

→ `mcp/docs/api/content-modelling#textwithheader-holds-several-heading-and-text-pairs`

## Files and images depend on the count

The shape of a `file` or `image` value depends on **how many files there are**, not on the entity it belongs to: one file is a single object, two or more is a list. `groupOfImages` is always a list, even with one image.

Write code that survives both:

```text
value is a list  → take the first element
value is an object → use it directly
```

An attribute that holds one image today can hold two tomorrow, because a content manager added one. Code that assumes the singular shape breaks at exactly that moment.

## The value form of a list and a radioButton

Both store the option's id rather than its label, and the labels live in the attribute definition, per locale — but the two types wrap that id differently, and only one of them is a bare string.

`radioButton` is the id on its own:

```json
{ "attributesSets": { "en_US": { "radioButton_id8": "yes" } } }
```

`list` is a **list of option objects**, each carrying the id under `value` and the label under `title`:

```json
{ "attributesSets": { "en_US": { "list_id7": [
    { "value": "rfq", "title": "Request for quote" } ] } } }
```

`[]` and `""` both clear the value. A bare string, or a list of bare strings, is refused with `400`; the message names the locale and the attribute key, states the expected form and shows an example.

Writing the label where the id belongs does not work, in either type.

## The shape checks only see canonically named keys

The refusals that guard value shape — the `list` one above, and the `date`, `dateTime` and `time` one below — are keyed off the **attribute key**, not off the type recorded in the attribute set. Each runs only where the key is built the canonical way, `<type>_id<id>`.

```text
"list_id7":     ["rfq"]        → 400, the shape is refused
"my_labels":    ["rfq"]        → 200, stored exactly as sent
"date_id3":     "2026-01-15"   → 400, the shape is refused
"when_custom":  "2026-01-15"   → 200, stored exactly as sent
```

An attribute addressed by any other key — a hand-written marker, one copied from another set — is not shape-checked at all. The malformed value is accepted, the call answers `200`, and the admin read echoes it back to you unchanged, so nothing in the exchange says anything is wrong.

The key convention is therefore not cosmetic. It is what turns a value a site cannot render into a refusal you see at once. Build the key from the set, and where you cannot, read the value back **through the public route** rather than trusting the status code.

→ `mcp/docs/api/verification-recipes#the-general-pattern`

## entity references use markers

An `entity` attribute holds references to other entities, expressed as markers rather than numeric ids — which is what makes such a reference survive being moved between instances.

Resolve the marker against the instance before writing it; a reference to a marker that does not exist is stored and then fails to resolve when read.

On the public side the value arrives as a list. On an instance carrying older content it can also arrive as a brace-wrapped string of quoted markers, until that entity is written again. Parse defensively — accept both forms — rather than assuming one shape.

## json is not an escape hatch

A `json` attribute holds arbitrary structure, which makes it tempting for anything awkward. The cost is that nothing else understands it: it cannot be filtered on, cannot be indexed for search, and does not render in the admin panel as anything but raw data.

Use a typed attribute whenever one fits, and reserve `json` for genuinely opaque payloads.

## Dates and times

`date`, `dateTime` and `time` values are objects, never bare strings. A string you formatted yourself — `"2026-01-15"`, `"16:00:00"` — and a numeric timestamp are both refused.

```json
{ "attributesSets": { "en_US": { "date_id4": {
    "fullDate": "2026-03-12T00:00:00.000Z",
    "formattedValue": "12-03-2026",
    "formatString": "DD-MM-YYYY" } } } }
```

`fullDate` is the timestamp, in ISO 8601, and is the field the value is actually read from. `formattedValue` and `formatString` describe how a human sees it; both may be omitted, but each must be a string when present. The usual `formatString` is `DD-MM-YYYY` for `date`, `DD-MM-YYYY HH:mm` for `dateTime` and `HH:mm` for `time`.

Sending a scalar answers `400`, and the message names the attribute key, the expected object and an example. `""`, `null` and `{}` are accepted and mean the field is empty — that is how a value is cleared. A `fullDate` that is present but empty is refused.

## Where the date check applies

The check runs on every write that carries `attributesSets` — creating and updating a product, updating a page, creating and updating a form — and on the system operation that updates attribute values in place. On those routes a bad shape is refused before anything is written.

Writes that do not carry `attributesSets` do not run it, so an ill-formed date can still be stored elsewhere without an error. After writing a date through any other route, read the value back rather than trusting the status code.

It is also skipped for any attribute key not named `date_id<id>`, `dateTime_id<id>` or `time_id<id>` — see the section above.

Form submissions are stricter than entities. `POST /api/content/form-data` requires all three fields and does not accept `""` for these types, so `{ "fullDate": "…" }` alone, and a cleared value, pass on an entity and are refused on a submission.

→ `mcp/docs/api/forms-and-form-data#fields-are-attributes`

## timeInterval is expanded when it is read

`timeInterval` describes recurring intervals and is expanded into concrete ranges at read time, so what you write and what a consumer sees are not the same shape.

## Read free text through the public route

Free text is unrestricted: length and punctuation, braces included, come back from the public read as written. Very long bodies still belong in the entity's own content rather than in an attribute — a choice about whoever edits it, not a limit.

The general rule the sections above keep pointing at: the admin read and the public read are different projections, and only the public one tells you what a site receives.

→ `mcp/docs/api/verification-recipes#the-general-pattern`

## When a value looks wrong after writing

1. Was the attribute key built as `<type>_id<id>` from the set, with the right type prefix?
2. Was it nested under a locale key?
3. For a file or image, did you send the shape matching the number of files?
4. For a choice type, did you send an option id rather than a label — wrapped in an option object for a `list`, bare for a `radioButton`?
5. Did you read the entity back by id, rather than looking at a list that may lag?
6. For a choice type with several selections, is `multiselect` set on the attribute?

→ `mcp/docs/api/list-options-and-extra-values`

→ `mcp/docs/api/attribute-sets#checklist-before-writing-values`
