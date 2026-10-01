# Reading the Content API from a site

The Admin API is what this server calls. The Content API is what the site you are building calls, and it is where you verify that a migration actually reached the public.

Its rules are not the admin ones: a different header carries the token, some entities have no public address at all, and a read taken immediately after a write can still show the previous value.

→ `mcp/docs/api/users-and-groups` · `mcp/docs/api/index-attributes`

## Public reads use the x-app-token header

The token an application holds goes in `x-app-token`. Sent as `Authorization: Bearer` it is not seen at all, and every route answers `401`:

```text
GET /api/content/pages/root?langCode=en_US
  Authorization: Bearer <app token>   → 401 {"message":"Invalid or missing app token"}
  x-app-token: <app token>            → 200 [ … ]
```

The same `401` is what a request with no token, an empty header, or a token this instance does not hold gets. A token whose lifetime has run out answers `401 App token is expired`, which is the one case the message names outright.

The status is the split worth remembering: `401` is always the token, `403` is always the rules or a closed resource. A `403` is never worth chasing with a new token, and a `401` is never worth chasing with a permission change.

## A refusal that no permission change will fix

`401` on **every** route, for a project whose guest group grants those routes, is the header — not the rules.

Work through it in this order, and stop at the first one that explains it:

1. The token is in `x-app-token`, not in an `Authorization` header.
2. The header is present and not empty.
3. The token belongs to this instance and its lifetime has not run out.

If every route still answers `403` after that, the token is being accepted and the question is a permissions one — which is where people start, and it is the wrong end.

→ `mcp/docs/api/users-and-groups#diagnosing-a-content-api-refusal`

## Why a public read still shows the previous value

The public projection catches up with an admin write a few seconds later — the same lag lists and filters have, and it applies to reading one entity by its address too.

So verify with a **paused read**, and never repeat the write.

→ `mcp/docs/api/index-attributes#when-a-written-value-becomes-searchable`

## What a 503 on a public read is telling you

A public read answers `503` with a JSON body and a `Retry-After` header when too many
requests are in flight for the instance at that moment:

```text
GET /api/content/products/103?langCode=en_US
  x-app-token: <app token>
  → 503 Retry-After: 1
    {"statusCode":503,"message":"Content API is temporarily overloaded, retry shortly"}
```

Nothing is missing and nothing is deleted. Wait the number of seconds the header names and
send the same request again; it succeeds once the burst passes.

This is the answer a prerender or an export that fans out hundreds of parallel reads will
meet first. Bound your own concurrency and retry on `503` rather than treating it as an
absent entity — a build that maps `503` onto "not found" bakes a missing page into the
output and keeps it until the next deploy.

The health route keeps answering while reads are being shed, so a `503` on content plus a
healthy instance is the expected combination, not a contradiction.

## What a 429 on a public read is telling you

An instance can also limit how **often** public reads are accepted, which is a different
limit from the one above: that one counts reads in flight at the same moment, this one
counts reads over a short window. Over the ceiling a read answers:

```text
GET /api/content/products/103?langCode=en_US
  x-app-token: <app token>
  → 429 Retry-After: 10
    {"statusCode":429,"message":"Too many requests, retry after the indicated delay"}
```

The window is counted per application token, so every page a storefront renders from the
same token shares one ceiling. Reads arriving without that header are counted per client
address instead. The health route is never subject to the limit.

Wait the number of seconds `Retry-After` names and send the same read again. Do not widen
your fan-out to catch up afterwards — the next window starts clean and a burst walks
straight back into the ceiling. An exporter or a prerender should cap its own rate and
honour the header per read, not per run.

`429` is the answer that means "slow down". A `403` on the same read means the reading
group is not allowed that content, and no amount of waiting changes it — see the refusal
section above. Treating one as the other either hides a real permission problem or
abandons content that was there all along.

Not every instance applies a frequency limit, so a run that never sees `429` does not
prove your client handles it. Handle the header, do not probe for the ceiling.

## Why a public list stops at the same number

The rules of the reading group decide how much of a list the public sees, not just whether the call succeeds. Nearly every content route is provisioned as a restricted read, and where that is applied it trims the answer to a fixed count — ten unless the instance says otherwise — and marks it in no way at all.

So a list that comes back the same short length whatever you ask for is a rule, and `limit` and `offset` cannot page past it. Read the record for the path before you treat the missing records as missing content. Which routes are trimmed varies, so measure the one you care about rather than assuming either way.

→ `mcp/docs/api/content-api-permission-rules#a-restricted-read-caps-the-list-at-ten`

## A page of type external page has no public address

`GET /api/content/pages/url/<url>` answers `404 Page not found` for a page whose general type is `external_page`, however correctly the page is set up. Nothing is wrong with it: that type has no public page of its own.

It is served **inside a menu** instead, and the address it points at arrives in `localizeInfos.<locale>.htmlContent` of its menu item.

That makes it the way to put an outside address into a section of the navigation tree, and it is worth choosing deliberately: the page exists in the content tree, so it can be nested, reordered and localized like any other.

→ `mcp/docs/api/general-types#which-type-to-pick` · `mcp/docs/api/menus#reading-a-menu-from-a-site`

## Which read to verify a migration with

An admin read shows what is stored. A public read shows what the site receives, and those are two different projections — a value can be perfect in one and absent from the other.

So check the work through the public route the site itself will call, with the locale the site asks for, after a pause. A migration reported as done on the strength of admin reads alone is one the customer finds the holes in.

→ `mcp/docs/api/verification-recipes` · `mcp/docs/api/bulk-content-migration`

## Common mistakes

- **Sending the application token as a bearer.** Every route answers `401 Invalid or missing app token`.
- **Reading a token `401` as a closed project** and rewriting the group permissions.
- **Repeating a write because the public read still shows the old value.** Wait and read again.
- **Treating a trimmed list as the whole list.** The restricted read says nothing about being cut.
- **Reading a `503` as missing content.** Respect `Retry-After` and send the read again.
- **Reading a `429` as a permission refusal.** It is a rate ceiling: wait out `Retry-After`.
- **Looking for a public address for an external page.** It arrives in the menu.
- **Verifying only through the admin read.** It is not what the site receives.

→ `mcp/docs/api/content-api-sign-in-and-cart` · `mcp/docs/server/doc-map`
