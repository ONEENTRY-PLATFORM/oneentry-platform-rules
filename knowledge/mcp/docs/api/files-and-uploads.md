# Files and uploads

Uploading a file, referencing it from an attribute, and dealing with the fact that this server strips binary-looking values out of responses.

The shape of a file attribute's value depends on how many files it holds, which is the detail that breaks code written against a single example.

→ `mcp/docs/api/attribute-types` · `mcp/docs/server/response-shaping`

## This server cannot upload a file

`cms_api_call` sends every request body as **JSON**. The upload operation wants `multipart/form-data`, so it cannot be called through this server at all. It is in the catalog because it exists, and `cms_api_describe` marks it `notExecutable` — no body shape, no retry and no encoding trick will help.

Say so to the human rather than working around it silently. Whoever uploads outside MCP gives up everything this server provides on the way: the confirm gate, the local permission check, `dryRun`, and the audit line. If a task needs dozens of images, that is worth stating before it starts, not after.

Everything below still applies to that outside upload — it is the same endpoint.

## Uploading

```text
cms_api_search { "query": "files" }
```

The binary goes in the multipart body; **everything else is a query parameter** — `type`, `entity`, `id`, `compress`, `edit` and `template`. The field name of the binary part is not enforced by the instance, so `file` and `files` both work.

An upload returns a record describing the stored file: its identity, its name, a download link and — only under the conditions in the next section — a preview link. Keep the **whole record**; that is what an attribute value references and what a site renders.

Uploads consume instance storage quota, so an upload made by mistake is not free. Do not re-upload a file to "make sure it worked"; read the record back instead.

## No preview template no preview and no error

`template` is **the numeric id of a `/template-previews` record**. It is not a flag, and `template=1` is not an incantation — it means "preview template number one", which on a fresh instance does not exist.

A fresh instance has **no template-previews records at all**. Upload an image without a valid template id and the file is stored correctly, `previewLink` is simply absent, and nothing reports a problem. Thirty images later that is thirty images a site cannot render as thumbnails.

So, once per instance, before the first image:

1. `GET /template-previews`. If it is empty, create one — title, identifier, and the proportions you want generated.
2. Note the **numeric id** from the response. A marker is not accepted here; a non-numeric `template` fails on the database type.
3. Upload with `template=<that id>`.
4. Read the returned record and confirm `previewLink` is present. If it is not, the template id is wrong — do not vary `type` and `entity` hoping for a different outcome, they have nothing to do with it.

Previews are generated for `png`, `jpeg` and `jpg` only. Documents get no preview by design, which is why an upload of a PDF never carries one.

→ `mcp/docs/api/templates-and-previews` · `mcp/docs/api/baseline-data`

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

→ `mcp/docs/api/verification-recipes`
