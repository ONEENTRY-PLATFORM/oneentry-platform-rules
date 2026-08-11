# About this repository

`oneentry-platform-rules` — the public documentation corpus for [`@oneentry/mcp-platform-server`](https://www.npmjs.com/package/@oneentry/mcp-platform-server).

This file is the contract for anyone — human or agent — editing the corpus. It is **not** served by the server; the runtime-visible subset lives at `knowledge/mcp/docs/server/authoring-knowledge.md`, and the two must stay in agreement.

## What you are editing

The server ships no documentation. At startup it downloads this repository at ref `main` and indexes `knowledge/**/*.md`. Every agent connected to every instance reads what is on `main`.

Two consequences, both load-bearing:

- **`main` is production.** There is no staging. A change reaches running servers when their cache expires, within an hour by default. A broken document is a broken product.
- **This repository is public and permanent.** A revert does not unpublish anything — archives are cached and mirrored. Publishing internal detail here is not an incident you can fix by deleting a line.

## Start of each session — mandatory checklist

### Before writing any documentation

1. Read this file **in full**.
2. `ls knowledge/mcp/docs/server/ knowledge/mcp/docs/api/` — see what already exists. Facts live in exactly one document; if the fact you are about to write already has a home, edit that home.
3. Read `knowledge/mcp/docs/server/doc-map.md` — it is the index agents navigate by, and it must be updated in the same commit as any new document.
4. If you are documenting API behaviour, confirm it against a real instance first. Do not document from memory, from an internal document, or from the OpenAPI schema alone.

### Before writing anything sourced from outside this repository

Stop and read "Never publish" below. Then rewrite from scratch — never paste and redact.

## Mechanics you must internalise

These come from the server's indexer. Violating one does not produce an error; it produces a document that cannot be found or is silently cut in half.

| Mechanic | Consequence for you |
|---|---|
| Only `knowledge/**/*.md` is indexed | Anything an agent needs at runtime lives under `knowledge/`. Root files are invisible to the server. |
| `docId` = path under `knowledge/` minus `.md` | The path *is* the public identifier. Renaming a file breaks every link and every agent that memorised it. |
| `<name>/index.md` collapses to `<name>` | Never create `index.md` — it can collide with a sibling `<name>.md` and the loser vanishes. |
| Sections split at `##` **and `###`** | Use `##` only. `####` and deeper are safe for inner structure. |
| Text before the first `##` is the empty-anchor section | A file that opens straight into a heading cannot be read without an anchor. Every file needs a preamble. |
| Anchors are slugged from heading text | Lowercase, punctuation stripped, spaces to hyphens, cut at 80 characters. Em dashes, slashes and emoji survive into the anchor and make it untypable. |
| Duplicate slugs get `-2`, `-3` | Two headings that slug the same silently produce an unusable second anchor. |
| A section over 12288 **bytes** is truncated | Work to 10 KB per file and 8 KB per section. Bytes, not characters. |
| Search boosts: heading ×4, docId ×3, doc title ×2, body ×1 | Headings and file names carry the retrieval. Write them as the question someone would ask. |
| Search combines terms with OR, with prefix and fuzzy matching | Boilerplate repeated across documents makes every document a weak match for every query. |
| Query tokens of two characters or fewer are dropped | Never make a short acronym load-bearing; expand it at least once per document. |
| Search does not translate | English only. |

## Hard rules

### Structure

```markdown
# One H1, on the first non-blank line, carrying the subject noun

Two to six lines of preamble. What this document covers, when to read it.
Ends with two to four pointers.

→ `mcp/docs/server/payload-conventions#localizeinfos-is-keyed-by-locale-code`

## A heading phrased as the question someone would ask
```

- ✅ One `# H1`. It becomes the document title and is boosted on every section.
- ❌ No YAML frontmatter. It is not parsed; it lands verbatim in the preamble and is shown to the model as content.
- ❌ No `###`. It splits a section and leaves a near-empty parent.
- ✅ Headings: letters, digits, spaces and hyphens only, 60 characters or fewer, unique after slugging.
- ❌ No numbered headings. Numbering makes anchors positional, so every insertion is a breaking change.
- ✅ Every code fence tagged and balanced. An unbalanced fence collapses the rest of the file into one truncated section — this is the single most destructive formatting error possible here.

### Headings decide whether anything is found

- ✅ `## Refused because of the allow level`
- ❌ `## Refusals`
- ✅ `## Why a value you just wrote is missing from a list`
- ❌ `## Consistency`

### Linking

- ✅ `` → `mcp/docs/api/orders#create-an-order` `` — the same identity an agent can paste straight into `cms_docs_read`.
- ❌ `[orders](../api/orders.md)` — invites a filesystem read that will fail.
- ❌ A link to an anchor ending in `-2`. Fix the duplicate heading instead.
- ✅ One pointer section per document, in its final section, with two to five links.
- ❌ A "see also" block repeated across documents.

### Size

- File ≤ 10 KB. Section ≤ 8 KB. Sweet spot: 300–1500 bytes per section, 8–15 sections per document.
- `knowledge/mcp/operating-rules.md` ≤ 7.5 KB total, ≤ 700 bytes per section. It is concatenated whole into an MCP resource that clients pin into context.

### Examples

- Verified against a real instance before they are committed.
- Placeholder data only: `https://your-instance.example`, ids of three digits or fewer, no real markers from a customer project.
- 25 lines or fewer.

### One fact, one home

| Kind of fact | Home |
|---|---|
| How the MCP server behaves | `mcp/docs/server/…` |
| How an Admin API area behaves | `mcp/docs/api/…` |
| Payload law that applies everywhere | `mcp/docs/server/payload-conventions` |
| The short rules an agent must know before its first write | `mcp/operating-rules` |

Cite; never restate. Duplication is not redundancy here — it is two documents that will disagree in six months, both of which the search will return.

## 🚨 Never publish

Everything here describes **observable behaviour of a public API** and the policy of a published client. Apply three filters to every sentence, in order:

1. **Black box.** Could a competent stranger who has an admin login to one instance, that instance's OpenAPI document, and time to experiment have written this? If it took access to the platform's source, database, cluster, provisioning scripts or issue tracker — cut it.
2. **Causeless.** Publish what the agent observes and what it must do. Never why the system behaves that way.
3. **Necessity.** Even if it passes both, cut it unless an agent would take a wrong action without it. This is an operating manual, not a reference work.

### The denylist

| Category | Examples of what to remove |
|---|---|
| Storage | Table, column, view, index, constraint or enum-type names; any DDL or query text; anything about how data is persisted |
| Named components | Queues, brokers, search engines, caches, vector stores, background workers |
| Technology stack | The frameworks, ORM, database engine or search engine the platform is built on |
| Source coordinates | File paths, line anchors, class / decorator / DTO / enum type names, module directory layout |
| Provisioning | Migration or seed script names and timestamps, deploy procedures, cluster tooling, service accounts |
| Infrastructure | Internal hostnames, ports, container names, environment variables of the platform itself |
| Process artifacts | Issue-tracker ids, branch names, merge-request text, dated internal specs, author names, personal email addresses |
| Internal names | Engineering code names for the platform repositories, and the word used internally for a deployment (say **instance**) |
| Defect framing | "bug", "broken", "known defect", root causes, what will be fixed when |
| Raw error strings | Any server message that embeds one of the above. Describe the failure as status + trigger + fix instead |
| Scale counts | Totals that imply internal size. They are instance-dependent and go stale anyway |

### Rewrite examples

- ❌ "A write reaches the index through the background queue, so the search engine lags the tables by a few seconds."
- ✅ "A value you just wrote may not appear in list or search responses for a few seconds. Re-read the entity by id to confirm. Never repeat the write — you will create a duplicate."

- ❌ "This 500 comes from a not-null violation on the processing type column."
- ✅ "This call answers 5xx when the body is not wrapped as the operation expects. Send it wrapped and the call succeeds."

- ❌ "The guest group is inserted by a migration at first deploy."
- ✅ "A user group with identifier `guest` is provisioned with the instance."

### Deliberate exceptions

These *are* publishable, because an agent physically cannot operate without them and any admin can see them:

- Operation ids exactly as the catalog emits them — the `Controller_method` form, e.g. `AdminProductsController_findAll`. A bare controller name with no `_method` suffix is a source reference and is **not** allowed.
- URL paths, parameter and field names, enum values, permission strings, status codes.
- Wire-format conventions: locale-keyed content, two-level attribute values, lexorank positions, marker versus id.
- This server's own client policy: allow levels, confirm gating, truncation, dry run.

### No bulk import

Every file under `knowledge/` is hand-authored for a public audience. There is no import script and one must not be added. Copying documentation out of an engineering checkout into this repository — in whole or in part, with or without edits — is prohibited. Those documents are written for readers who have the source open; they cite paths, storage identifiers and tracker ids on nearly every page, and a file that started as a copy cannot be made publishable by deleting lines from it.

Cyrillic anywhere under `knowledge/` is near-proof of a paste rather than authorship. The corpus is English-only; translate, do not carry across.

## docIds are a public API

A `docId` is referenced by agents, by cross-links in this corpus, and — for `mcp/operating-rules` — by the server binary itself. There is no redirect mechanism.

- Renaming or deleting a document is a breaking change. Do not do it as part of another change.
- `mcp/operating-rules` is immovable: its docId appears in an MCP resource URI, in the server's instruction string, in the guide text, and in the bundled offline copy.
- Filenames: lowercase ASCII, kebab-case, `.md`. No `index.md`, no `README.md` under `knowledge/`, no non-markdown files.

## When to stop and ask

- You are about to rename or delete a `docId`.
- You are about to change a section title in `mcp/operating-rules` (it breaks every cross-link and every pinned reference).
- You cannot verify a behaviour against a real instance and would have to describe it from memory.
- A fact seems necessary but cannot be written without naming something on the denylist. That is a signal the instruction itself is wrong for a public audience — ask, do not soften.
- Content originates from an internal checkout, an internal document, or an internal conversation.

## Before you commit

- [ ] One `# H1`; non-empty preamble before the first `##`.
- [ ] No `###`; headings ≤ 60 chars, letters/digits/spaces/hyphens, unique, unnumbered.
- [ ] File < 10 KB; every section < 8 KB.
- [ ] Fences tagged and balanced.
- [ ] Cross-links use `docId#anchor` and resolve.
- [ ] `doc-map.md` updated if a document was added.
- [ ] Examples verified against a real instance; placeholder data only.
- [ ] Nothing on the denylist; no Cyrillic; English throughout.
- [ ] Nothing in this diff was copied from an internal source.
