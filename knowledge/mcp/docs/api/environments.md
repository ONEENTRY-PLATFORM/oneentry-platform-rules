# Data environments

An instance can hold more than one set of data. One of them is the main set, the one every instance has always had; the others are **environments**, created and named by an administrator. A request chooses which set it works in, and this document covers how it chooses, what the two sets share, and how to tell which one your own calls are reaching.

Read it before acting on a sentence like "put this on staging". That phrase means an environment, and whether you can reach one is a property of how this server was started.

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission` · `mcp/docs/server/cms-api-call`

## A request names its environment in a header

Every Admin API and Content API route reads one request header — `x-environment` — and works in the environment it names. Send no header and the request works in the main set of data, exactly as it did before environments existed, so an integration that has never heard of them keeps working untouched.

The header is the only way to choose. There is no per-environment address, no per-environment subdomain and no token that selects one: the same base URL and the same credentials reach all of them, and the client decides on every single request.

`public` is the name of the main set, accepted as a header value, so a client may name it explicitly instead of leaving the header out.

## Which data set your own calls reach

**Read this before your first write on any instance that has environments.** You do not choose: the operator does, once, when the server starts. No tool takes an environment argument, and nothing you can pass to `cms_api_call` changes the set your call lands in.

`cms_whoami` is the answer, under `environment`:

| What it says | What it means |
|---|---|
| a name | every call of this session reads and writes **that** environment, and the main set is not reachable from here |
| `null` | every call works in the main set of data |
| no such field at all | the server predates environments and sends no header, so every call works in the main set |

So a human asking you to add a page "on staging" is asking a question about the server's configuration, not about your payload. Check `cms_whoami` first. If it does not name that environment, say so and stop — the work needs a server pointed at it, or a person in the admin panel.

**The failure this prevents has no error to catch.** An instance ignores no header and refuses no missing one: a write meant for an environment, sent without the header, lands in the live data and answers success. Nothing in the response distinguishes the two, which is why reading `cms_whoami` once at the start of the task is the whole defence.

The registry operations are another matter and work from any session: they act on the list of environments, not on the data inside one.

## What a wrong environment name answers

The two API surfaces refuse differently, which matters when you are reading somebody else's logs.

| Surface | Answer |
|---|---|
| Admin API | `400` naming the cause — `Unknown environment "<name>"`, `Environment "<name>" is not ready`, or `Invalid x-environment header value` |
| Content API | `401 Invalid or missing app token` |

On the Admin API the refusal comes **before authentication**: a request with no credentials at all and a bad environment name answers `400`, not `401`. So a `400` here is never about your token, and a token fix never clears it.

On the Content API the same mistake is indistinguishable from a token problem — an unknown name and a malformed name both answer the token message. When a storefront starts answering `401` right after somebody pointed it at an environment, check the spelling of the name before reissuing anything.

## The shape an environment name may take

A name that does not fit answers `400 Invalid x-environment header value` on any route, and that refusal is about the characters, not about whether the environment exists:

- lowercase letters, digits and underscores only, starting with a letter;
- no two underscores in a row and no trailing underscore;
- at most forty characters.

So `dev_env` and `stage2` are names; `Dev_env`, `1env`, `dev-env`, `dev__env` and `dev_env_` are not. Read that `400` as a typo in the header rather than as a missing environment — the message for a missing one names it.

## What the two sets share and what they do not

Separate per environment: pages, products, blocks, menus, forms and their submissions, visitors, orders, attribute sets — content and operational records alike. Each answers its own listings, and an id means nothing outside the set it came from.

Shared across all of them: administrator accounts and their permission maps, the admin sign-in itself, the active locales, and platform settings. One admin login works in every environment with no second sign-in, and granting a key grants it everywhere at once.

The record quota is shared too, counting the main set and every ready environment **together**: `records-usage` answers the same totals whichever environment the request names, so an environment filled with test content spends the live one's allowance.

→ `mcp/docs/api/settings#the-record-limit-and-what-it-feels-like`

## Application tokens belong to one environment

The application tokens a storefront reads the Content API with are **not** shared. A token issued in the main set does not appear in an environment's list and does not work there, and a newly created environment starts with none at all.

So a storefront pointed at an environment needs a token issued inside it, and a read before anyone has issued one answers the ordinary token refusal. That is the second reason a Content API `401` follows an environment being named — check it after the spelling.

→ `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Who may create and delete an environment

Reading the registry is open: `AdminEnvironmentsController_findAll` and `AdminEnvironmentsController_findOne` answer for any signed-in administrator, with no key at all. Everything else — creating, editing, deleting, checking one and copying content between two — needs `environments.manage` and answers `403 Forbidden resource` without it.

A record carries `name`, `description`, a `status` of `creating`, `ready`, `failed` or `deleting`, and an `error` that holds the cause when creating one did not finish. A failed record **stays** in the registry rather than disappearing, so a listing that holds an environment nobody can use is telling you to read its `error`, not to create it again.

The name is fixed once the environment exists: the update takes the `description` and nothing else. A `name` sent beside it answers `200` and changes nothing, so check the record you read back before telling anyone it was renamed.

→ `mcp/docs/api/silent-no-ops#why-a-read-back-is-the-only-detection`

## Creating and deleting one are both asynchronous

Neither call waits for the work. Create answers `201` with the record already in hand, `status: "creating"` — and until that status reads `ready`, every request naming the environment answers `400 Environment "<name>" is not ready`. So a create followed immediately by a write to the new environment fails on the second call, and the fix is to re-read the record until it is ready, not to resend anything.

Delete answers at once too, and the record then sits at `status: "deleting"` while the data goes. That takes a noticeable while, and the record leaves the listing only when the removal has finished — which is also when the name becomes free again.

Deleting takes the environment's whole set of data with it, and nothing keeps a copy. Treat it as the most destructive delete on the instance: show the human the environment's name, say what it holds, and wait for them to agree.

Nothing here happens quietly, because every mutation under the path is confirm-gated at every allow level — creating, editing, deleting and copying alike. A dry run gives you the target; show it and wait.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission`

## Copying content from one environment to another

`AdminEnvironmentTransferController_transfer` carries content between two environments — entity by entity, by id, never as a whole-instance copy. `cms_api_describe` declares the body in full, including which kinds of entity may travel, so build it from there rather than from a guess.

Two things the operation's own declaration cannot tell you:

- **A real run is asynchronous.** It answers `201` with a `jobId` and an `items` list that is **empty**. That empty list is not "nothing was transferred" — the plan only ever comes back from a dry run. Read the entities in the target to confirm the outcome.
- **A dry run returns more than you asked for.** Whatever an entity needs in order to exist is added to the plan and marked `isDependency: true`, so one page can produce several items. Show the human the whole plan, not the ids they named.

Orders, visitors and submissions are not among the kinds that travel, and asking for one answers `400` — that is a boundary, not a gap to work around.

→ `mcp/docs/server/confirm-and-dry-run`

## Common mistakes

- **Assuming a write reached an environment.** Ask `cms_whoami` which one this session is in; a missing header is never refused.
- **Reading an Admin API `400` as a credentials problem.** The environment check runs before authentication.
- **Reading a Content API `401` as a dead token.** A misspelled environment name answers the same message.
- **Reusing an application token in another environment.** Tokens do not cross; issue one inside it.
- **Taking the empty `items` of a transfer as a failure.** The run is asynchronous; the plan is in the dry run.
- **Writing to an environment in the same breath as creating it.** It is not ready yet, and that `400` says so.
- **Creating a second environment because the first says `failed`.** Read its `error` first.
