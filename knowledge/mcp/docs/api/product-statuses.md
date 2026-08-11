# Product statuses

A product status says what can be done with a product — whether it is available, hidden, out of stock, or in whatever other state the instance defines.

Statuses are instance data. The markers below a customer uses are theirs, not the platform's, so read them rather than assuming.

→ `mcp/docs/api/products` · `mcp/docs/api/baseline-data`

## Statuses are resolved by marker

Each status has a marker, a stable string that a site and an API client use to identify it. The numeric id exists but is local to the instance.

Use the marker. A product payload that references a status by an id copied from elsewhere points at a different status or at nothing.

## Read the statuses first

```text
cms_api_search { "query": "product statuses", "mutating": false }
```

The listing gives you every status with its marker and its localized labels. Do this before writing a status onto a product, and before writing any code path that branches on one.

There is no platform-wide set of markers. `in_stock` is a plausible marker and also a guess — an instance may call the same idea something else entirely, or may not have it.

## Marker validation

There is an operation that checks whether a marker is free before you create a status with it. Use it rather than discovering the collision from a failed create, especially when a human has asked for a specific name.

## Create a status

Statuses are customer data, so creating one is legitimate. Before you do:

- confirm the concept is not already covered by an existing status under a different name;
- pick a marker that reads well and will not need renaming, because things reference it;
- provide localized labels for every active locale — the label is what a human sees in the admin panel and on a site.

Dry run, send, then read the list back and confirm the marker is what you intended.

## Setting a status on a product

The status is part of the product payload. Read a comparable product to see the exact field, then include it in your update.

Because a status change is what makes a product buyable or unbuyable, treat it as a content change with business meaning: say which products you are about to change and to what, before you do it in bulk.

## Statuses and site behaviour

A status affects what the Content API exposes and what a site shows. Two implications:

- Changing a status can make a product disappear from a customer-facing listing. That is the intended mechanism, and it is also how an accidental bulk change becomes visible to real users.
- Deleting a status that products still reference leaves those products pointing at nothing. Reassign first, then delete.

## Common mistakes

- **Guessing a marker.** Read the list.
- **Referencing a status by id.** Ids are local to the instance.
- **Deleting a status still in use.** Reassign the products first.
- **Creating a near-duplicate.** Two statuses meaning the same thing split the catalogue in ways that are tedious to unpick.

→ `mcp/docs/api/verification-recipes#products`
