# Documentation corpus for `@oneentry/mcp-platform-server`

Two repositories:

- **Part A** — `oneentry-platform-rules` (this working directory): the public documentation corpus.
- **Part B** — `dnk-mcp` @ `e910da7` (v0.1.1): remove the doc relations, clean the strings the agent actually sees, and align the seed. Part A is correct without Part B, but the two overlap badly until Part B ships.

## Context

`@oneentry/mcp-platform-server` v0.1.1 is published on npm. Unlike v0.1.0, it **ships almost no content**: at startup it resolves `ONEENTRY_MCP_KNOWLEDGE_REPO` (default `ONEENTRY-PLATFORM/oneentry-platform-rules`) at ref `main`, downloads the repo tarball from codeload, and indexes **only `knowledge/**/*.md`**.

That repository — this working directory — **is empty** (one empty `README.md`, one `init` commit; the GitHub remote returns 409 "Git Repository is empty"). Consequence today: every running server falls back to a bundled seed containing a single document, and `cms_guide` prints `> **Knowledge is degraded.**`.

The outcome we want: this repo becomes the complete, public, hand-authored documentation for the server and for operating the OneEntry Admin API through it — structured like the sibling repo `ONEENTRY-PLATFORM/oneentry-sdk-rules`, which is the same thing for `@oneentry/mcp-server`. It must contain nothing about our source, database, infrastructure or tracker.

### Two decisions that shape this plan

1. **Doc relations are removed from the server (Part B).** The package hard-codes a tag→docIds map (`TAG_DOCS` in `src/api/build-catalog.ts`) that puts 25 `back/docs/*`, `back/tests/*` and `front/docs/*` ids into `docLinks`, surfaced as `cms_api_describe`'s `docs` array, `cms_api_call`'s error `docs`, and the 400/422 hint *"Read `<docLinks>` with cms_docs_read before retrying."* Those relations go away, so **this corpus does not create `back/**` or `front/**` namespaces at all.** With `docs` hints gone, `cms_docs_search` becomes the only navigation path — which makes the corpus index document and the heading discipline below load-bearing rather than nice-to-have.

2. **The corpus must document baseline data an agent cannot recreate.** A large amount of data already exists on every provisioned instance (the `Guest` user group and its ~110 content-API permission rows, 18 admin modules, ~24 general types, 11 attribute set types, 19 attribute field types, 182 locale entries with only `en_US` active, 9 dynamic block types and their 9 default templates, and several singleton settings records). Some of these fail loudly on a duplicate create; the dangerous ones **succeed silently** and produce a shadow record that nothing points at. This gets its own document and its own rule in the operating rules.

**Out of scope** (per the user): validator, CI workflows, and repo governance files (`CONTRIBUTING.md`, `SECURITY.md`, `CODEOWNERS`) — conventions are enforced by review.

---

# Part A — `oneentry-platform-rules`

## The publishability rule (governs every sentence)

Three filters, applied in order. All three must pass.

1. **Black-box.** Could a competent stranger who has an admin login to one instance, that instance's OpenAPI document, and time to experiment have written this sentence? If it required reading our source, database, cluster, migrations, or tracker — cut it.
2. **Causeless.** Publish what the agent *observes* and what it must *do*. Never why the system behaves that way. "A read right after a write may not show the write; re-read by id" ✅. "…because values reach the index via a queue" ❌.
3. **Necessity.** Even if it passes 1 and 2, cut it unless an agent would take a wrong action without it. This is an operating manual, not a reference work.

**Publishable:** operation ids exactly as the catalog emits them (`AdminProductsController_findAll` — the agent cannot function without them), URL paths, parameter and field names, enum values, permission strings, status codes, wire-format conventions (`localizeInfos` locale keying, two-level `attributesSets`, `<type>_id<attrId>` keys, lexorank `position`, `marker` vs `id`), observable timing, the closed vocabularies an agent must choose from, and this server's own client-side policy (allow levels, gating, truncation, dry-run).

**Never publishable:** table / column / view / index / queue / cache names and any DDL; **migration and seed file names, timestamps, or the fact that a specific migration created a specific row**; framework, ORM, search-engine, broker or vector-store names; source file paths, line anchors, class / decorator / DTO / enum *type* names; internal hostnames, ports, deploy or k3s steps; issue ids, branch names, MR text, dated specs, personal emails; internal repo names (`dnk-back`, `dnk-front`, `dnk-mcp`, `cms-sb-back`); verbatim server error strings that embed any of the above; defect framing ("bug", "broken", "known defect", root causes); counts that imply internal scale.

Bare `AdminXController` without a `_method` suffix is a source-code reference and must not appear. The `_`-joined operationId form is the sole exception.

### Public vocabulary

| Concept | Use | Never |
|---|---|---|
| The backend | the OneEntry platform; the Admin API | `dnk-back`, `cms-sb-back`, "the NestJS backend" |
| The admin web UI | the OneEntry admin panel | `dnk-front`, "the admin SPA" |
| One deployment | **instance** ("your instance") | **stand**, "dev stand", named hosts |
| This server | the OneEntry Platform MCP server / this server | `dnk-mcp`, `mcp-oneentry` |
| Person running it | operator | "devops", team names |
| Pre-existing rows | **baseline data**, "provisioned with the instance" | "seeded", "created by migration", seed/migration names |
| A broken route | "the supported path is…", "not available on current instances" | "known defect", "bug", "workaround" |
| Storage / search | say nothing; describe timing only | Postgres, Elasticsearch, Redis, Bull, pgvector |

