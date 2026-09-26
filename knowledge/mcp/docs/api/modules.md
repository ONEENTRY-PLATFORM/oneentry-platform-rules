# Modules

A module is a section of the admin panel — catalogue, orders, forms, users — and the thing general types and attribute set types attach to so entities appear in the right place.

Eighteen are provisioned. Mutating them is permanently confirm-gated, and almost every task that seems to need it does not.

→ `mcp/docs/api/baseline-data#modules` · `mcp/docs/api/general-types`

## The modules you will find

`settings`, `forms`, `catalog`, `content`, `admins`, `blocks`, `journal`, `menu`, `users`, `payments`, `events`, `orders`, `workflows`, `collections`, `discounts`, `import-data`, `subscriptions`, `filters`, `file-search`

Read them rather than trusting the list:

```text
cms_api_search { "query": "modules", "method": "get" }
```

## Address them by identifier

The identifier is the stable handle. Ids are local to the instance, and while these particular ones tend to be stable, nothing guarantees it.

Identifiers here are **not unique**, which is the dangerous part: creating a second module called `catalog` succeeds, produces a shadow module wired to nothing, and everything keyed off the real one carries on ignoring it.

## Why mutations are gated

Modules are the wiring the admin panel hangs off. General types attach to them; attribute set types attach to them; visibility of whole sections follows from them.

A change here does not break one entity, it changes what an operator can see and reach. That is why every mutation needs a confirm token at every allow level, and why the right response to "the module configuration needs changing" is usually to ask why.

→ `mcp/docs/server/allow-levels#paths-that-are-always-confirm-gated`

## Module visibility

A module can be hidden, which removes its section from the admin panel without deleting anything. Content of that kind still exists and is still reachable through the API.

So "the orders section is missing" may be visibility rather than data, and hiding a module is a reversible way to simplify a panel — unlike deleting anything inside it.

## Modules and permissions

What an admin can do inside a module is governed by permission keys, not by the module record. Hiding a module does not remove rights, and granting rights does not reveal a hidden module.

Diagnose the two separately.

→ `mcp/docs/api/admins-and-permissions`

## When a task really involves modules

Rarely. The legitimate cases are inspection — reading which modules exist, to resolve a general type's placement or explain why a section is absent — and, occasionally, visibility changes an operator explicitly asks for.

Creating a module is not how a new content kind is introduced; general types and attribute sets are.

## If you think you need to change one

1. Say what you believe the change achieves.
2. Read the module and show the human its current state.
3. Dry run and show the target.
4. Wait for explicit agreement, then confirm.

If the goal is "make this content appear in the admin panel", the answer is almost certainly a general type or an attribute set type, not a module.

## What a custom module needs before it deploys

Besides the provisioned panel sections, an instance can run a **custom** module: a container image the platform pulls and runs for the project. Creating the record and running the container are two separate calls, and creating the record starts nothing.

The record is created with `type` of `custom` and carries a free-form `config` object. The deploy call takes **no request body** — it reads that stored `config`. Exactly one option is required: `config.docker.image`, the image reference including its tag. Everything else is optional: `config.docker.host`, `config.docker.user` and `config.docker.pass` for an image in a private registry, and `config.env` — an **array** of `{ "name": …, "value": … }` pairs — for environment variables.

```json
{
  "identifier": "agenda_sync",
  "localizeInfos": { "en_US": { "title": "Agenda sync" } },
  "type": "custom",
  "config": {
    "docker": { "image": "registry.your-instance.example/org/app:1.0.0" },
    "env": [{ "name": "TARGET_URL", "value": "https://your-instance.example/feed" }]
  }
}
```

Any other key in `config` is accepted, stored and read back — and ignored when the module runs. A top-level `image`, a `command`, a `port`, a `repository` or a `git` object do nothing: the platform pulls a prebuilt image and does not build from source. Processor and memory limits are not taken from `config` either; they come from the instance settings.

The create call validates none of this, so a wrong shape surfaces only at deploy: `400 Missing required options to deploy: config.docker.image`. The fix is to put the image under `config.docker.image` — as an object, not a string — and deploy again. Deploying a module that already runs replaces its container with the current `config`, so there is no separate update call to reach for.

The deploy call answers with the identifier of the asynchronous task that creates the container, not with the container itself. Read the module state afterwards to see what happened.

## What the container status values mean

A container state read answers `status` and `actionRunning`. While an operation is in flight, `actionRunning` is `true` and `status` is that operation: `deploying`, `suspending`, `unsuspending` or `deleting`. Otherwise `status` describes the container:

- `Not found` — no container exists for this module. **This is what a freshly created custom module reports**; it means "not deployed yet", not an error. Deploy is what creates the container.
- `Not ready` — starting up. Read again; a container that stays `Not ready` long enough is reported as `Failed`.
- `Running latest version` — running the image from the current `config`.
- `Running old version` — running an older image. Deploy again to roll it forward.
- `Suspended` — stopped by a suspend call, and resumable.
- `Failed` — the last operation did not succeed. Read the container log before deploying again.

A state read needs a reachable container platform, so on an instance where it is unavailable this read answers 5xx while the record itself still reads back normally.

## Why a module cannot run on a schedule

There is no schedule, interval or cron option — not in `config`, not on any module call. A `schedule` key in `config` is one of the ignored keys above: it is stored, echoed back, and does nothing. Deploy starts a long-running container and leaves it running; the platform only starts, suspends, resumes and deletes it.

So recurring work belongs inside the image: the container schedules itself and stays up between runs. Scheduled events are not an alternative — an event's actions send notifications and cannot refresh content.

## What a container log read returns

A container log read answers with an object, not a bare list: `lines` holds the log rows, `total` is how many rows came back, and `streams` is how many log streams the read covered.

A module running in several replicas produces one stream per replica. `lines` carries the rows of every stream, merged and sorted by time, oldest first — so a single read is the whole picture, not one replica's view.

`total` and `streams` are what make an empty result readable. `streams` of zero means nothing matched the module's container at all; `streams` above zero with `total` of zero means the stream exists and had no rows in the window you asked for. Check both before concluding that a line you are looking for was never written — widen the window or fix the module identifier instead.

Two refusals here belong to the read and not to the module. A bound that is not a date, or a start later than the end, answers `400` naming the field, and nothing is fetched. When the log store cannot be reached the answer is `502 Log storage is currently unavailable` — a statement about the store, carrying nothing about the module and nothing that narrowing the window will change. Report it and stop; a tight retry loop is the wrong response to either.

## Common mistakes

- **Creating a module.** Identifiers are not unique; you get a shadow.
- **Changing module wiring to fix a missing entity.** Look at the general type instead.
- **Confusing visibility with permissions.** Two independent mechanisms.
- **Treating the gate as a formality.** It exists because these changes are wide.
- **Asking for a month of container logs in one call.** The window is capped at seven days.
- **Expecting a created custom module to be running.** Creating the record starts nothing; deploy does.
- **Putting the image anywhere but `config.docker.image`.** Other keys are stored and ignored.
- **Looking for a schedule option.** There is none; the container schedules itself.
