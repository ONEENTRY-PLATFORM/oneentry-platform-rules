# Where the operation catalog comes from

The list of operations this server can call is built at runtime from the instance you point at, not shipped inside the package. That is why it always describes the API that is actually running — and why an unreachable instance produces an empty catalog rather than a stale guess.

Read this when `cms_api_search` finds nothing, or when `cms_whoami` reports catalog warnings.

→ `mcp/docs/server/cms-api-search` · `mcp/docs/server/errors-startup`

## Two sources

**The instance's own API document.** Requested once at startup from the Admin API, under `workflows/nodes/admin-api/swagger.json` relative to the configured base URL. It supplies paths, operation ids, parameters, body schemas and response descriptions.

**A bundled permission map.** The permission each operation requires is not expressed in that document, so the package ships a map from operation id to permission string. It is generated from the platform sources at build time and updated with the package.

The two are merged into one catalog: every operation, with its risk classification, its declared permission, and whether it is permanently confirm-gated.

## Paths outside the Admin API are dropped

Anything not under `/api/admin` is removed from the catalog and reported as a warning. This server exposes the Admin API only, and that is enforced when the catalog is built rather than left to the caller.

So a Content API route is never a search result, however real it is. `cms_api_search` returning nothing for one is not evidence it does not exist — the public routes a storefront uses are documented under `mcp/docs/api/content-api-reads`, `mcp/docs/api/content-api-sign-in-and-cart` and `mcp/docs/api/files-and-uploads` instead.

## The catalog is cached per base URL

The fetched API document is cached on disk under the cache directory, keyed by the base URL, so two instances never share a catalog. On a later start the cache is used if the instance cannot be reached, and `cms_whoami` reports that:

```text
The instance at … was unreachable; this catalog came from the on-disk cache and may not
match the API that is actually running.
```

A cached catalog is better than none, but treat an unknown-operation error against a cached catalog as "possibly stale", not "does not exist".

## When the catalog is empty

If the instance served no API document and there is no cache, the catalog is empty and says so loudly:

```text
The operation catalog is EMPTY: … did not serve its API document (…). No API operation
can be searched or called until the instance is reachable.
```

Empty is safe — there is nothing to call — but silence would read as "the platform has no such endpoint", which is why the warning is explicit. Nothing works until this is fixed: check the base URL, whether the instance is up, and whether it exposes the Admin API.

In remote mode a server that started without credentials fills its catalog from the first session that authenticates successfully, so this state can resolve itself without a restart.

## Catalog warnings and what they mean

`cms_whoami` reports them; `cms_guide` shows them at the end of the briefing.

| Warning | Meaning | What to do |
|---|---|---|
| Paths outside the Admin API were dropped | Normal. The document describes more than this server exposes | Nothing |
| Operations have no operation id and cannot be called | Those endpoints are unreachable through this server | Report it; there is no client-side workaround |
| Handlers declare a permission but are absent from the instance's API document | The bundled permission map is newer or older than the instance | Usually harmless. If the catalog looks incomplete, the package needs updating |
| Permissions are required by operations but cannot be held by any admin | Those operations answer 403 permanently | Treat them as unavailable and tell the human |
| The catalog came from the on-disk cache | The instance was unreachable at startup | Restart once the instance is up |

## The API document is not valid JSON Schema

Some declared types in the source document are expressions in the platform's own type language rather than JSON Schema types. The build normalises what it can and marks the rest `"x-loose": true`, keeping the original under `x-source-type`.

For those fields the `example` is the contract, and client-side validation of your body is deliberately advisory — the instance is the real validator, so a call is never blocked because a loose field could not be checked.

→ `mcp/operating-rules#trust-the-example-not-the-type`

## What the catalog does not tell you

It describes shape, not meaning. It cannot tell you that attribute values are two levels deep, that a position is a lexorank string, or that a create you are about to make already exists on the instance.

Three limits are worth knowing before you plan around the catalog:

- **Present does not mean callable.** The file upload operation is listed because it exists, but its body is `multipart/form-data` and this server sends JSON only. `cms_api_describe` marks such an operation `notExecutable`.
- **The document contradicts itself in places.** Where a field's `example` does not satisfy its own declared type, the catalog marks the field `x-example-mismatch` and lists it in `cms_api_describe`. The example is the contract; the type is the defect.
- **Some behaviour is curated, not read.** A handful of operations carry a `note`, a `silentNoOp` or a `verifyWith` written by hand, because no schema can say which of two valid-looking bodies the instance accepts, or that a write answers success without doing anything.

That is what the rest of this corpus is for: find the operation here, then read the document for the entity before building the body.

→ `mcp/docs/api/silent-no-ops` · `mcp/docs/api/files-and-uploads`