---

## Mechanical constraints (verified against the 0.1.1 source at `dnk-mcp@e910da7`)

These are not style preferences — violating them breaks retrieval.

- Only `knowledge/**/*.md` is indexed. `README.md` and `CLAUDE.md` at the root are **invisible to the server**; anything an agent needs at runtime must live under `knowledge/`.
- `docId` = path under `knowledge/` minus `.md`. `foo/index.md` collapses to `foo` — **ban `index.md`** (it can silently collide with `foo.md`). Ban `README.md` under `knowledge/` too.
- Sections split at `##` **and `###`**. Use `##` only; `####` and deeper do not split and are safe for inner structure.
- Text before the first `##` becomes the section with the **empty anchor** — this is what `cms_docs_read { docId }` returns. A file that opens straight into a heading has **no such chunk**, and the natural read fails with `Document "…" has no section ""`. **Every file needs a preamble.**
- Anchor slug: lowercase, a fixed punctuation set stripped, spaces → `-`, truncated at 80 chars. Em dashes, slashes and emoji **survive into the anchor** and make it untypable. Headings: letters, digits, spaces, hyphens only; ≤ 60 chars; no leading numbers (numbering makes anchors positional — every insertion becomes a breaking change); unique *after* slugging (`## Create` and `## Create:` both slug to `create`; the second silently becomes `create-2`).
- `cms_docs_read` truncates a section over **12288 bytes**. Work to **file ≤ 10 KB, section ≤ 8 KB**. Byte counts, not characters.
- An **unbalanced code fence** makes the splitter treat every later `##` as body text and collapses the rest of the file into one truncated section. Tag and balance every fence.
- Ranking: minisearch over title/docTitle/docId/text with boosts 4/2/3/1, prefix + fuzzy 0.2, `combineWith: OR`. Query tokens of ≤ 2 alphanumeric chars are dropped. Post-multipliers by path **segment 1** (`mcp` ×1.6, `back` ×1, `front` ×0.9; anything else ×1) and **segment 2** (`docs` ×1.15, `tests`/`rules` ×1, `fixes`/`specs` ×0.6; anything else ×1). Segments beyond the second do not affect ranking.
- Because search is OR-combined, fuzzy and prefixed, **repeated boilerplate poisons the index** — a shared "see also" footer listing twenty docIds makes every document a weak match for every query. One pointer section per document, 2–5 links.
- `oneentry://knowledge/mcp/operating-rules` concatenates **all** sections of that docId into one resource. Keep it ≤ 7.5 KB total, ≤ 700 B per section.
- No YAML frontmatter under `knowledge/` — it is not parsed, it lands verbatim in the preamble and is shown to the model.
- English only. Retrieval does not translate, and the tool descriptions, refusal messages, operation ids and swagger summaries the agent is holding are all English. Where source material exists only in Russian, **translate it** — never carry Cyrillic across.

### Namespace choice

Everything goes under **`knowledge/mcp/docs/…`**, split into two subtrees:

- `knowledge/mcp/docs/server/…` — the MCP server itself
- `knowledge/mcp/docs/api/…` — operating the Admin API through it

Rationale: `kindOf` reads only segments 1 and 2, so both subtrees score `1.6 × 1.15 = 1.84` — the global maximum — and neither is handicapped relative to the other. Any other top-level segment (`admin-api/`, `platform/`) would silently penalise the domain guides to ×1.15, which is backwards: those are the documents an agent needs most. Depth 4 is free.

