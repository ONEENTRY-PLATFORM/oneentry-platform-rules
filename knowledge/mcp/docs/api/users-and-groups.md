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

A permission record is provisioned for every Content API path, each with a section and a set of rules. They cover everything the Content API exposes.

So the workflow when a route is refused is:

1. Identify the exact path the site called.
2. Find the existing permission record for it.
3. Adjust its rules for the group in question.

A permission path is unique per group, so creating one that exists fails — and a create is never the right move here anyway.

## Read limits are a rule not a missing permission

A site showing the same small number of items for every listing is hitting a read restriction on the group, not running out of content — and that restriction is how every content route is provisioned. Check it before investigating anything else; lifting it is a rules change on the existing record, never a new one.

→ `mcp/docs/api/content-api-permission-rules#a-restricted-read-caps-the-list-at-ten`

## Diagnosing a Content API refusal

| Symptom | Usual cause |
|---|---|
| **Every** route answers `403 Resource is closed` | The application token was sent as a bearer instead of in `x-app-token` |
| A permission error naming the route | The route is not granted to the group |
| Every listing returns the same small count | A read restriction on the group |
| Works signed in, fails signed out | The rule is on the signed-in group, not on `guest` |
| Works for one language only | Not permissions — content exists in one locale |
| A listing read with `POST` is refused | The **add** rule opens it, not a read rule |

The refusal names the rule and the record to fix — `requires the "addRule" rule to be enabled on the permission (permissionId: 36) linked to the user group`. Which flag opens which method is in `mcp/docs/api/content-api-permission-rules#the-five-rules-and-what-each-one-opens`.

Rule out the first row before changing any rules: it looks exactly like a closed project, and the group's own rules will show the routes as granted the whole time.

→ `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Working with users

User records carry attribute values from a user attribute set, like any other entity — two levels, locale-keyed. That is what audience rules on blocks compare against.

Treat user data as personal data. Do not copy it into conversations, reports or examples; when you need to show that a user exists, show an identifier rather than their details.

→ `mcp/docs/api/block-types#audience-filtering`

## The isLogin field is what a user signs in with

One attribute of a sign-in form carries the flag isLogin. Its value is not an ordinary profile field: it is the credential the person signs in with. Writing it in the user card moves the credential too, so the displayed value and the working one never drift apart. You do not name a form when you do it — the form of the user's own sign-in provider is used.

A value counts as a change only when it differs from what the profile already holds under that marker. The card sends `formData` back whole on every save, so an edit that leaves the field alone is never refused, whatever state the account is in.

A real change answers `400` and writes nothing when:

- the value is blank;
- it is already taken by another user, **including removed ones**;
- the body carries the marker under several locales with different values;
- the account came from a social network, where the credential belongs to the network.

Nothing is written on refusal — neither the profile nor group membership changes, even when the same call also sends `groupIds`. Retry with a corrected value rather than splitting the call in two.

The value is stored exactly as sent. Alone among the form's fields it is not escaped, so `&` and `<` survive and the person signs in with the characters you wrote.

## Create a group

Legitimate, and worth doing deliberately. A new group starts with no permissions, so nothing works for its members until you grant the routes they need — one existing permission record at a time.

Before creating one, check whether an existing group already expresses the distinction you need. Groups multiply easily and are tedious to consolidate.

## Put a user into a group

Membership is written on the user, not on the group. There is no add-member call on a group; use `AdminUsersController_update` — `PUT /api/admin/users/{id}` — with `groupIds`.

The field is a **full replacement**, not an addition. Send every group the user should end up in, including the ones they already have.

- `groupIds: [1, 4]` — the user ends up in exactly groups 1 and 4.
- `groupIds: []` — every membership is removed.
- field omitted entirely — membership is left as it is.

`authProviderId` is required in the body and must be the provider the user actually signed up through; anything else answers 404. An id in `groupIds` that no group carries answers 400 and writes nothing at all, so a rejected call never leaves a half-applied set.

```json
{
  "authProviderId": 1,
  "langCode": "en_US",
  "isActive": true,
  "groupIds": [1, 4]
}
```

The same call also writes `formData` and `notificationData` from whatever the body carries, so omitting them replaces them with empty values. Read the user first and send those fields back unchanged unless you mean to change them.

## Add a sign-in provider for a social network

Every social provider is created with `type: "oauth"`. There is no per-network type value — `"apple"` or `"yandex"` as a `type` is rejected. The network is named in the settings instead, under `oauthProvider`.

```json
{
  "type": "oauth",
  "identifier": "apple-signin",
  "localizeInfos": { "en_US": { "title": "Sign in with Apple" } }
}
```

Create the provider first, then set its settings with the update call. Settings with no `oauthProvider` are treated as Google, so providers configured before the other networks existed keep working untouched.

`oauthAuthUrl` is required for every network, and it is the only setting a Content API reader can see. Secrets you store are never returned there.

## Which settings each social network needs

Ask the instance rather than guessing: `AdminUsersAuthProviderController_getOauthCatalog` — `GET /api/admin/users-auth-providers/oauth-catalog` — lists every supported network with the settings keys it requires.

```json
[
  {
    "key": "apple",
    "tokenUrl": "https://appleid.apple.com/auth/token",
    "requiredFields": [
      "oauthClientId", "oauthAuthUrl",
      "oauthTeamId", "oauthKeyId", "oauthPrivateKey"
    ],
    "optionalFields": ["oauthTokenUrl"]
  }
]
```

Most networks need `oauthClientId`, `oauthSecret` and `oauthAuthUrl`. Apple is the exception, and the catalog shows it: it takes key material — `oauthTeamId`, `oauthKeyId` and a private key — and asks for no `oauthSecret`, because the short-lived secret Apple requires is produced for each sign-in rather than stored. A hand-made value in `oauthSecret` is ignored for Apple.

`oauthTokenUrl` is optional everywhere and overrides the network's standard endpoint when set.

## Why a social sign-in answers 400

- **The network reported an address it has not verified.** The sign-in is refused so an unverified address cannot claim an existing account. Ask the person to confirm their address with the network, then retry.
- **The settings name a network the instance does not support.** Compare `oauthProvider` against the catalog; the value is case-sensitive.
- **`oauthAuthUrl` is missing**, or the authorization code was issued for a different redirect address than the one sent with it.

A returning person is recognised by the account id the network reports rather than by their address, so changing an email address at the network does not create a second account. One supported network reports no address at all; those accounts get a generated identifier instead. Signing in through two different networks still produces two separate users, even for one address.

## Common mistakes

- **Creating a permission that already exists.** Adjust the existing record.
- **Inventing a `type` for a social network.** It is always `oauth`, plus `oauthProvider`.
- **Creating a second `guest` group.** Succeeds silently, achieves nothing.
- **Granting an admin permission to fix a Content API 403.** Different model.
- **Rewriting group rules over a `403` on every route.** Check the token header first.
- **Sending only the groups you want to add.** `groupIds` replaces the set.
- **Updating a user without echoing back its form data.** It is overwritten, and that includes the sign-in field.
- **Missing a read restriction** and investigating the data instead.
- **Putting user details into a report.** Personal data stays in the instance.

→ `mcp/docs/api/content-api-permission-rules` · `mcp/docs/api/verification-recipes#permissions`
