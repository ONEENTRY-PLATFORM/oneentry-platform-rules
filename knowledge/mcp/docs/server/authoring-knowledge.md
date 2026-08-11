# Writing documents for this corpus

The rules for adding to or changing the documentation you are reading. If someone asks you to fix, extend or correct these documents, follow this before you write anything.

The corpus lives in the public repository the server fetches at startup. Every document is hand-authored for a public audience; nothing is imported from anywhere.

→ `mcp/docs/server/knowledge-subsystem` · `mcp/docs/server/doc-map`

## Where a new document goes

Two places, and no others:

- `knowledge/mcp/docs/server/<name>.md` — the MCP server itself: configuration, tools, safety, errors.
- `knowledge/mcp/docs/api/<name>.md` — operating the Admin API: one document per area.

`knowledge/mcp/operating-rules.md` is fixed and must keep exactly that path. Its docId is referenced by the server itself.

Filenames are lowercase, hyphenated, `.md`. Never `index.md` — it collapses onto the parent name and can shadow a sibling. Never `README.md` under `knowledge/`. Nothing that is not markdown.

Before creating a document, check whether the fact already has a home. Facts belong in exactly one place and are cited from everywhere else.

## Size limits that are not negotiable

- File: 10 KB.
- Section: 8 KB. The server truncates at 12 KB, and the margin is for safety.
- `mcp/operating-rules`: 7.5 KB total, because it is concatenated whole into a resource.

These are **bytes**. If a document outgrows the limit, split it by subject into a second document and cross-link — do not split it mid-topic.

## Headings decide whether anything is found

A heading is weighted four times more than body text, so it is the retrieval unit.

- Write it as the question someone would ask: `## Why a value you just wrote is missing from a list`, not `## Consistency`.
- Letters, digits, spaces and hyphens only. Other punctuation survives into the anchor and makes it untypable.
- 60 characters or fewer.
- Unique within the document after slugging. Two headings that slug the same silently produce an unusable `-2` anchor.
- Never numbered. Numbering makes anchors positional, so inserting a section renames every one after it.

Use `##` only. `###` also splits a section and leaves a near-empty parent; `####` and deeper are safe for inner structure.

## Every document needs a preamble

The text between the `# H1` and the first `##` is what `cms_docs_read { docId }` returns when no anchor is given. A document that opens straight into a heading has no such section, and the natural read fails.

Two to six lines: what this covers, when to read it, then two to four pointers.

## Linking between documents

One form only:

```text
→ `mcp/docs/api/orders#create-an-order`
```

That is the same identity an agent pastes into `cms_docs_read`. Relative markdown links invite a filesystem read that will fail; repository URLs are worse.

Put pointers in the preamble and in one final section. Do not repeat a link block across documents: the search combines terms with OR, so shared boilerplate makes every document a weak match for every query.

## Language policy

English, throughout. The tool descriptions, refusal messages, operation ids and API summaries an agent is holding are all English, and the search does not translate — a document in another language cannot be reached from the text that would send someone looking for it.

Material that exists only in another language is translated, never pasted.

## What must never be published

This repository is public and permanent. A revert does not unpublish anything.

Apply three filters to every sentence: could someone with only an admin login, the instance's API document and time to experiment have written it; does it describe what to observe and do rather than why the system works that way; and would an agent take a wrong action without it. All three must pass.

Never include storage identifiers or schema, the technologies the platform is built on, source paths or type names, provisioning and deployment detail, internal hostnames, tracker ids, branch names, internal repository names, personal data, or a raw error message that embeds any of those. Describe a failure as status, trigger and fix instead.

Operation ids, URL paths, field names, enum values, permission strings and status codes **are** publishable — an agent cannot work without them.

## Before you commit

- One `# H1` on the first line; a non-empty preamble follows it.
- No `###`; headings unique, unnumbered, 60 characters or fewer, plain characters only.
- File under 10 KB; every section under 8 KB.
- Every code fence tagged and balanced. An unbalanced fence collapses the rest of the file into one truncated section — the most destructive formatting error possible here.
- Cross-links use `docId#anchor` and resolve.
- `mcp/docs/server/doc-map` updated if a document was added.
- Examples verified against a real instance; placeholder data only.
- Nothing from the denylist above.

Then verify it the way an agent will see it:

```bash
npx -y @oneentry/mcp-platform-server --knowledge-path .
```

and search for the document you just wrote.
