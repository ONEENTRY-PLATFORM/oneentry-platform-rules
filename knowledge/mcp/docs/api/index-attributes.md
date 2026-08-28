# Index attributes and when a value becomes searchable

An attribute has to be marked as indexed before it can be filtered on, sorted by, or found through search. Everything in this document follows from that.

The second half answers the question that brings most people here: why a value you just wrote is not in a list yet.

→ `mcp/docs/api/filters` · `mcp/docs/api/attribute-sets`

## What an index attribute is

An ordinary attribute holds a value on an entity. An **indexed** attribute additionally participates in querying: it can appear in a filter, in a facet, in a sort, and in search results.

Marking an attribute as indexed is a decision about the content model, not an optimisation. Attributes nobody queries do not need it; attributes a site filters on do.

## Declaring an attribute as indexed

The setting lives with the attribute in its attribute set. Find the operations with:

```text
cms_api_search { "query": "index attributes" }
```

When an attribute becomes indexed, existing values have to be picked up before they are queryable. That is not instantaneous on a catalogue of any size — expect a delay between the change and the data being fully usable, and verify with a query rather than assuming.

## When a written value becomes searchable

The line is between the **admin side and the public side**, not between one entity and a list.

An admin read reflects your write immediately — by id, and in an admin listing once the listing catches up, a few seconds later.

Every public Content API read lags by those same seconds, and that includes reading the one entity you just wrote by its address. So the read a site makes is not a way to confirm a write that has just gone in.

```text
write a page            → 200
admin read by id        → the new value, immediately
public read by url      → the previous value, for a second or three
public read by url      → the new value
```

Both answers are correct. They are two projections, and one of them catches up.

Check with a **paused read**, and do not repeat the write — a second write is never what a lagging read needs.

## A new attribute key is absent until it holds a value

An attribute you have just added to a set does not appear in the public answer at all until some entity has a value written into it. The key is missing rather than empty, and waiting does not bring it out.

That reads exactly like "the API does not have this field", and the conclusion people draw from it — that the write has to be repeated, or that the platform needs working around — is wrong both times. Write a value, wait, read again.

→ `mcp/docs/api/content-api-reads#why-a-public-read-still-shows-the-previous-value`

## Why a value you just wrote is missing from a list

In order of likelihood:

1. **Timing.** You looked too soon. Re-read by id on the admin side to confirm the write landed, then look again.
2. **The attribute is not indexed.** A filter on it matches nothing, silently.
3. **The attribute map was one level deep.** The write was accepted and stored nothing. Reading by id shows the attribute empty — this is the check that distinguishes it from timing.
4. **Wrong locale.** Values are locale-keyed; you may be reading a language you did not write.
5. **The response was truncated.** Your item may be beyond the cap. Check for a `_truncated` envelope.

## What not to conclude and what not to do

Do **not** repeat the write. A missing entry in a list is not evidence that the write failed, and a repeated create produces a duplicate that consumes the instance's record quota and has to be cleaned up by hand.

Do not conclude the field does not exist. The admin read by id is the authoritative and immediate one; a public read that disagrees with it is behind, not right.

Do not add retry loops that re-send writes. Retry the **read**, if anything.

→ `mcp/operating-rules#a-read-straight-after-a-write-can-lag`

## Reading indexed values back

There is an operation returning the distinct values an indexed attribute currently holds across the catalogue. Two good uses:

- checking a filter value exists before sending a filter that would otherwise return nothing;
- seeing what a facet on a site will actually offer.

## Why a search by meaning finds nothing for a kind

Global search and the per-kind `POST /{kind}/vector/search` routes match by meaning, and a kind whose records carry no vector answers them with nothing at all. An empty answer reads exactly like "no match", so check the coverage before concluding either.

`IndexAttributeController_getHealth` — `GET /index-attributes/health` — reports it per kind under `sinks.vectors.byTable`, with an entry each for pages, discounts, users, admins and orders:

```json
{ "sinks": { "vectors": { "byTable": {
    "pages": { "total": 14, "eligible": 14, "vectorized": 0,
               "missingSample": [1, 38, 39] } } } } }
```

`total` counts the records. `eligible` counts those carrying enough text to produce a vector — a record with no title and no usable attribute cannot. `vectorized` counts those that have one, and `missingSample` lists up to fifty ids that ought to and do not.

`eligible` above `vectorized`, or a non-empty `missingSample`, means the coverage is behind. Say so, name the kind, and stop there: rewriting the entities does not provoke it, and a second search does not either.

The `vectorized` and `total` sitting beside `byTable` count products alone. Read the per-kind entries instead — the two outer numbers can look complete while a whole kind is uncovered.

→ `mcp/docs/api/global-search`

## Bulk changes take proportionally longer

Importing or updating many entities at once means the queryable side lags by more than seconds. Verify a bulk change by sampling — read a handful of entities by id, then check the listing once, rather than polling the listing repeatedly.

→ `mcp/docs/api/import`

## Common mistakes

- **Filtering on a non-indexed attribute** and reading the empty result as "no matches".
- **Re-writing after a list looks stale.** Read by id instead.
- **Re-writing after a public read looks stale.** It lags the admin read; wait and read again.
- **Reading a new attribute key as unsupported** because the public answer does not carry it. Write a value into it first.
- **Verifying a bulk import by refreshing a listing.** Sample by id.
- **Assuming indexing is instant after marking an attribute.** Existing values are picked up progressively.
- **Reading an empty search by meaning as no match.** Check the coverage for that kind first.

→ `mcp/docs/api/verification-recipes`
