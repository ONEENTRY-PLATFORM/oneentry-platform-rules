# Attribute field types

The closed list of field types an attribute can have, and the value shape each one expects and returns. Read this when a value you wrote came back looking different from what you sent.

There are nineteen types and no way to add a twentieth. Pick from the list.

→ `mcp/docs/api/attribute-sets` · `mcp/docs/server/payload-conventions`

## The nineteen types

| Type | Value in a payload |
|---|---|
| `string` | a plain string |
| `text` | rich text, carrying a markup form alongside a plain form |
| `textWithHeader` | rich text with a heading |
| `integer` | a whole number |
| `real`, `float` | a number |
| `date` | a date value |
| `dateTime` | a date and time value |
| `time` | a time value |
| `file` | one or more file references |
| `image` | one or more image references |
| `groupOfImages` | always a list of image references |
| `radioButton` | the id of a chosen option |
| `list` | the chosen option ids |
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

## Files and images depend on the count

The shape of a `file` or `image` value depends on **how many files there are**, not on the entity it belongs to: one file is a single object, two or more is a list. `groupOfImages` is always a list, even with one image.

Write code that survives both:

```text
value is a list  → take the first element
value is an object → use it directly
```

An attribute that holds one image today can hold two tomorrow, because a content manager added one. Code that assumes the singular shape breaks at exactly that moment.

## Choice types store ids not labels

`radioButton` stores the id of the chosen option; `list` stores the chosen option ids. The human-readable labels live in the attribute definition, per locale.

So to set a choice you need the option id from the attribute set, and to display one you need to resolve the id back to a label in the locale you are rendering. Writing the label as the value does not work.

## entity references use markers

An `entity` attribute holds references to other entities, expressed as markers rather than numeric ids — which is what makes such a reference survive being moved between instances.

Resolve the marker against the instance before writing it; a reference to a marker that does not exist is stored and then fails to resolve when read.

## json is not an escape hatch

A `json` attribute holds arbitrary structure, which makes it tempting for anything awkward. The cost is that nothing else understands it: it cannot be filtered on, cannot be indexed for search, and does not render in the admin panel as anything but raw data.

Use a typed attribute whenever one fits, and reserve `json` for genuinely opaque payloads.

## Dates and times

`date`, `dateTime` and `time` values are objects with a full timestamp inside rather than bare strings. Read one back from an existing entity and copy the shape rather than sending a string you formatted yourself.

`timeInterval` is different again: it describes recurring intervals and is expanded into concrete ranges at read time, so what you write and what a consumer sees are not the same shape.

## When a value looks wrong after writing

1. Was the attribute key built as `<type>_id<id>` from the set, with the right type prefix?
2. Was it nested under a locale key?
3. For a file or image, did you send the shape matching the number of files?
4. For a choice type, did you send an option id rather than a label?
5. Did you read the entity back by id, rather than looking at a list that may lag?

→ `mcp/docs/api/attribute-sets#checklist-before-writing-values`
