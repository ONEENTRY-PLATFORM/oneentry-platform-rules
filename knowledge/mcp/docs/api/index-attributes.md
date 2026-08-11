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

Reading an entity **by id** reflects your write immediately.

Lists, filters and search reflect it a short time later — usually seconds.

```text
write a product        → 201, id 512
list products          → 512 is not there yet
get product 512        → there it is, with the new value
list products again    → now it is there
```

Both readings are correct. They are answering from different places, and one of them catches up.

## Why a value you just wrote is missing from a list

In order of likelihood:

1. **Timing.** You looked too soon. Re-read by id to confirm the write landed, then look again.
2. **The attribute is not indexed.** A filter on it matches nothing, silently.
3. **The attribute map was one level deep.** The write was accepted and stored nothing. Reading by id shows the attribute empty — this is the check that distinguishes it from timing.
4. **Wrong locale.** Values are locale-keyed; you may be reading a language you did not write.
5. **The response was truncated.** Your item may be beyond the cap. Check for a `_truncated` envelope.

## What not to conclude and what not to do

Do **not** repeat the write. A missing entry in a list is not evidence that the write failed, and a repeated create produces a duplicate that consumes the instance's record quota and has to be cleaned up by hand.

Do not conclude the API is broken. Read by id — that answer is authoritative and immediate.

Do not add retry loops that re-send writes. Retry the **read**, if anything.

→ `mcp/operating-rules#a-read-straight-after-a-write-can-lag`

## Reading indexed values back

There is an operation returning the distinct values an indexed attribute currently holds across the catalogue. Two good uses:

- checking a filter value exists before sending a filter that would otherwise return nothing;
- seeing what a facet on a site will actually offer.

## Bulk changes take proportionally longer

Importing or updating many entities at once means the queryable side lags by more than seconds. Verify a bulk change by sampling — read a handful of entities by id, then check the listing once, rather than polling the listing repeatedly.

→ `mcp/docs/api/import`

## Common mistakes

- **Filtering on a non-indexed attribute** and reading the empty result as "no matches".
- **Re-writing after a list looks stale.** Read by id instead.
- **Verifying a bulk import by refreshing a listing.** Sample by id.
- **Assuming indexing is instant after marking an attribute.** Existing values are picked up progressively.

→ `mcp/docs/api/verification-recipes`
