# Data environments

An instance can hold more than one set of data. One of them is the main set, the one every instance has always had; the others are **environments**, created and named by an administrator. A request chooses which set it works in, and this document covers how it chooses, what the two sets share, and why nothing you write through this server reaches an environment.

Read it before acting on a sentence like "put this on staging". That phrase usually means an environment, and the answer is not a different address or a different login.

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission` · `mcp/docs/server/cms-api-call`

## A request names its environment in a header

Every Admin API and Content API route reads one request header — `x-environment` — and works in the environment it names. Send no header and the request works in the main set of data, exactly as it did before environments existed, so an integration that has never heard of them keeps working untouched.

The header is the only way to choose. There is no per-environment address, no per-environment subdomain and no token that selects one: the same base URL and the same credentials reach all of them, and the client decides on every single request.

`public` is the name of the main set and is accepted as a header value, so a client that would rather always send the header can name it explicitly instead of leaving it out.

## This server always works in the main data set

`cms_api_call` sends no `x-environment` header, and it has no parameter for one. Every read and every write you make through this server therefore works in the **main** set of data, whatever environments the instance holds.

Two consequences worth saying out loud instead of working around:

- **You cannot put content into an environment from here.** A human asking you to create a page "on staging" is asking for something this server does not do. Say so plainly, and leave it to the admin panel, which sends the header.
- **A write made while believing otherwise is a write to the live data.** Nothing fails and nothing warns: the call succeeds, and the entity exists in the set you did not mean.

What you can still do from here is tell a human which environments exist, read one by id, and — with the permission below — create, check or delete one. Those operations are about the registry of environments, not about the data inside them, so they work from this server like any other call.

## What a wrong environment name answers

The two API surfaces refuse differently, and the difference matters when you are reading somebody else's logs.

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

Separate per environment: pages, products, blocks, menus, forms and their submissions, visitors, orders, attribute sets — the content and the operational records. Each environment answers its own listings, and an id means nothing outside the set it came from.

Shared across all of them: administrator accounts and their permission maps, the admin sign-in itself, the active locales, and platform settings. One admin login works in every environment with no second sign-in, and granting a key grants it everywhere at once.

The instance's record quota is also shared, and it counts the main set and every ready environment **together**. `records-usage` answers the same totals whichever environment the request names, so an environment filled with test content spends the allowance of the live one.

→ `mcp/docs/api/settings#the-record-limit-and-what-it-feels-like`

## Application tokens belong to one environment

The application tokens a storefront reads the Content API with are **not** shared. A token issued in the main set does not appear in an environment's list and does not work there, and a newly created environment starts with none at all.

So standing up a storefront against an environment takes a token issued inside that environment, and the first read before anyone has issued one answers the ordinary token refusal. That is the second reason a Content API `401` follows an environment being named, and it is the one to check after the spelling.

→ `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Who may create and delete an environment

Reading the registry is open: `AdminEnvironmentsController_findAll` and `AdminEnvironmentsController_findOne` answer for any signed-in administrator, with no key at all. Everything else — creating, editing, deleting, checking one and copying content between two — needs `environments.manage` and answers `403 Forbidden resource` without it.

A record carries `name`, `description`, a `status` of `creating`, `ready`, `failed` or `deleting`, and an `error` that holds the cause when creating one did not finish. A failed record **stays** in the registry rather than disappearing, so a listing that holds an environment nobody can use is telling you to read its `error`, not to create it again.

The name is fixed once the environment exists: the update takes the `description` and nothing else. A `name` sent beside it answers `200` and changes nothing, so check the record you read back before telling anyone it was renamed.

→ `mcp/docs/api/silent-no-ops#why-a-read-back-is-the-only-detection`

## Creating and deleting one are both asynchronous

Neither call waits for the work. Create answers `201` with the record already in hand, `status: "creating"` — and until that status reads `ready`, every request naming the environment answers `400 Environment "<name>" is not ready`. So a create followed immediately by a write to the new environment fails on the second call, and the fix is to re-read the record until it is ready, not to resend anything.

Delete answers at once too, and the record then sits at `status: "deleting"` while its data is removed. That takes a noticeable while on a busy instance, and the record disappears from the listing only when the removal has finished. A name only becomes free to use again at that point.

Deleting takes the environment's whole set of data with it, and nothing keeps a copy. Treat it as the most destructive delete on the instance: show the human the environment's name, say what it holds, and wait for them to agree.

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission`

## Copying content from one environment to another

`AdminEnvironmentTransferController_transfer` carries content between two environments — entity by entity, by id, never as a whole-instance copy. `cms_api_describe` declares the body in full, including which kinds of entity may travel, so build it from there rather than from a guess.

Two things the operation's own declaration cannot tell you:

- **A real run is asynchronous.** It answers `201` with a `jobId` and an `items` list that is **empty**. That empty list is not "nothing was transferred" — the plan only ever comes back from a dry run. Read the entities in the target to confirm the outcome.
- **A dry run returns more than you asked for.** Whatever an entity needs in order to exist is added to the plan and marked `isDependency: true`, so one page can produce several items. Show the human the whole plan, not the ids they named.

Orders, visitors and submissions are not among the kinds that travel, and asking for one answers `400` — that is a boundary, not a gap to work around.

→ `mcp/docs/server/confirm-and-dry-run`

## Common mistakes

- **Believing a write from here reached an environment.** It reached the main set, and it answered `201` doing so.
- **Reading an Admin API `400` as a credentials problem.** The environment check runs before authentication.
- **Reading a Content API `401` as a dead token.** A misspelled environment name answers the same message.
- **Reusing an application token in another environment.** Tokens do not cross; issue one inside it.
- **Taking the empty `items` of a transfer as a failure.** The run is asynchronous; the plan is in the dry run.
- **Treating an environment as free space.** The record quota is shared with the live data.
- **Writing to an environment in the same breath as creating it.** It is not ready yet, and that `400` says so.
- **Creating a second environment because the first says `failed`.** Read its `error` first.
- **Reading a name still in the listing as `deleting` as a stuck delete.** The record goes when the data has gone.
