# OneEntry Platform Rules

Documentation that teaches an AI agent to operate the **OneEntry Admin API** correctly and safely: the operating rules, a full reference for the MCP server, and a guide per API area.

This is the content source for [`@oneentry/mcp-platform-server`](https://www.npmjs.com/package/@oneentry/mcp-platform-server) — the server ships no content of its own, it fetches these files at runtime.

> **Public documentation, hand-authored. Do not import from internal repositories.**

## How the server reads this repository

On startup the server resolves the configured ref (`main` by default), downloads the repository archive once, caches it by commit, and indexes the markdown. Editing a file here — through the GitHub UI, a pull request, whatever — takes effect on the next restart of any server, with no npm release.

Five mechanics decide whether a document is usable:d

1. **Only `knowledge/**/*.md` is indexed.** This `README.md` and `CLAUDE.md` are for humans and agents working on *this repository*; the server never sees them.
2. **`docId` = the path under `knowledge/` without `.md`.** `knowledge/mcp/docs/api/orders.md` → `mcp/docs/api/orders`. A `<name>/index.md` collapses to `<name>`, which is why this repo never uses that filename.
3. **Documents are split into sections at `##`.** The text before the first `##` is the *preamble*, and it is what an agent gets when it reads a document without an anchor. Every file must have one.
4. **A section over 12 KB is truncated.** Keep files under 10 KB and sections under 8 KB — bytes, not characters.
5. **Headings are the retrieval unit.** They carry four times the weight of body text, so a heading should read like the question someone would ask.

The full authoring contract is in [CLAUDE.md](CLAUDE.md), and a runtime copy that agents can read through the server itself lives at `knowledge/mcp/docs/server/authoring-knowledge.md`.

## Layout

```
knowledge/
  mcp/operating-rules.md        -> mcp/operating-rules      the must-read before any write
  mcp/docs/server/*.md          -> mcp/docs/server/<name>   the MCP server: install, config, tools, safety
  mcp/docs/api/*.md             -> mcp/docs/api/<name>       operating the Admin API through it
```

`mcp/operating-rules` is addressed directly by the server — it is exposed as an MCP resource and named in the server's own instructions — so its `docId` must never change.

## What belongs here

Everything in this repository describes **observable behaviour of a public API** and the policy of a published client. Write from the API outward: make the call, read the response, document what an operator sees and what they should do.

Nothing here describes how the platform is built. No database identifiers, no schema, no infrastructure, no source paths, no internal repository or tracker references. If a sentence could only have been written by someone with access to OneEntry's source or database, it does not belong in this repository — and no amount of editing makes a copied internal document publishable.

## Contributing

1. Branch from `main`.
2. Read [CLAUDE.md](CLAUDE.md) before your first change — the mechanical rules above are not style preferences, and breaking one silently breaks retrieval for every running server.
3. Verify examples against a real instance before committing them.
4. Open a pull request. `main` is what every server reads, so changes land there through review, not directly.

To preview your changes exactly as an agent will see them, point a server at your working copy:

```bash
npx -y @oneentry/mcp-platform-server --knowledge-path .
```

Then call `cms_whoami` (it should report `source: "local"`), and search for the document you just wrote.
