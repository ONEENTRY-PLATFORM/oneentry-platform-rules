# Where this documentation comes from

The corpus you are reading is not shipped inside the package. It is fetched from a public repository at runtime, cached by commit, and re-checked at most once an hour.

Read this to understand what `cms_whoami` is telling you about the documentation, or to run a server against a local copy while editing it.

→ `mcp/docs/server/authoring-knowledge` · `mcp/docs/server/cms-docs-search-and-read`

## Where this knowledge comes from

By default: the public repository `ONEENTRY-PLATFORM/oneentry-platform-rules` at ref `main`. Both are configurable — `--knowledge-repo` and `--knowledge-ref` — and the ref may be a branch, a tag or a commit, so an instance can be pinned to a known corpus.

Editing a document there takes effect on the next restart of any server, with no package release. That is the whole point of the design.

## How a document becomes searchable

1. Only files matching `knowledge/**/*.md` are taken from the repository.
2. Each file's path under `knowledge/`, minus `.md`, becomes its `docId`.
3. Each file is split into sections at its `##` headings. Text before the first heading becomes the preamble, addressed by an empty anchor.
4. Each section is indexed on its heading, the document title, the docId and its body — headings weighted most heavily.

So a document's path and its headings, not its prose, determine whether it can be found.

## docId anchors and sections

`cms_docs_read { docId }` with no anchor returns the preamble plus the list of sections and their sizes. With an anchor it returns that one section.

Anchors are slugs of the heading text: lowercase, punctuation removed, spaces replaced by hyphens. A section over 12 KB is truncated, and documents in this corpus are written to stay well under that.

## The cache and its lifetime

The repository is downloaded once as an archive of a specific commit and unpacked into the cache directory — by default under `~/.cache/oneentry-mcp-platform`, overridable with `--cache-dir`. Only one commit is kept; older ones are pruned.

| Situation | What the server does |
|---|---|
| Cache younger than the refresh interval | No network at all |
| Cache older, commit unchanged | One small request, no download |
| Commit moved | One small request plus one archive download |
| Repository unreachable, cache present | Uses the cache and reports `source: cache` |
| Repository unreachable, no cache | Falls back to the bundled rules and says so |

The refresh interval is one hour by default (`--knowledge-ttl`). A `GITHUB` token is optional and only raises the API rate limit; the repository is public.

## Running offline or from a local path

`--offline` never touches the network: the cache is used if present, otherwise the bundled rules. Useful in a sealed environment, and honest about what it is giving you.

`--knowledge-path <dir>` reads a local clone instead. Either the clone root or its `knowledge/` directory works.

```bash
oneentry-mcp-platform --knowledge-path ../oneentry-platform-rules
```

While it is set the network is never contacted and the cache is not consulted, so edits take effect on the next restart with no commit and no push. `cms_whoami` reports `source: local`.

A path that cannot be read is a **startup error**, not a fallback — a bad path is a configuration mistake, and degrading silently to the bundled rules would look like an empty corpus.

## The bundled seed and the degraded warning

The package carries one document as a last resort: the operating rules. When the repository cannot be reached and no cache exists, that is all you get, and both `cms_guide` and `cms_whoami` say so:

```text
> **Knowledge is degraded.** Knowledge repository … is unavailable … Only the bundled
operating rules are available — module reference docs will not be found.
```

In that state a search returning nothing means nothing is loaded, not that nothing exists. Say so to the human instead of concluding the documentation has no answer.

## What the search cannot do

- **It does not translate.** The corpus is English; a question in another language will not reach it.
- **It does not reason.** It matches terms, weighted by where they appear. Phrase a query the way a heading is written.
- **It ignores very short words.** Tokens of two characters or fewer are dropped, so an unexpanded acronym can silently carry no signal.
- **It has no notion of recency.** Every document is equally current, which is why a stale document is worse here than a missing one.

## Reporting a gap in the documentation

If the answer was not here and you had to work it out, tell the human in one line: what you were doing, what you searched for, and what turned out to be true. Documents in this corpus are added from exactly those reports.
