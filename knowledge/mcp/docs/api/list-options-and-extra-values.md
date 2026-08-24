# List options and their extra values

An option of a `list` or `radioButton` attribute is more than a label: each option can carry one **extra value** — a swatch colour, a badge image, an icon — and that is what lets a site render the option instead of printing its text.

Read this before building any attribute a visitor picks from. Three details here each make a correctly stored value invisible in the admin panel, and none of them shows up in a read-back.

→ `mcp/docs/api/attribute-types#the-value-form-of-a-list-and-a-radiobutton` · `mcp/docs/api/attribute-sets#an-attribute-definition-is-locale-keyed-too`

## What an option is made of

Options live in the attribute definition under `listTitles`, keyed by locale code, as a list:

```json
{ "listTitles": { "en_US": [
  { "title": "Cherry", "value": "cherry", "position": 1 },
  { "title": "Dishwasher safe", "value": "dishwasher-safe", "position": 2 } ] } }
```

`title` is what a human reads, `value` is the stable identifier an entity stores, `position` is the order shown. An entity records the option by `value`, never by `title`.

## Extra values sit in extended with no locale key

The extra value of an option is the `extended` object **inside that option**:

```json
{ "title": "Cherry", "value": "cherry", "position": 1,
  "extended": { "type": "string", "value": "#d11241" } }
```

Two fields, flat: `type` and `value`. There is **no locale key inside `extended`** — this is the one exception to the locale-first rule that governs `localizeInfos`, `validators` and `listTitles` around it. Add a locale level "by analogy" and the write succeeds, every read returns exactly what you sent, and the field is blank in the admin panel.

## Which types an extra value can be

The type is declared per option, in `extended.type`. There is no separate list of extra fields to define first.

Accepted: `string`, `integer`, `realNumber`, `fixedPointNumber`, `date`, `dateAndTime`, `time`, `image`, `file`, `json`.

An RGB colour is a `string`. A badge or an icon is an `image`. Anything a site has to parse itself is a sign the wrong type was chosen.

## An image extra value is one object

For `image` and `file`, `extended.value` is a **single file record** — the whole record returned by the upload, not a list of one:

```json
{ "title": "Dishwasher safe", "value": "dishwasher-safe", "position": 2,
  "extended": { "type": "image",
                "value": { "filename": "dishwasher-safe.png",
                           "downloadLink": "https://your-instance.example/files/dishwasher-safe.png" } } }
```

This is the opposite of a `file` or `image` *attribute*, whose shape follows the number of files. A list of one here is stored and shown nowhere.

## Several selected options need multiselect

An attribute whose value may be more than one option needs `multiselect` on the **attribute**:

```json
{ "type": "list", "identifier": "labels", "multiselect": true, "listTitles": { "en_US": [] } }
```

Without it every selected option is stored and returned by both the admin and the public read, and the admin panel displays only the **first** one.

The flag itself is readable, from both reads that matter. The attribute read by marker returns `multiselect` alongside `type` and `listTitles`, and so does **the form read by marker a site uses** — beside `listTitles`, `validators` and `settings`. So a check after writing is a real check, and a site can tell a multi-select field from a single-select one without a second call.

It is a plain boolean, not locale-scoped, and `false` when the attribute never set it. On anything that is not a `list` it is `false` and means nothing. Set it when you create the attribute, not when someone notices.

## additionalFields and image are not extra values

Two neighbouring fields look like the right home and are not:

| field | what it actually is |
|---|---|
| `additionalFields` on the attribute | a separate list of fields, keyed by `marker`. It holds no option extras; an object written there stays in the schema unread |
| `image` on the option itself | accepted, stored, survives a read — and displayed by nothing |

Three ways to attach a picture to an option, one that works, and the API response cannot tell them apart. Use `extended`.

## Why extra values are missing from a product read

Reading an entity returns only `title` and `value` for the selected options, with `additionalFields` empty:

```json
{ "labels": { "type": "list", "value": [ { "title": "Cherry", "value": "cherry" } ] } }
```

That is not a delay and not a lost write. Extra values belong to the **attribute definition**, so a site reads them once per catalogue from the attributes read by marker — the same route used for validators — and matches them to an entity's options by `value`.

Plan for two reads on the consumer side, and never conclude from an entity read that the extras were not saved.

→ `mcp/docs/api/attribute-sets#two-reads-two-answers`

## Identify an option by something stable

Labels in a source system are not normalised: the same badge arrives written three ways, differing in case, hyphens or an extra word. Building options from label text produces a set of options far larger than the set of real values, and the filter that was the point of using a list then lists each variant separately.

Pick the option by something stable — the icon file, an article number, an identifier — and choose **one** label per value. Case is part of the label: a badge shown as `NEW` on a site is `NEW`, not `New`.

## Checking an option you wrote

1. Read the set and confirm `extended` sits directly on the option, with no locale key inside it.
2. Confirm `multiselect` is on the attribute if more than one option can be chosen.
3. Read the attribute **by marker** and confirm the extras are in the projection a site receives.
4. Ask a human to open the attribute in the admin panel. For a field that exists to be rendered, that is the only proof — a read-back returns your own input either way.

→ `mcp/docs/api/verification-recipes#fields-that-exist-for-the-admin-panel` · `mcp/docs/api/content-modelling`
