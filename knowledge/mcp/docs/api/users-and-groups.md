# Users and user groups

Users are the customers of a site. Groups collect them and carry the permissions that decide what the Content API will serve.

This is where "the site returns 403" is usually resolved, and it is almost never solved by creating anything.

→ `mcp/docs/api/baseline-data#user-groups-and-the-guest-group` · `mcp/docs/api/admins-and-permissions`

## Users groups and admins are three things

- **Users** — customers. They sign in through the Content API, place orders, submit forms.
- **User groups** — collections of users carrying Content API permissions.
- **Admins** — the accounts that operate the admin panel and the Admin API. A different entity with a different permission model.

Confusing the last two is common and consequential: granting an admin permission does not open a Content API route, and granting a group permission does not let anyone into the admin panel.

## The guest group

A group with identifier `guest`, normally id 1, is provisioned with the instance. Every Content API permission is attached to it, and what an unauthenticated visitor may do is decided by its rules.

Its identifier is **not unique**, so creating a second group called `guest` succeeds and produces a group that nothing points at. Adjust the rules on the existing group; never create one.

A second group for signed-in customers, with identifier `user`, exists on fully provisioned instances and is absent on minimal ones. Check for it.

## Permissions are per route and already exist

Around 110 permission records are provisioned, one per Content API path, each with a section and a set of rules. They cover everything the Content API exposes.

So the workflow when a route is refused is:

1. Identify the exact path the site called.
2. Find the existing permission record for it.
3. Adjust its rules for the group in question.

A permission path is unique per group, so creating one that exists fails — and a create is never the right move here anyway.

## Read limits are a rule not a missing permission

A group's rules can allow reading with a restriction, which caps how many records a listing returns. A site that shows exactly the same small number of items for every listing is usually hitting that, not running out of content.

Check the restriction before investigating anything else. Removing it is a rules change on the existing permission.

## Diagnosing a Content API refusal

| Symptom | Usual cause |
|---|---|
| A permission error naming the route | The route is not granted to the group |
| Every listing returns the same small count | A read restriction on the group |
| Works signed in, fails signed out | The rule is on the signed-in group, not on `guest` |
| Works for one language only | Not permissions — content exists in one locale |

## Working with users

User records carry attribute values from a user attribute set, like any other entity — two levels, locale-keyed. That is what audience rules on blocks compare against.

Treat user data as personal data. Do not copy it into conversations, reports or examples; when you need to show that a user exists, show an identifier rather than their details.

→ `mcp/docs/api/block-types#audience-filtering`

## Create a group

Legitimate, and worth doing deliberately. A new group starts with no permissions, so nothing works for its members until you grant the routes they need — one existing permission record at a time.

Before creating one, check whether an existing group already expresses the distinction you need. Groups multiply easily and are tedious to consolidate.

## Common mistakes

- **Creating a permission that already exists.** Adjust the existing record.
- **Creating a second `guest` group.** Succeeds silently, achieves nothing.
- **Granting an admin permission to fix a Content API 403.** Different model.
- **Missing a read restriction** and investigating the data instead.
- **Putting user details into a report.** Personal data stays in the instance.

→ `mcp/docs/api/verification-recipes#permissions`
