# Authentication and identity

How the server authenticates to the Admin API, what it does when a token expires, and where the permission list it enforces comes from.

Credentials never travel as tool arguments. In local mode they come from the environment; in remote mode from connection headers. A prompt-injected instruction therefore cannot swap identities.

→ `mcp/docs/server/configuration` · `mcp/docs/server/remote-mode`

## Two ways to authenticate

**Login and password.** Set `ONEENTRY_CMS_LOGIN` and `ONEENTRY_CMS_PASSWORD`. The server logs in against the Admin API and holds the resulting tokens in memory only. Nothing is written to disk.

**A pre-issued access token.** Set `ONEENTRY_CMS_TOKEN`. The token is used as-is. There is no refresh token in this mode, so when it expires the server cannot recover on its own — see below.

The password is only ever read from the environment; `--password` is refused outright, because command-line arguments are visible in the machine's process list.

## What happens when a token expires

Exactly one retry, and only on a 401:

1. The call answers 401.
2. The server refreshes and repeats the call once.
3. If refreshing fails and a login and password are available, it logs in again and repeats the call once.
4. If neither works, the call fails with an authentication error.

There is no blind retrying beyond that. A repeated mutation would risk creating duplicates, so nothing else is ever retried automatically — including a 5xx.

With a pre-issued token and no login and password, step 3 is unavailable:

```text
Access token rejected and no login/password available to re-authenticate.
```

## Developer accounts are rejected

The Admin API answers 401 for accounts flagged as developer accounts. Those belong to the Developer API, which this server does not expose.

If credentials that work elsewhere are rejected here, that is the first thing to check — the server appends a note to the 401 saying so. Use an ordinary admin account.

## Where the permission list comes from

After authenticating, the server reads the admin's own record and keeps the permissions whose value is true. That list is what the local pre-check compares against the permission an operation declares, and it is what `cms_whoami` reports.

Two behaviours worth knowing:

- If the read fails for any reason, the list comes back **empty**, which means "unknown" rather than "none". The local pre-check is then skipped entirely, and permission failures surface as 403 from the instance instead.
- The list is read once for the session. A grant made in the admin panel while your session is open is not picked up until the session reconnects.

→ `mcp/docs/api/admins-and-permissions`

## Identity in remote mode

A hosted server carries no credentials of its own. Each MCP session supplies them as connection headers when it initialises — `x-cms-token`, or `x-cms-login` together with `x-cms-password` — and the server keeps one identity, one token store and one confirm-token store per session.

Sessions never share any of that. Two agents connected to the same process act as two different admins, and a confirm token minted for one is meaningless to the other.

→ `mcp/docs/server/remote-mode#identity-is-per-session-and-comes-from-headers`

## Diagnosing an authentication problem

Call `cms_whoami` first. It distinguishes the two failure modes explicitly:

- `authenticated: false` with a hint about checking the login and password means credentials were configured and rejected — wrong password, wrong instance, unreachable instance, or a developer account.
- `authenticated: false` with a hint about setting the variables means no credentials reached the server at all — usually a variable name typo, or an MCP client that does not pass `env` through.

If `authenticated` is true but `admin.permissions` is empty, authentication worked and the permission read did not. Calls will still go out; they will just be refused by the instance rather than locally.

## What is never logged or stored

- Passwords and tokens exist only in memory, for the life of the process or the session.
- The audit log records a **hash** of a call's arguments, never the arguments themselves, and never any credential.
- Nothing is written to the cache directory except documentation and the instance's API document.

→ `mcp/docs/server/audit-log`
