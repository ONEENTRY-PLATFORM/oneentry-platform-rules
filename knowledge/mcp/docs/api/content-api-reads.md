# Reading the Content API from a site

The Admin API is what this server calls. The Content API is what the site you are building calls, and it is where you verify that a migration actually reached the public.

Its rules are not the admin ones: a different header carries the token, some entities have no public address at all, and a read taken immediately after a write can still show the previous value.

→ `mcp/docs/api/users-and-groups` · `mcp/docs/api/index-attributes`

## Public reads use the x-app-token header

The token an application holds goes in `x-app-token`. Sent as `Authorization: Bearer` it is not seen at all, and every route answers `403`:

```text
GET /api/content/pages/root?langCode=en_US
  Authorization: Bearer <app token>   → 403 {"message":"Resource is closed"}
  x-app-token: <app token>            → 200 [ … ]
```

The same `403` is what a request with no token gets, which is why the header is worth checking first: the two are indistinguishable from the response.

## A 403 that no permission change will fix

`Resource is closed` on **every** route, for a project whose guest group grants those routes, is the header — not the rules.

Work through it in this order, and stop at the first one that explains it:

1. The token is in `x-app-token`, not in an `Authorization` header.
2. The token belongs to this instance.
3. The guest group grants the route, and the read restriction on it is not what you are seeing.

Only the third is a permissions question, and it is the one people start with.

→ `mcp/docs/api/users-and-groups#diagnosing-a-content-api-refusal`

## Why a public read still shows the previous value

The public projection catches up with an admin write a few seconds later — the same lag lists and filters have, and it applies to reading one entity by its address too.

So verify with a **paused read**, and never repeat the write.

→ `mcp/docs/api/index-attributes#when-a-written-value-becomes-searchable`

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

- **Sending the application token as a bearer.** Every route answers `403 Resource is closed`.
- **Reading that `403` as a closed project** and rewriting the group permissions.
- **Repeating a write because the public read still shows the old value.** Wait and read again.
- **Looking for a public address for an external page.** It arrives in the menu.
- **Verifying only through the admin read.** It is not what the site receives.

→ `mcp/docs/api/content-api-sign-in-and-cart` · `mcp/docs/server/doc-map`
