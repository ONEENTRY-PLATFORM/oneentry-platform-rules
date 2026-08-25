# Files and uploads

Uploading a file, referencing it from an attribute, and dealing with the fact that this server strips binary-looking values out of responses.

The shape of a file attribute's value depends on how many files it holds, which is the detail that breaks code written against a single example.

→ `mcp/docs/api/attribute-types` · `mcp/docs/server/response-shaping`

## Uploading goes through two tools of its own

`cms_api_call` sends every request body as **JSON**, and the upload operation wants `multipart/form-data` — so it has two tools instead: `cms_upload_file` for a file on the machine running the server, `cms_import_file_from_url` for one the server fetches. `cms_api_describe` still marks the operation as not callable through `cms_api_call`, and points at them.

Both are writes: `--allow=write`, audited, `dryRun` supported, and both refuse a source outside the bounds the operator set. Uploading outside MCP is what gives up the confirm gate, the permission check and the audit line, so prefer the tools.

→ `mcp/docs/server/cms-upload-file`

## Uploading

```text
cms_api_search { "query": "files" }
```

The binary goes in the multipart body; **everything else is a query parameter** — `type`, `entity`, `id`, `compress`, `edit` and `template`. The field name of the binary part is not enforced by the instance, so `file` and `files` both work.

An upload returns a record describing the stored file: its identity, its name, a download link and — only under the conditions in the next section — a preview link. Keep the **whole record**; that is what an attribute value references and what a site renders.

Uploads consume instance storage quota, so an upload made by mistake is not free. Do not re-upload a file to "make sure it worked"; read the record back instead.

## A visitor uploads through the public files route

A storefront does not need an admin session to attach a file. There is a public files route, reached with the site's app token, and it returns the same record the admin one does — a list, one entry per file part sent.

`type`, `entity` and `id` are **folder names and nothing else**. No entity is checked to exist: nonsense values are accepted and simply produce a path nobody will look under. They are for tidiness, not for binding a file to anything.

Two consequences worth saying out loud when advising on this:

- Anyone holding the site's public token can store a file on the instance, because the permission that governs the route is granted to the guest group by default. Storage quota is the exposed resource.
- The catalog this server searches is the Admin API only, so `cms_api_search` will never return this route. That is not evidence it does not exist.

→ `mcp/docs/server/api-catalog#paths-outside-the-admin-api-are-dropped`

## No preview template no preview and no error

`template` is **the numeric id of a `/template-previews` record**. It is not a flag, and `template=1` is not an incantation — it means "preview template number one", which on a fresh instance does not exist.

A fresh instance has **no template-previews records at all**. Upload an image without a valid template id and the file is stored correctly, `previewLink` is simply absent, and nothing reports a problem. Thirty images later that is thirty images a site cannot render as thumbnails.

So, once per instance, before the first image:

1. `GET /template-previews`. If it is empty, create one — title, identifier, and the proportions you want generated.
2. Note the **numeric id** from the response. A marker is not accepted here; a non-numeric `template` fails on the database type.
3. Upload with `template=<that id>`.
4. Read the returned record and confirm `previewLink` is present. If it is not, the template id is wrong — do not vary `type` and `entity` hoping for a different outcome, they have nothing to do with it.

Previews are generated for `png`, `jpeg` and `jpg` only. Documents get no preview by design, which is why an upload of a PDF never carries one.

A preview is not only a thumbnail — the inline placeholder a site shows while the full image loads comes from it too, so an image without one cannot be rendered progressively. Settle the template id before the first upload, not after the last.

→ `mcp/docs/api/templates-and-previews` · `mcp/docs/api/baseline-data`

## What the two preview keys in a record hold

`previewLink` is not a single link. It is keyed by the **proportion markers** of the preview template the upload named, and each value is a pair: the inline placeholder as a `data:` URI first, the preview's own address second.

```json
{ "previewLink": { "main": [ "data:image/webp;base64,UklGRs4A",
                             "https://your-instance.example/photo.preview.main.png" ] },
  "defaultPreview": "main" }
```

`defaultPreview` names **which marker to use** by default. It is a key of `previewLink`, not an address — in an `<img src>` it renders nothing.

Both keys are absent when the upload named no valid `template`, so read them defensively.

## Referencing a file from an attribute

A `file` or `image` attribute holds references to uploaded files. Write the reference in the shape the attribute already uses — read an existing entity with a populated file attribute and copy it.

Like every attribute value, it lives two levels deep, under a locale key and then the attribute key. A file that is the same in every language still goes under each active locale.

→ `mcp/docs/api/attribute-sets#two-levels-always`

