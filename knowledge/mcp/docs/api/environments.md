# Data environments

An instance can hold more than one set of data. One of them is the main set, the one every instance has always had; the others are **environments**, created and named by an administrator. A request chooses which set it works in, and this document covers how it chooses, what the two sets share, and how to tell which one your own calls are reaching.

Read it before acting on a sentence like "put this on staging". That phrase means an environment, and whether you can reach one is a property of how this server was started.

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission` · `mcp/docs/api/environment-markers` · `mcp/docs/server/cms-api-call`

## A request names its environment in a header

Every Admin API and Content API route reads one request header — `x-environment` — and works in the environment it names. Send no header and the request works in the main set of data, exactly as it did before environments existed, so an integration that has never heard of them keeps working untouched.

The header is the way to choose. There is no per-environment subdomain and no token that selects one: the same base URL and the same credentials reach all of them, and the client decides on every single request. The one exception is the payment-provider callback, described below, because a provider cannot be told to send a header.

`public` is the name of the main set, accepted as a header value, so a client may name it explicitly instead of leaving the header out.

A request may also name an environment by **marker** — a stable name bound to one environment, re-pointable without touching the clients that send it — in `x-environment-marker`.

→ `mcp/docs/api/environment-markers#a-marker-is-a-second-way-to-name-an-environment`

## Payment callbacks name the environment in the address

A payment provider calls `POST /api/content/payments/webhook/{marker}` with headers of its own choosing, so it can never send `x-environment`. On that path only, the environment may instead be named in the query string: `?environment=<name>` or `?environmentMarker=<marker>`. Without either, the callback works in the main set, and a payment account that exists only inside an environment answers `404 Payment account not found`.

The query parameters are checked exactly like the headers: an unknown or malformed name answers `401 Invalid or missing app token`. A request that carries both a header and its query parameter must give them the same value (`x-environment` with `?environment=`, `x-environment-marker` with `?environmentMarker=`), and each parameter may appear in the address only once; a disagreement or a repeated parameter answers the same `401`. When the two agree, the request works in that environment. On every other route these parameters select nothing.

So the callback address registered in a provider's dashboard for an environment is the ordinary webhook address plus `?environment=<name>`. A provider whose callback the platform registers itself receives that address automatically when the registration is made from inside an environment.

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

The sender address of event e-mails is separate too. A write that names an environment changes the sender for that environment only; the main set and the other environments keep theirs. Read it back in the environment that will send the mail.

Shared across all of them: administrator accounts and their permission maps, the admin sign-in itself, the active locales, and platform settings. One admin login works in every environment with no second sign-in, and granting a key grants it everywhere at once.

The record quota is shared too, counting the main set and every ready environment **together**: `records-usage` answers the same totals whichever environment the request names, so an environment filled with test content spends the live one's allowance.

The storage allowance works the same way, and copying content spends it: a file that travels with an entity is stored a second time under the target environment and counts against the one instance-wide total, exactly as an upload does. Over the allowance the file is not stored and the entity does not arrive whole, so make room or raise the allowance before copying a media-heavy set — repeating the call changes nothing. A dry run cannot tell you whether there is room; read the storage figures first.

Deleting an environment gives that space back. Once the removal finishes, the files stored under it are gone and the instance-wide storage figure drops by what they took, so removing a test environment you no longer need is a real way to make room. Read the figure again after the record has left the listing, not while it still reads `deleting`.

→ `mcp/docs/api/settings#the-record-limit-and-what-it-feels-like`

→ `mcp/docs/api/files-and-uploads#no-file-type-is-refused`

## Application tokens belong to one environment

The application tokens a storefront reads the Content API with are **not** shared. A token issued in the main set does not appear in an environment's list and does not work there, and a newly created environment starts with none at all.

So a storefront pointed at an environment needs a token issued inside it, and a read before anyone has issued one answers the ordinary token refusal. That is the second reason a Content API `401` follows an environment being named — check it after the spelling.

→ `mcp/docs/api/content-api-reads#public-reads-use-the-x-app-token-header`

## Who may create and delete an environment

Reading the registry is open: `AdminEnvironmentsController_findAll` and `AdminEnvironmentsController_findOne` answer for any signed-in administrator, with no key at all. Everything else — creating, editing, deleting, checking one and copying content between two — needs `environments.manage` and answers `403 Forbidden resource` without it. An admin provisioned with the instance holds that key, so check your own map before treating the refusal as a grant to ask for.

A record carries `name`, `description`, a `status` of `creating`, `ready`, `failed` or `deleting`, and an `error` that holds the cause when creating one did not finish. A failed record **stays** in the registry rather than disappearing, so a listing that holds an environment nobody can use is telling you to read its `error`, not to create it again.

Creating one refuses a few names the header shape would allow: `public` (the main set), `information_schema`, and any name starting with `pg_`. Those answer `400` on create instead of producing a record, so pick another name rather than retrying.

