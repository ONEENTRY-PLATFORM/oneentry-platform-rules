# Uploading files with cms upload file

Two tools put a file into an instance: `cms_upload_file` sends one from the machine running the server, `cms_import_file_from_url` has the server fetch it first. Both then call the same upload endpoint.

Read this before any task that involves images, and before the first upload of a migration — one query parameter decides whether the files get previews, and it cannot be fixed afterwards.

→ `mcp/docs/api/files-and-uploads` · `mcp/docs/api/templates-and-previews`

## Why uploading has tools of its own

`cms_api_call` serialises every request body as JSON, and the upload endpoint wants `multipart/form-data`. Rather than sending agents outside the server — where the allow level, the confirm gate, the local permission check and the audit line do not exist — uploading has two tools that keep all of it.

An upload is a **write**: it is refused at `--allow=read` before anything is read from disk or fetched from the network, it is audited, and `dryRun` works on it exactly as on `cms_api_call`.

## Uploading a file from this machine

```text
cms_upload_file { "path": "assets/mug.png", "type": "image", "entity": "product",
                  "id": 512, "template": 3, "dryRun": true }
```

- `path` is absolute or relative to the server's upload root;
- `type`, `entity` and `id` say what the file is and what it belongs to;
- `template` is the **numeric id** of a preview-template record — see below;
- `compress` asks the instance to compress an image, `edit` replaces an existing file.

Local mode only. In remote mode the filesystem belongs to whoever hosts the server rather than to the caller, so the tool refuses and names the other one.

## Importing a file from an address

```text
cms_import_file_from_url { "url": "https://source.example/img/mug.png",
                           "type": "image", "entity": "product", "id": 512, "template": 3 }
```

The server downloads the file and forwards it in one step, which is the whole of "take this picture from the customer's site into the CMS". `filename` overrides the name taken from the address.

A dry run does **not** download anything — it reports the request it would send and the address it would fetch.

## What both tools refuse

| refusal | why |
|---|---|
| a path outside the server's upload root, symlinks resolved | the path is a tool argument, and a tool argument can be planted by content the agent read |
| a file over the configured size limit | checked both on disk and against what actually arrives |
| a scheme other than http or https | nothing else is fetched |
| an address resolving into the network the server runs in | loopback, private, link-local and carrier-grade ranges, re-checked on every redirect hop |
| a host outside the operator's allowlist, where one is set | in remote mode the URL import stays off until the operator sets it |

A refusal here happens before anything is read or sent, and says so. Report it to the human — none of these are worked around by rephrasing.

## Get a preview template id first

`template` is the numeric id of a preview-template record, not a flag and not a filename. A fresh instance has **no such records**.

Upload an image without a valid id and the file is stored correctly, no preview link is generated, and nothing reports a problem. The only repair is uploading the file again — so on a run of any size this is the first thing to settle:

1. read the preview templates; if there are none, create one;
2. take the numeric id from the response;
3. upload with that id and confirm the returned record carries a preview link.

The preview matters beyond thumbnails: the inline placeholder a site shows while the full image loads comes from it, so an image without one cannot be rendered progressively.

## What to do with the record you get back

Keep the **whole record** — that is what an attribute value references and what a site renders. Storing only the download link loses everything else about the file.

The record carries no field for alternative text. Decide once per project where that lives — a sibling attribute next to each image field, named consistently — and write the convention down, because nothing in the platform enforces it and every project otherwise invents its own.

→ `mcp/docs/api/attribute-types#files-and-images-depend-on-the-count`

## When something looks wrong

| symptom | first thing to check |
|---|---|
| refused with no request sent | the allow level, then the source bound named in the message |
| stored but no preview link | the `template` id, not `type` or `entity` |
| the site shows a missing image | the whole record was stored in the attribute, not just a link |
| the file cannot be found afterwards | it was uploaded to another instance; references do not travel |

→ `mcp/docs/api/bulk-content-migration#uploading-many-files` · `mcp/docs/server/allow-levels`
