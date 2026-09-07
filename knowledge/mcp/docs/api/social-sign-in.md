# Signing in with a social network

Configuring a sign-in provider for Google, Apple, VK, Yandex, Microsoft, Facebook, GitHub or Reddit, and reading the refusals that follow. Covers the settings each network needs, when those settings are checked, and what a social sign-in does to the user record.

Read this before creating an `oauth` provider or diagnosing a 400 from one. The sign-in call itself, its required device header and the session rules around it live with the other visitor routes.

→ `mcp/docs/api/content-api-sign-in-and-cart#sign-in-needs-the-x-device-metadata-header` · `mcp/docs/api/users-and-groups`

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

## Settings are checked when you save them

The create and update calls validate the settings of an `oauth` provider. Two things are refused with `400`:

- an `oauthProvider` the instance does not support — the message lists the values it accepts;
- a key the chosen network requires, left empty or absent — the message names each missing key.

So a provider that saves successfully is one the catalog agrees is complete. Send the settings the catalog asks for and read the message rather than guessing when it refuses; the value is case-sensitive.

Providers of other types are not checked this way, and settings stored before this check existed are left alone — they are still read at sign-in time.

## Keep the name and picture from the network

By default the network's display name and avatar are used to complete the sign-in and are then discarded, because the instance cannot know which of your form fields should receive them.

Name the fields and they are kept:

```json
{ "oauthNameMarker": "full_name", "oauthAvatarMarker": "photo" }
```

Both are optional. Four rules govern them:

- Values are written **only when the account is created**. A later sign-in never overwrites what the person has since edited.
- A marker that is not a field on the provider's form is skipped, as is a provider with no form at all. A sign-in never fails because of these settings.
- Only text fields can receive them. An avatar pointed at an image field is skipped: image fields hold uploaded files, and a network supplies a link. To keep the picture as a file, fetch and upload it yourself.
- The avatar is stored as the link the network returned. It may expire or require the person to still have an account there.

One network supplies no display name at all, so `oauthNameMarker` has no effect for it even when the field exists.

## Why a social sign-in answers 400

- **The network reported an address it has not verified.** The sign-in is refused so an unverified address cannot claim an existing account. Ask the person to confirm their address with the network, then retry.
- **The stored settings name a network the instance does not support.** Compare `oauthProvider` against the catalog.
- **`oauthAuthUrl` is missing**, or the authorization code was issued for a different redirect address than the one sent with it.
- **The authorization code was already used, or has expired.** These codes are single-use and short-lived; obtain a fresh one rather than replaying the request.

When the network itself rejects the exchange, the response says only that the attempt failed. Check the credentials and the redirect address registered with the network — the same code cannot be retried.

## How a returning person is recognised

A returning person is matched on the account id the network reports rather than on their address, so changing an email address at the network does not create a second account.

Two consequences worth planning for:

- One supported network reports no address at all. Those accounts are given a generated identifier, and they work normally.
- Signing in through two different networks produces two separate users, even for one address, and even when both addresses match. There is no call that merges them.

## Common mistakes with social sign-in

- **Inventing a `type` for a social network.** It is always `oauth`, plus `oauthProvider` in the settings.
- **Filling in `oauthSecret` for Apple.** It is ignored; supply the key material instead.
- **Saving settings without consulting the catalog.** The call refuses and names what is missing.
- **Expecting the network's name and picture to be stored.** Name the fields first.
- **Retrying a failed sign-in with the same authorization code.** It is already spent.
- **Treating two accounts for one person as a fault.** Each network is its own account.

→ `mcp/docs/api/users-and-groups#working-with-users` · `mcp/docs/api/content-api-sign-in-and-cart#where-the-session-routes-live` · `mcp/docs/api/forms-and-form-data`