The name is fixed once the environment exists: the update takes the `description` and nothing else. A `name` sent beside it answers `200` and changes nothing, so check the record you read back before telling anyone it was renamed.

→ `mcp/docs/api/silent-no-ops#why-a-read-back-is-the-only-detection`

## Creating and deleting one are both asynchronous

Neither call waits for the work. Create answers `201` with the record already in hand, `status: "creating"` — and until that status reads `ready`, every request naming the environment answers `400 Environment "<name>" is not ready`. So a create followed immediately by a write to the new environment fails on the second call, and the fix is to re-read the record until it is ready, not to resend anything.

Delete answers at once too, and the record then sits at `status: "deleting"` while the data goes. That takes a noticeable while, and the record leaves the listing only when the removal has finished — which is also when the name becomes free again.

A transfer that is still running into or out of the environment holds the removal back: the data goes only after that run finishes, so the record can stay at `deleting` for as long as the transfer takes. That is not a stuck delete — keep re-reading the record.

Deleting one that still reads `creating` is accepted the same way. The removal waits until creating it has finished, so the record reads `deleting` for that time too and then leaves the listing; it does not pass through `ready` or `failed` on the way.

Deleting takes the environment's whole set of data with it, and nothing keeps a copy. Treat it as the most destructive delete on the instance: show the human the environment's name, say what it holds, and wait for them to agree.

Nothing here happens quietly, because every mutation under the path is confirm-gated at every allow level — creating, editing, deleting and copying alike. A dry run gives you the target; show it and wait.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

→ `mcp/docs/api/admins-and-permissions#managing-data-environments-has-its-own-permission`

## Copying content from one environment to another

`AdminEnvironmentTransferController_transfer` carries content between two environments — entity by entity, by id, never as a whole-instance copy. `cms_api_describe` declares the body in full, including which kinds of entity may travel, so build it from there rather than from a guess.

Three things the operation's own declaration cannot tell you:

- **A real run is asynchronous.** It answers `201` with a `jobId` and an `items` list that is **empty**. That empty list is not "nothing was transferred" — the plan only ever comes back from a dry run. When the run finishes, the **target** environment's journal gets one entry for it, module `ENVIRONMENT_TRANSFER`: read the journal with the target selected, whichever environment sent the request. Its `delta.items` lists every entity with its `status` and `targetId`; `result` is `FAILURE` when the run stopped or any entity failed. A dry run leaves no entry.
- **A dry run returns more than you asked for.** Whatever an entity needs in order to exist is added to the plan and marked `isDependency: true`, so one page can produce several items. Show the human the whole plan, not the ids they named.
- **Name each plan item by its `identifier`, not its `sourceId`.** `identifier` on a plan item is the key the entity is known by in the source, and which key that is depends on the kind: a page is named by its address (`pageUrl`), an error page by its `code`, every other kind by its own `identifier`. It is `null` only where the source has left that key empty. `sourceId` is a position inside the source environment and means nothing in the target, so a plan read out as a list of numbers tells the human nothing about what is about to be written.

A dry run item at `status: "duplicate"` is not written by a real run until you resolve it, and `overwrite` reaches the dependencies an element pulls in — both are covered with grouping several kinds into one run in `mcp/docs/api/environment-transfer`.

Orders, visitors and submissions are not among the kinds that travel, and asking for one answers `400` — that is a boundary, not a gap to work around.

Project settings travel only in part. Error pages, sign-in providers and events are kinds you may name, and a sign-in provider arrives with its settings but **without its secrets** — its signing key and OAuth secret stay in the source, and a dry run says so in `warnings`. A provider the transfer creates gets a fresh signing key of its own, so tokens issued in one environment are never valid in the other; the OAuth secret stays empty, so fill it in before anyone signs in to the target through that OAuth provider. An update keeps the secrets the target already has. Order storages with their statuses, and Content API access rules, do not travel at all: a dry run lists the ones the source has filled in, with counts on both sides, in `warnings`, and the human sets them up by hand.

→ `mcp/docs/server/confirm-and-dry-run`

## Common mistakes

- **Assuming a write reached an environment.** Ask `cms_whoami` which one this session is in; a missing header is never refused.
- **Reading an Admin API `400` as a credentials problem.** The environment check runs before authentication.
- **Reading a Content API `401` as a dead token.** A misspelled environment name answers the same message.
- **Reusing an application token in another environment.** Tokens do not cross; issue one inside it.
- **Taking the empty `items` of a transfer as a failure.** The run is asynchronous; the outcome is in the target's journal.
- **Treating a copy into an environment as free.** Copied files spend the one storage allowance; check it first.
- **Expecting a copied OAuth sign-in provider to let anyone in.** Its OAuth secret stayed behind; fill it in first.
- **Writing to an environment in the same breath as creating it.** It is not ready yet, and that `400` says so.
- **Creating a second environment because the first says `failed`.** Read its `error` first.
- **Editing a client's configuration to move it between environments.** Bind a marker and re-point that instead — see `mcp/docs/api/environment-markers`.
