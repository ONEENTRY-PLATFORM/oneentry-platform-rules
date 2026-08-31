# What a group permission decides about a public answer

Every Content API address has a permission record, and the rules on it for the reading group decide three things: whether the call is refused, how many records come back, and whether a visitor may write.

So a public answer that looks wrong — a list that stops at the same number every time, an upload a visitor should not have been able to make — is a rule far more often than a defect. Read the rules before you report either.

→ `mcp/docs/api/users-and-groups#permissions-are-per-route-and-already-exist` · `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## The five rules and what each one opens

`AdminUserPermissionsController_findAll` — `GET /api/admin/user-permissions` — lists the records. Each has a `path`, a `section`, and `rules.permissions` with five flags:

| Rule | What it opens |
|---|---|
| `readAllRule` | `GET`, the whole list |
| `readRestrictionRule` | `GET`, the list cut to the restricted length |
| `addRule` | `POST` |
| `changeRule` | `PUT` |
| `deleteRule` | `DELETE` |

Each flag may be written as `true`/`false` or as `1`/`0`; both are read the same way. A `GET` passes when either read rule is set, and everything else needs its own flag.

The flag follows the HTTP method, not the meaning of the call: a search whose query travels in the body is a `POST`, so `addRule` is what opens it.

## A restricted read caps the list at ten

`readAllRule: 0` with `readRestrictionRule: 1` is a read that succeeds and is trimmed. The public answer stops at the restricted length — ten records unless the instance says otherwise — and nothing in the response says it was cut. With `readAllRule: 1` the same call returns everything.

It applies to every public list on that path, and to a page's blocks exactly as it applies to a listing.

A list that returns the same small count for every query, on a project that plainly has more content, is this. Confirm it by reading the same data through the Admin API, which the restriction does not touch: two different counts settle it.

→ `mcp/docs/api/pages#why-a-page-returns-fewer-blocks-than-you-attached`

## Where the restricted length is set and read

The number lives under `user` in the general settings, as `restrictedDataLength`, and it is instance-wide rather than per path. On a fresh instance the key is not there at all and the fallback applies, so finding no such setting tells you nothing about what public reads are being cut to.

To read the value in force without guessing, `AdminUserPermissionsController_getHealth` — `GET /api/admin/user-permissions/health` — reports it in `config`:

```json
{ "config": { "RESTRICTED_DATA_LENGTH": { "live": 10, "fallback": 10 } } }
```

`live` is what public reads are being cut to now; `fallback` is what applies when the setting carries no value. The rest of that response is diagnostics for the panel.

## Paging cannot reach past the cap

A capped read still honours `limit` and `offset`, but only inside the window:

- `offset` at or past the cap returns an empty array, not the next page;
- `limit` is trimmed to what is left of the cap, so `offset: 5, limit: 50` on a cap of ten yields five records.

A site paging a capped listing therefore sees the first ten records and then nothing at all, whatever the total. Raising the setting or granting unrestricted read is the only way through — no request shape gets past it.

## What a new instance grants the guest group

Every content permission record provisioned with the instance is linked to the guest group, so the routes are open to anyone holding the application token from the start. Two things follow that are worth checking rather than assuming.

**Nearly every read route is provisioned as a restricted read.** So the ten-record ceiling is the default state of almost every public listing on a new project, and a fresh site hits it as soon as any list grows past it. A couple of routes ship with read-all instead — the active subscriptions list and the refund read — and the write-only routes carry no read rule at all, so treat "restricted" as the default to check rather than a guarantee.

**Many of them are provisioned with `addRule` on**, because the visitor-facing writes need it: sign-up and the other sign-in provider calls, form submissions, order creation, payment sessions, event subscription, the searches by meaning — and `/api/content/files`.

## Why a public token can upload a file

`/api/content/files` is provisioned with `addRule: true` for the guest group, so `POST` with a valid application token answers `201` on a new instance. That is the shipped default, not a hole someone opened, and it is what makes visitor uploads work out of the box.

Decide deliberately which way you want it. If the site never uploads, clear `addRule` on that record and the same call answers `403`. If it does upload, leave it and say so in the handover, because anyone with the token — which a browser necessarily has — can put files on the domain.

Toggling that one flag is also how you verify a report either way, in both directions, in under a minute.

→ `mcp/docs/api/files-and-uploads`

## Change a rule and make it take effect

`AdminUserPermissionsController_update` — `PUT /api/admin/user-permissions/{id}` — writes the record. `rules` is replaced whole, so send the complete object back with every flag and any `additionalData` it already carried; a partial `rules` silently drops the include and exclude lists with it.

```json
{
  "rules": {
    "permissions": {
      "readAllRule": 1,
      "readRestrictionRule": 0,
      "addRule": false,
      "changeRule": false,
      "deleteRule": false
    },
    "additionalData": {}
  }
}
```

A change is not visible to the public API immediately — a read can answer under the old rules for a few minutes. `AdminUserPermissionsController_flushCache` — `POST /api/admin/user-permissions/cache/flush` — makes it immediate, and needs the `userPermissions.update` permission. Without the flush, re-test after the wait rather than concluding the write did not land.

## Read the rules before reporting a public answer

Two shapes look exactly like defects and are not:

| What you see | Read this first |
|---|---|
| A list returns the same small count every time | `readRestrictionRule` on that path |
| A public token writes something you expected refused | `addRule` on that path |

The sequence is the same for both: find the record whose `path` matches the address the site called, read `rules.permissions`, flip the one flag, repeat the call. If the answer changes, the rules explained it and there is nothing to report.

## Common mistakes

- **Working around the ten-record ceiling in the content.** Merging articles into fewer, larger records to fit a cap is content damage in exchange for a setting.
- **Reading a trimmed list as the whole list.** Nothing marks it as cut.
- **Paging past the cap** and reporting the empty page as broken.
- **Assuming a new project ships with public writes closed.** File upload is open by default.
- **Sending a partial `rules` object.** It replaces the whole of it.
- **Re-testing a rule change in the same second** and concluding it did not apply.

→ `mcp/docs/api/users-and-groups#diagnosing-a-content-api-refusal` · `mcp/docs/api/verification-recipes#permissions`