The single exception is `knowledge/mcp/operating-rules.md`, which must keep exactly that docId — it is hard-wired in four places (the resource URI, the server's `instructions` string, the `cms_guide` text, and the bundled seed loader) and scores ×1.6. That is unavoidable; compensate by giving it headings no other document competes for and never duplicating its content elsewhere.

---

## File tree

### Root (humans; not served)

| Path | Purpose | Size |
|---|---|---|
| `README.md` | GitHub/npm landing page: what this repo is, that the server fetches it at runtime from `main`, the five mechanical rules, the namespace layout, the no-bulk-import rule, contributor quickstart. | ~4 KB |
| `CLAUDE.md` | Navigator for an agent editing *this* repo, modelled on the sibling repo's: mandatory checklist, mechanics table, ✅/❌ hard rules, the redaction denylist, "docIds are a public API", "`main` is production", when to stop and ask. | ~12 KB |

`README.md` opens with the sibling repo's framing, adapted: *"This is the content source for `@oneentry/mcp-platform-server` — the server ships no content of its own, it fetches these files at runtime."* It also carries, above the fold: *"Public documentation, hand-authored. Do not import from internal repositories."* — the package has a `publish:knowledge` script that bulk-copies internal `.claude` trees into `--out`; running it against this repo would publish table names, source paths and issue ids in one command, clobber curated files, and mint new docIds through its 12 KB splitter. Nothing here is ever `cp`-ed in.

Do **not** put the tool reference, flag matrix or payload rules in `README.md` — they would be invisible where they matter.

### `knowledge/mcp/operating-rules.md` — hard-wired, ≤ 7.5 KB

The must-read before any write, concatenated whole into an MCP resource.

### `knowledge/mcp/docs/server/` — the server itself (22 files, ≈ 135 KB)

| docId (under `mcp/docs/server/`) | Purpose | Size |
|---|---|---|
| `doc-map` | Every docId with a one-line "read this when". **Load-bearing now that `docs` hints are gone** — this is how an agent discovers the corpus. | 4 KB |
| `getting-started` | Install (npx / global), `.mcp.json` for local mode, first three tool calls. | 6 KB |
| `configuration` | Flag × env × `oneentry-mcp-platform.json` matrix; precedence; base-URL rule; never pass a password as a flag. | 9 KB |
| `remote-mode` | Streamable HTTP, `x-cms-token` / `x-cms-login`+`x-cms-password`, session isolation, Origin allowlist, `GET /health`, mandatory audit. | 7 KB |
| `authentication` | login/password vs pre-issued token, the single 401 retry, `isDeveloper` rejection, where the permission map comes from. | 5 KB |
| `allow-levels` | read/write/destructive, risk from HTTP method, refusal *before* auth, the permanently gated path list, local permission pre-check. | 7 KB |
| `confirm-and-dry-run` | 24-hex tokens, 5 minutes, single use, bound to `sha256(opId + args)`, reading the `target` preview. | 6 KB |
| `audit-log` | JSONL shape, hashed args, the three outcomes, why it is required remotely. | 4 KB |
| `cms-guide-and-whoami` | The two zero-argument tools; reading warnings, versions, knowledge origin. | 4 KB |
| `cms-docs-search-and-read` | Searching this corpus, phrasing a query for *this* index, preamble-then-anchor paging, `sections[].bytes`, the two MCP resources. | 6 KB |
| `cms-api-search` | Keyword / tag / method / mutating filters; the only authority on what exists. | 4 KB |
| `cms-api-describe` | Params, body schema, permission, risk, `looseFields`, `alwaysConfirm`. | 5 KB |
| `cms-api-call` | Arguments, reads, dry-run first, confirm, refusals, truncation, worked example. The most-read doc. | 9 KB |
| `payload-conventions` | `localizeInfos`, two-level `attributesSets`, lexorank vs numeric `position`, `marker` vs `id`, `x-loose`, eventual consistency, read-back. | 9 KB |
| `response-shaping` | 24 KB cap, `_truncated` envelopes, blob stripping, `target` summarisation, narrowing instead. | 4 KB |
| `api-catalog` | Runtime build from the instance's own OpenAPI document, the bundled permission map, warning categories, empty-catalog behaviour, per-base-URL cache. | 7 KB |
| `knowledge-subsystem` | GitHub fetch, ref pinning, TTL, offline, `--knowledge-path`, seed fallback, "degraded", what search cannot do. | 6 KB |
| `authoring-knowledge` | The rules for this repo, served at runtime so an agent told "fix the docs" obeys them. Runtime-visible subset of `CLAUDE.md`. | 7 KB |
| `errors-and-refusals` | Every tool-level refusal and error → cause → the single next action. Found when a human pastes an error, so headings carry the message's distinguishing words. | 8 KB |
| `errors-startup` | Config, connectivity, empty catalog, degraded knowledge, cache failures. | 6 KB |
| `agent-workflows` | End-to-end recipes chaining tools and citing domain docIds. | 8 KB |
| `glossary` | Terms an agent meets in payloads and messages: marker, locale code, attribute set, lexorank, loose field, confirm token, allow level, instance, baseline data. | 4 KB |

### `knowledge/mcp/docs/api/` — operating the Admin API (30 files, ≈ 190 KB)

| docId (under `mcp/docs/api/`) | Purpose | Size |
|---|---|---|
| **`baseline-data`** | **⭐ What already exists and must never be created.** See the outline below. | 9 KB |
| `locales` | Reading active locale codes; activate, never create; never hardcode a locale. | 4 KB |
| `attribute-sets` | Sets, `attributeSetId`, the audience type, reading a set before building a payload. | 8 KB |
| `attribute-types` | The closed list of field types and the value shape each one returns and accepts. | 8 KB |
| `general-types` | What a general type is, how to list them, why you pick rather than create. | 5 KB |
| `pages` | Page tree, root vs child, general type, positions, page config. | 9 KB |
| `products` | Create / update / list; the real list operation; the quick-search shortcut. | 9 KB |
| `product-relations` | Relation templates and conditions at API level. | 5 KB |
| `product-statuses` | Statuses and marker validation. | 4 KB |
| `blocks` | Blocks, markers, attachment to pages, ordering, per-type settings. | 8 KB |
| `block-types` | The dynamic block types, their content endpoints, and their default templates. | 6 KB |
| `menus` | Menus, items, polymorphic parents, lexorank reordering operations. | 6 KB |
| `forms-and-form-data` | Forms (including the wrapped create payload), field configuration, submissions. | 8 KB |
| `orders` | Order composition, creation, listing, filtering, paging. | 9 KB |
| `order-statuses` | Status axes, allowed transitions, the globally unique identifier trap. | 7 KB |
| `discounts` | Discount and coupon payloads and their enums. | 6 KB |
| `payments` | Accounts, account types, sessions, why webhook paths are gated. | 7 KB |
| `subscriptions` | Plans and user subscriptions. | 5 KB |
| `filters` | Filter and sort payloads for list operations. | 6 KB |
| `index-attributes` | Which attributes are searchable, and when a written value becomes visible in lists. | 6 KB |
| `global-search` | Cross-entity search operations and their limits. | 4 KB |
| `templates-and-previews` | Templates, template previews, the default templates you inherit. | 6 KB |
| `users-and-groups` | Users, groups, the guest group, group permissions. | 6 KB |
| `admins-and-permissions` | Permission strings, grants, what a denial means, why some are ungrantable. | 8 KB |
| `modules` | The admin modules you inherit and why you address them by identifier. | 5 KB |
| `settings` | Singleton settings records — update only, never create. | 5 KB |
| `files-and-uploads` | Upload operations, response shapes, blob stripping in this server. | 5 KB |
| `import` | Import templates and sessions. | 5 KB |
| `ai-gateway` | AI Gateway operations and provider accounts. | 6 KB |
| `verification-recipes` | How to prove a change worked through the public Admin API: call these operations in this order against a throwaway entity, assert these fields, then delete it. Replaces what used to be the `back/tests/*` ids. Zero internal test detail. | 7 KB |

**Total: 53 files under `knowledge/`, ≈ 330 KB, plus 2 root files.**

---

## `mcp/docs/api/baseline-data` — the document the user asked for

Every OneEntry instance arrives already populated. An agent that treats the API as an empty database will either hit a unique-constraint failure or — worse — silently create a shadow record that nothing points at. This document is the antidote.

**Framing rules:** describe baseline data as *"provisioned with the instance"*. Never mention migrations, seeds, run commands, file names, timestamps, or that a particular row came from a particular script. Never publish table or column names. Present identifiers as **API-visible** values and always tell the agent to confirm them with a read against its own instance, because provisioning paths differ and ids are not guaranteed across instances.

Section outline:

```
## Read before you create anything
## The rule that matters most
## Silent duplicates versus hard failures
## User groups and the guest group
## Content API permissions
## Admin permission keys
## Modules
## General types
## Attribute set types and field types
## Locales
## Block types and their default templates
## Settings records are singletons
## Order statuses are globally unique
## What may be missing on a minimal instance
## Every needless record consumes instance quota
## How to check what your instance already has
```

Content the document must carry:

- **The rule:** *list first, create second.* Before creating any of the entity kinds below, list them and match by identifier. If it exists, use it. If a "create" looks necessary, say so to the human and stop.
- **`## Silent duplicates versus hard failures`** — the single most valuable table in the corpus, because a hard failure is self-correcting for an agent and a silent duplicate is not:

  | Creating a duplicate of… | What happens |
  |---|---|
  | locale code, general type name, template identifier, menu identifier, order status identifier, import template identifier, a permission path within a group, a group↔permission link | **fails loudly** — treat as "it already exists" |
  | **user group identifier**, **module identifier**, **attribute set type name** | **succeeds silently** — you now have a shadow record that nothing references |
  | singleton settings records | **succeeds silently** — a second record appears and the platform reads only one |

- **Guest group** — a user group with identifier `guest` (commonly id 1) exists and every content-API permission is attached to it. Its identifier is **not unique**, so creating another `guest` group succeeds and produces a group that no permission points at. To open up guest access, grant permissions to the existing group; never create one. A second group with identifier `user` exists on fully provisioned instances but may be absent on minimal ones — check, do not assume.
- **Content API permissions** — roughly 110 permission records already exist, one per content-API path, each grouped by section, all attached to the guest group. The section value is a closed vocabulary; an agent cannot invent one. A permission path is unique per group, so re-creating one fails. The correct action is to adjust the rules on the existing record.
- **Admin permission keys** — a fixed vocabulary of dotted strings (`orders.get`, `blocks.preview`, `settings.attributes.create`, …). Read the admin's own permission map to see what is held. Never invent a key, and never replace the map wholesale. `orders.export`, `payments.export` and `users.export` are not grantable on current instances — treat those operations as unavailable and tell the human rather than retrying.
- **Modules** — 18 admin modules exist, addressed by identifier (`settings`, `forms`, `catalog`, `content`, `admins`, `blocks`, `journal`, `menu`, `users`, `payments`, `events`, `orders`, `workflows`, `collections`, `discounts`, `import-data`, `subscriptions`, `filters`). Identifiers are not unique — a duplicate creates a shadow module. Modules are also on the permanently confirm-gated path list, so mutations require explicit human confirmation regardless of allow level.
- **General types** — the page / block / product / form / order / discount types are provisioned, addressed by name (`product`, `common_page`, `catalog_page`, `error_page`, `external_page`, `common_block`, `product_block`, `similar_products_block`, `product_preview`, `form`, `order`, `service`, `discount`, plus the nine dynamic block types). The name **is unique** — creating one fails. Ids are not contiguous and differ by instance: resolve by name.
- **Attribute set types and field types** — attribute set types (`forAdmins`, `forBlocks`, `forOrders`, `forPages`, `forProducts`, `forUsers`, `forForms`, `forUserGroups`, `forEvents`, `system`, `forDiscounts`) exist and their names are **not** unique, so a duplicate is silent and ambiguous. Field types are a closed list of 19 (`string`, `text`, `textWithHeader`, `integer`, `real`, `float`, `dateTime`, `date`, `time`, `file`, `image`, `groupOfImages`, `radioButton`, `list`, `spam`, `button`, `entity`, `timeInterval`, `json`) — pick from it, never extend it.
- **Locales** — 182 locale entries exist and only `en_US` is active. Codes are unique, so creating one always fails. There is effectively nothing to create: the operation you want is *activate*.
- **Block types and default templates** — the nine dynamic block types each ship a default template whose identifier is `<block_type>_default`. Template identifiers are unique, so re-creating one fails; blocks of that type are already pointed at it, and hand-rolling a replacement diverges from what the platform expects.
- **Settings records** — general settings, plan limits, usage counters, discount settings and event e-mail settings are **singletons**. Update them; a create succeeds silently and produces a second record.
- **Order statuses** — the identifier is unique **across the whole instance**, not per order storage, so two storages cannot both have a `new`. On instances that have been upgraded you will also find payment-axis statuses named `<identifier>-payment` that you did not create — do not duplicate them.
- **What may be missing** — on a minimal instance, the starter content (pages, products, order storages, payment accounts, the second user group, the default block attribute set) is absent. Check before assuming; report the gap to the human rather than reconstructing it.
- **Quota** — instances enforce a total-record limit across the business entity types. Every needless duplicate consumes it, and exceeding it makes *all* further creates fail, including unrelated ones. This is the practical reason "list first, create second" is not merely tidy.
- **`## How to check what your instance already has`** — the read operations to run first (`cms_api_search` for `locales`, `general-types`, `modules`, `user-groups`, `attributes-sets`, `templates`, then `cms_api_call` on each), and the reminder that this list, not this document, is the truth for a given instance.

The operating rules get a matching one-paragraph section, `## Baseline data already exists do not recreate it`, pointing here.

---

## Section outlines for the other load-bearing documents

Headings are the retrieval unit at boost 4 — write each as the phrase an agent would type. Flat `##` only.

**`mcp/operating-rules`** (≤ 7.5 KB; the seed's `## 0.`…`## 12.` numbering and its `§n` cross-references must **not** be carried over):
`## The loop you must follow` · `## Trust the example not the type` · `## Content is locale keyed` · `## Attribute values are two level and locale keyed` · `## Positions are lexorank or numeric depending on the endpoint` · `## A read straight after a write can lag` · `## Prefer marker over id` · `## Baseline data already exists do not recreate it` · `## Never touch these without a human saying so` · `## Permissions are checked before the request is sent` · `## Truncated responses are deliberate` · `## Operations with a single supported path` · `## Where to look next`

`## Operations with a single supported path` replaces the seed's "Known defects" section — keep the turn-saving instruction, drop the disclosure. No "defect/bug/broken", no cause, no verbatim 5xx body (those strings carry column names), no internal repo name:

> For the calls below, one route works and the obvious alternative does not. Use the supported one directly — a different body or a retry on the other route will not help.
> - **Create a form** — send the payload wrapped in `newForm`. The OpenAPI document shows the fields unwrapped; the wrapped form is the one the endpoint accepts.
> - **Update a product** — always include `blocks` (send `blocks: []` when you have nothing to set), and never include `forms`.
> - **Read locale codes** — use `AdminLocalesController_findAllActive`.
> - **List products** — the list operation is `POST /products/all`; there is no `GET /products`.
> - **Export operations** — `orders.export`, `payments.export` and `users.export` are not grantable on current instances; treat those operations as unavailable and tell the human rather than retrying.
>
> If a call outside this list answers 5xx, stop and report it with the operation id and the request you sent. Do not retry it with a modified body.

The `## Where to look next` section is now the corpus's primary navigation aid — with `docs` hints removed from `cms_api_describe`, it and `mcp/docs/server/doc-map` are how an agent finds anything.

**`mcp/docs/server/cms-api-call`**: `## Arguments` · `## Calling a read operation` · `## Always dry run a mutation first` · `## Reading the dry run target` · `## Confirm tokens` · `## Refusals and what each one means` · `## Response size and truncation` · `## What gets written to the audit log` · `## Worked example create then delete a template` · `## Mistakes that cost a retry`

**`mcp/docs/server/allow-levels`**: `## The three levels` · `## What each level permits` · `## Risk is derived from the HTTP method` · `## A level refusal happens before authentication` · `## Paths that are always confirm gated` · `## The local permission pre check` · `## Choosing a level when you run the server` · `## Asking the operator to raise the level`

**`mcp/docs/server/configuration`**: `## Precedence flags then environment then file then defaults` · `## Command line flags` · `## Environment variables` · `## The oneentry-mcp-platform json file` · `## The base URL must point at the Admin API` · `## Never pass a password as a flag` · `## Minimal local configuration` · `## Minimal remote configuration` · `## Checking what the server actually resolved`

**`mcp/docs/server/remote-mode`**: `## Starting the server over HTTP` · `## Identity is per session and comes from headers` · `## Why credentials are never tool arguments` · `## What is isolated between sessions` · `## The Origin allowlist` · `## The health endpoint` · `## The audit log is mandatory here` · `## Running behind a reverse proxy` · `## Failure modes and what they look like to the client`

**`mcp/docs/server/payload-conventions`**: `## localizeInfos is keyed by locale code` · `## Get the active locale codes first` · `## attributesSets is two levels deep` · `## attributeSetId and the audience type id` · `## position is a lexorank string or a number` · `## marker is portable id is not` · `## x-loose fields where the example is the contract` · `## Writes are eventually consistent` · `## Read the entity back after creating it` · `## Narrowing a list instead of asking for everything`

**`mcp/docs/server/errors-and-refusals`**: `## How to read an error from this server` · `## Refused because of the allow level` · `## Refused because of a missing permission` · `## Needs a confirm token` · `## Confirm token expired used or mismatched` · `## Unknown opId` · `## Unknown docId` · `## Could not build the request` · `## Authentication failed` · `## The CMS returned an error` · `## A truncated response is not an error`

**`mcp/docs/server/knowledge-subsystem`**: `## Where this knowledge comes from` · `## How a document becomes searchable` · `## docId anchors and sections` · `## The cache and its lifetime` · `## Running offline or from a local path` · `## The bundled seed and the degraded warning` · `## What the search cannot do` · `## Reporting a gap in the documentation`

**`mcp/docs/server/authoring-knowledge`**: `## Where a new document goes` · `## Size limits that are not negotiable` · `## Headings decide whether anything is found` · `## Every document needs a preamble` · `## Linking between documents` · `## Language policy` · `## What must never be published` · `## Before you commit`

**`mcp/docs/api/orders`**: `## What an order is made of` · `## Finding the operations` · `## Creating an order` · `## Attaching a payment` · `## Listing filtering and paging orders` · `## Discounts on an order` · `## Verifying an order change`

**`mcp/docs/api/index-attributes`** (the most leak-prone file — every sentence must be observable API behaviour, no mechanism, no component names): `## What an index attribute is` · `## Declaring an attribute as indexed` · `## When a written value becomes searchable` · `## Reading indexed values back` · `## Why a value you just wrote is missing from a list` · `## What not to conclude and what not to do`

---

## Authoring conventions

- **No frontmatter** under `knowledge/`. One `# H1` on the first non-blank line (it becomes `docTitle`, boost 2 on every section — it must carry the subject noun). Mandatory preamble of 2–6 lines ending in 2–4 pointers.
- **`##` only.** Never `###`. `####`+ is safe for inner structure.
- Headings: letters, digits, spaces, hyphens; ≤ 60 chars; no numbering; unique after slugging; phrased as the query (`## Refused because of the allow level`, not `## Refusals`).
- File ≤ 10 KB, section ≤ 8 KB, sweet spot 300–1500 B per section, 8–15 sections per document.
- Cross-links in exactly one form: `` → `mcp/docs/api/orders#creating-an-order` `` — the same `docId#anchor` identity that can be pasted straight into `cms_docs_read`. Never relative `.md` links (they invite filesystem reads that fail), never GitHub URLs. Never link an auto-deduplicated `-2` anchor; fix the duplicate heading instead.
- Every fence tagged and balanced. Examples ≤ 25 lines, placeholder data only (`https://your-instance.example`, ids ≤ 3 digits).
- One pointer section per document, 2–5 links, in the final section. No repeated boilerplate anywhere.
- Filenames lowercase ASCII kebab-case.
- Each fact has exactly one home: server mechanics → `mcp/docs/server`; domain semantics → `mcp/docs/api`; cross-cutting payload law → `mcp/docs/server/payload-conventions`, cited by both. Never restate.

---

## Build order

**Phase 0 — stop the degradation.** `README.md`, `CLAUDE.md`, `knowledge/mcp/operating-rules.md`, `mcp/docs/server/doc-map`, `mcp/docs/server/getting-started`, and `mcp/docs/api/baseline-data`. Commit and push — the fix reaches running servers within one cache TTL and clears "Knowledge is degraded".

**Phase 1 — the server corpus.** The remaining 20 `mcp/docs/server/**` files.

**Phase 2 — the API corpus.** The 29 remaining `mcp/docs/api/**` files.

`baseline-data` is pulled into Phase 0 deliberately: it is the document whose absence causes the most expensive class of mistake (a silent duplicate that consumes quota and breaks permission wiring), and it is the one an agent has no other way to learn.

---

## Verification (Part A)

No CMS instance is needed to verify the corpus itself — the server can read this repo from disk.

1. **Structural self-check**, from the repo root:
   - every `knowledge/**/*.md` has one `# H1` on the first non-blank line and non-empty text before its first `##`;
   - `grep -rn '^### ' knowledge/` returns nothing;
   - fence count per file is even;
   - `find knowledge -name '*.md' -size +10k` returns nothing;
   - no `index.md` / `README.md` / non-`.md` file under `knowledge/`;
   - redaction sweep — `grep -rniE 'dnk-|cms-sb|webdnk|gitlab\.com|\b(CMS|DNK)-[0-9]{2,4}\b|src/[A-Za-z0-9_./-]+\.(ts|sql)|#L[0-9]+|migration|seed file|postgres|typeorm|nestjs|elastic|redis|bull|pgvector|materiali[sz]ed view|CREATE (TABLE|VIEW|INDEX)|k3s|kubectl|localhost:[0-9]+|\.oneentry\.cloud|[[:alnum:]._%+-]+@[[:alnum:].-]+\.[a-z]{2,}' knowledge/ README.md CLAUDE.md` must be empty;
   - Cyrillic sweep — `grep -rnP '[\p{Cyrillic}]' knowledge/` must be empty (any hit is near-proof of copied internal text);
   - snake_case sweep — `grep -rnE '\b[a-z][a-z0-9]*(_[a-z0-9]+){1,}\b' knowledge/` reviewed by hand, allowing only wire-format tokens (`en_US`-style locale codes, `<type>_id<n>` attribute keys, general type and block type names such as `common_page`, operationIds, `_truncated`, `_summary`).

2. **Load the corpus through the real server** (no instance required — knowledge resolution happens before any API call):
   ```bash
   npx -y @oneentry/mcp-platform-server --knowledge-path /media/andrvm/disk_d/WebDevelop/oneentry-platform-rules
   ```
   Then, from an MCP client:
   - `cms_whoami` → `knowledge.source` must be `local`, `docs` / `sections` must match the file and heading counts, and there must be no `knowledge.warning`.
   - `cms_guide` → the `Knowledge:` line reads `local clone …`, and the `> **Knowledge is degraded.**` block is gone.
   - `cms_docs_read { docId }` with **no anchor** for all 53 docIds → every one returns a preamble, never `has no section ""`.
   - `sections[].bytes` for every document → nothing over 8192; nothing returns the `…[section truncated]` marker.
   - `cms_docs_search` for `"create a product"`, `"guest group"`, `"already exists"`, `"order status"`, `"confirm token"`, `"allow level"`, `"attributesSets"`, `"lexorank"`, `"truncated"` → the intended document is the top hit each time.
   - Every `→ docId#anchor` cross-link in the corpus resolves via `cms_docs_read` (grep them out and loop).
   - Read the resource `oneentry://knowledge/mcp/operating-rules` → total size ≤ 7.5 KB.

3. **Against a live instance** (once one is reachable): confirm the identifier lists in `mcp/docs/api/baseline-data` against `GET /modules`, `GET /general-types`, `GET /locales`, `GET /user-groups`, the attribute set types listing and the templates listing. Any mismatch means the document is over-specific — soften it to "check your instance" rather than correcting the number.

4. **After push**: a run with default config (no `--knowledge-path`) must report `knowledge.source: github` with the new commit in `cms_whoami`.

---

# Part B — `dnk-mcp` (the package)

Repo: `/media/andrvm/disk_d/WebDevelop/dnk-mcp`, branch `develop` @ `e910da7`, version `0.1.1` (matches what is on npm). Four groups of changes; B1 and B2 are the ones that must ship with Part A.

## B1 — Remove the doc relations

The 25 hard-wired docIds must stop being emitted. Delete the field rather than emptying it, so nothing can quietly re-populate it later.

| File | Change |
|---|---|
| [src/api/build-catalog.ts](../../media/andrvm/disk_d/WebDevelop/dnk-mcp/src/api/build-catalog.ts) | Delete the whole `TAG_DOCS` constant (~L35–61) and the `docLinks: [...(TAG_DOCS[tag] ?? []), 'mcp/operating-rules'],` entry (~L236). |
| [src/api/types.ts](../../media/andrvm/disk_d/WebDevelop/dnk-mcp/src/api/types.ts) | Remove `docLinks: string[]` and its JSDoc from `Operation` (~L62–63). |
| [src/tools/api-discovery.ts](../../media/andrvm/disk_d/WebDevelop/dnk-mcp/src/tools/api-discovery.ts) | Remove `docs: operation.docLinks,` (L89). Change the non-read `next` (L91–92) from *"Read the linked docs, then call cms_api_call with dryRun: true first."* to **"Search the knowledge base with cms_docs_search for this entity, then call cms_api_call with dryRun: true first."** |
| [src/tools/api-call.ts](../../media/andrvm/disk_d/WebDevelop/dnk-mcp/src/tools/api-call.ts) | Remove `docs: operation.docLinks,` from the HTTP-error result (L183). |
| [src/api/client.ts](../../media/andrvm/disk_d/WebDevelop/dnk-mcp/src/api/client.ts) | The 400/422 hint (L97–103) currently ends `Read ${operation.docLinks.slice(0, 2).join(', ')} with cms_docs_read before retrying.` → **`Search the knowledge base with cms_docs_search before retrying.`** (keep the `x-loose` clause). |
| `__tests__/unit/{policy,shape}.test.ts` | Drop the `docLinks: []` fixture field. |
| `__tests__/protocol/mcp.test.ts:155` | Drop `docs: string[]` from the destructured describe result and any assertion on it. |
| `README.md` | Remove the claim that operations carry linked docs, if present in the tool table. |

`mcp/operating-rules` keeps its four other bindings (resource URI, server `instructions`, `cms_guide` step 1, seed loader) — those stay.

## B2 — Clean the strings an agent actually sees

Every one of these is emitted to the model or the operator, and every one carries an internal name. Vocabulary per the Part A table: **instance**, **operator**, **the Admin API**.

| Location | Now | Replace with |
|---|---|---|
| `src/api/client.ts:107` | `Forbidden. Check the admin's permissions and the module visibility rules (back/docs/user-permissions).` | `Forbidden. Check the admin's permissions and the module visibility rules.` |
| `src/api/client.ts:110` | `…may belong to another stand — prefer marker-based…` | `…may belong to another instance — prefer marker-based…` |
| `src/api/client.ts:113` | `Server-side failure. Check the dnk-back logs; do not retry blindly.` | `Server-side failure on the instance. Report it to your operator; do not retry blindly.` |
| `src/api/client.ts:191` | `Is the stand reachable? In local mode dnk-back must be running with API_TYPE=/api/admin.` | `Is the instance reachable at the configured base URL? It must expose the Admin API under /api/admin.` |
| `src/tools/docs.ts:54` | example docId `"back/docs/menus-module"` | `"mcp/docs/api/orders"` |
| `src/api/build-catalog.ts:277–278` | warning names `dnk-back`, `stand`, `npm run sync:permissions` | `…are absent from the instance's OpenAPI document (e.g. …). The bundled permission map was generated from a different platform revision than this instance runs.` |
| `src/api/catalog.ts:73, 89` · `src/tools/whoami.ts:30` · `src/bin/cli.ts:28` · `src/api/swagger-source.ts:24, 79, 99` | `stand` in warnings, hints, `--help` text and the `origin: 'stand' \| 'cache'` union | `instance` throughout |
| `README.md` (npm landing page) | `stand.example`, `--back ../dnk-back --front ../dnk-front`, "one stand", "the stand's own" | instance vocabulary; drop the `publish:knowledge` section entirely (see B4) |

**Also strip comments from the published build.** `dist/**.js` currently ships the full Russian JSDoc, including comments that name `dnk-back` (`src/config/config.ts`, `src/knowledge/loader.ts`, `src/api/build-catalog.ts:7`). Add `"removeComments": true` to `compilerOptions` in `tsconfig.build.json` — one line, removes the whole class of leak from the artifact. Build-time-only files (`build/**`, which name `dnk-back` freely) are not published, so they need no change.

## B3 — Seed and metadata

- **Seed alignment.** `knowledge/operating-rules.md` in the package is the offline fallback and will drift from `knowledge/mcp/operating-rules.md` in the rules repo. Establish the direction: **the rules repo is the source; copy it into the package at release time; never hand-edit the seed.** Add that line to the release checklist.
- **`src/server.ts:15`** — `SERVER_VERSION = '0.1.0'` while `package.json` says `0.1.1`, so the MCP handshake misreports the version. Read it from `package.json`, or at minimum keep the two in sync.
- **`package.json`** — `repository`, `bugs` and `homepage` all point at `gitlab.com/webdnk1/msvc/mcp-oneentry`, and those render on the public npm page. Point them at the public GitHub org or remove them. Separately, `"license": "UNLICENSED"` contradicts `publishConfig.access: "public"` and the README's `npm i -g` — pick one and make it deliberate.

## B4 — `publish:knowledge` must go

`build/publish-knowledge.ts` bulk-copies `.claude/docs`, `.claude/tests` and `.claude/rules` out of the internal checkouts into `--out ../oneentry-platform-rules`. Against the corpus in Part A it would, in one command: publish table names, source paths and issue ids; clobber curated files at the same paths; and mint new docIds through its 12 KB splitter. The corpus is hand-authored, so the script has no remaining purpose.

Delete `build/publish-knowledge.ts` and the `publish:knowledge` script. If a placeholder is wanted so nobody re-adds one, squat the name with a refusal:

```json
"publish:knowledge": "node -e \"console.error('REFUSED: the knowledge repository is hand-authored. Bulk-copying internal docs into it is prohibited.'); process.exit(1)\""
```

`sync:permissions` stays — it reads controller sources at build time and emits only a permission map, and its output is already public API surface.

## B5 — The published 0.1.0 tarball

`@oneentry/mcp-platform-server@0.1.0` is still downloadable and contains `data/knowledge.json` (2.07 MB): 143 internal documents with database identifiers, ~1265 `src/**` paths, materialized-view DDL, internal issue ids, branch names, service hostnames and real email addresses. 0.1.1 dropped it, but the old tarball is served indefinitely and may be mirrored.

- `npm deprecate "@oneentry/mcp-platform-server@0.1.0" "Superseded by 0.1.1 — do not use."` (immediate, always available).
- Unpublishing a single version is only self-service within 72 hours of publish; 0.1.0 went out 2026-08-06, so removal now requires an npm support request. Worth filing, given what the tarball contains.
- Rotate the admin credentials sitting in `dnk-mcp/.env` as hygiene — that file is on a machine that runs publish scripts.

## Verification (Part B)

```bash
cd /media/andrvm/disk_d/WebDevelop/dnk-mcp
npm run lint && npm test && npm run build
```

Then, before publishing, the artifact itself must be clean:

```bash
npm pack --dry-run                       # review the file list
grep -rniE 'dnk-|cms-sb|webdnk|gitlab\.com|\bstand\b|back/docs|front/docs|docLinks' dist/ data/ knowledge/ README.md package.json
grep -rnP '[\p{Cyrillic}]' dist/         # must be empty once removeComments is on
```

Then a behavioural check against the new corpus:

- `cms_api_describe` on any operation → no `docs` key; the non-read `next` points at `cms_docs_search`.
- Force a 400 → the hint says "Search the knowledge base with cms_docs_search before retrying", with no docIds in it.
- `cms_whoami` → reports the real package version at handshake.

---

# Rollout order

1. **Part A Phase 0** — push the rules repo. Servers pick it up within one cache TTL; `Knowledge is degraded` clears; `mcp/operating-rules` starts resolving for real.
2. **Part B B1 + B2**, then release. Doing B1 before Part A Phase 0 would leave `cms_api_describe` with no navigation hint *and* no `doc-map` to fall back on; doing Part A first means the corpus is already there when the hints disappear.
3. **B3–B5** with the same release, except B5, which is independent and should be done today.
4. **Part A Phases 1–2** continuously afterwards — each push reaches running servers within a TTL, no release required. That is the whole point of the design.