## One file is an object two are a list

The value shape follows the **number of files**, not the entity kind:

- one file — a single object;
- two or more — a list;
- `groupOfImages` — always a list, even with one image.

Handle both. An attribute holding one image today holds two the moment a content manager adds one, and code that assumed the singular shape breaks at exactly that point.

```text
value is a list   → take the first element
value is an object → use it directly
```

On the way in, a single file is accepted **either way** — as the bare record or as a list holding it. The admin read then answers with whichever form you sent, so the shape you write is the shape you get back.

Write the list. It is the form that survives a second file being added, and it makes read-modify-write leave the value's shape alone.

## A file record has nowhere to keep alt text

The record an upload returns carries the file's identity, its name, its links, its size and its content type — and no `alt`, no `title`. There is nowhere in it to put the text a screen reader announces, and no image passes an accessibility review without one.

There is somewhere to put it one level up. When you write the file into an image or file attribute, the slot object keeps the keys you add to it: send `alt` and `title` alongside the upload's own fields and both come back from the public read of that attribute. Use exactly those two names — an agreed pair is what makes alt text readable across projects.

Those two names are also the pair the admin panel uses, on the same slot object — so text you set through the API is what a human sees and edits, and what a human types is what your next read returns. Where the wording needs reviewing or translating on its own schedule, a sibling attribute beside the image is still the better shape.

## Attaching a file to a form submission

A form's file field takes the **upload record itself**. Send the object the upload returned, inside a list, as the field's value: its content type, its keyed `previewLink` and its marker `defaultPreview` are all accepted as they arrived, so there is nothing to repack.

```json
{ "formData": { "en_US": [ { "marker": "photo", "type": "image",
                             "value": [ { "filename": "files/form/photo.png",
                                          "downloadLink": "https://your-instance.example/photo.png",
                                          "size": 3008 } ] } ] } }
```

`filename`, `downloadLink` and `size` are the trio a value must carry; leave one out and the answer is `400` naming the missing key. `image` and `file` take one entry, `groupOfImages` several.

A visitor uploads through the public files route with the site's app token and gets the same record, so a storefront attaches a photo to a review with no admin session.

→ `mcp/docs/api/forms-and-form-data#fields-are-attributes`

## There is no batch delete

Files are removed one at a time. Re-uploading a gallery therefore leaves the previous files in storage, consuming quota, and finding them afterwards means matching by name.

So on a large run, settle the preview template and the file naming before the first upload rather than re-uploading to fix them.

→ `mcp/docs/api/bulk-content-migration#uploading-many-files`

## Binary values are stripped from responses

This server replaces any long encoded-binary-looking string with a marker:

```json
{ "fileContent": "[stripped 184320 chars]" }
```

The rest of the response is intact — the id, the name and the download link all survive. Only the bytes are removed, so one embedded file cannot consume the whole response budget.

If you genuinely need the bytes, fetch them from the download link outside this server.

## Delete a file

Deleting removes the stored file, not the references to it. Attributes pointing at it are left with a reference that resolves to nothing, and the entity renders with a missing image rather than an error.

Find the references first. If you cannot establish what uses a file, say so before confirming the delete.

The delete call names the file, **not** its stored path. It takes `type`, `entity` and `id` as its own parameters — the same folder values the upload used — and the file's own name separately. The name a record carries is the full path those folders produced, so passing that record's name back is the common mistake: the answer is `404`, which reads like "already gone" and is not. Send the last segment of the path alone.

## Development mode differences

An instance can be configured to keep uploads locally rather than in its usual storage. Where that is on, links and behaviour differ from production.

Do not assume a link you obtained on one instance works on another, and do not carry file references between instances — upload again on the target.

## Common mistakes

- **Reading `template` as a boolean.** It is a record id, and there are no records on a fresh instance.
- **Trying `type` × `entity` combinations when previews are missing.** The cause is the template, not those.
- **Storing only the download link in the attribute.** Put the whole record there.
- **Assuming the singular value shape.** It changes with the count.
- **Re-uploading to check the first upload worked.** Read the record; uploads cost quota.
- **Treating a stripped value as data loss.** The metadata is all there.
- **Deleting a file without finding its references.** Silent missing images.
- **Copying a file reference between instances.** Upload again instead.
- **Expecting an `alt` field on the upload record.** Put `alt` and `title` on the attribute slot instead.
- **Reading `defaultPreview` as an address.** It names a key of `previewLink`.
- **Repacking an upload record for a form submission.** It is taken as it is.
- **Re-uploading a gallery to fix a naming decision.** The old files stay.

→ `mcp/docs/api/verification-recipes`
