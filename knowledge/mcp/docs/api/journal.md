# The instance journal

The instance keeps its own record of what changed: which module, which entry, which admin, whether the call succeeded. Reading it answers "who did this and when". Clearing it is irreversible and takes an explicit time window.

This is **not** the audit file this server writes. Two separate records, kept by two different systems, and only one of them is on the instance.

→ `mcp/docs/server/audit-log` · `mcp/docs/api/admins-and-permissions`

## Reading the journal

`JournalController_list` — `GET /journal` — answers `{ "items": [...], "total": <number> }`. Reading needs `journal.get`. Without it the call answers `403 Forbidden resource`, and the message does not name the key. The same key covers the session-traffic listing beside it.

Narrow it with query parameters rather than paging and filtering yourself:

| Parameter | What it takes |
|---|---|
| `from`, `to` | ISO 8601; either may be omitted |
| `moduleName` | uppercase module name, one value or a comma-separated list |
| `moduleEntryId` | the id of the entry inside that module |
| `action` | `CREATE`, `UPDATE` or `DELETE` |
| `result` | `SUCCESS` or `FAILURE` |
| `adminId` | who made the change |
| `limit`, `offset` | pagination |

`action` is derived from the HTTP method of the original call and `result` from its status, so a refused call is recorded as `FAILURE` rather than left out. An entry carries `id`, `createdAt`, `moduleName`, `moduleEntryId`, `action`, `result` and the admin beside them.

## Clearing the journal needs an explicit window

`JournalController_clear` — `DELETE /journal` — deletes entries by time, and `from` and `to` are both **required**. There is no form of this call that means "everything".

```text
DELETE /journal?from=2026-01-01T00:00:00.000Z&to=2026-01-31T23:59:59.999Z
```

Omit either one, or send a `from` later than the `to`, and the call answers `400`. Narrow it further with the same `moduleName`, `action` and `result` the listing takes — those are optional, and leaving them off means every entry inside the window.

The window is the whole safety mechanism, so widen it deliberately and never by accident. A window opened far enough into the past takes everything before it, and nothing on the instance brings those entries back.

## The clear is gated on journal delete

Clearing requires `journal.delete`. Without it the call answers `403 Forbidden resource`, and the message does not name the key.

**The permission is checked before the window is validated.** A call with no `from` and no `to`, made without the grant, answers `403` rather than `400` — so the missing window is still there once the grant arrives. Fix the permission, then expect to fix the window too, and read the `400` as a second problem rather than a sign the first one was misdiagnosed.

`journal.delete` and `journal.get` are both keys an older admin account will not hold. Check it against the list of keys the instance recognises before treating the refusal as a configuration fault.

→ `mcp/docs/api/admins-and-permissions#a-key-can-exist-without-any-admin-holding-it`

## What the clear answers

A successful clear returns the count and nothing else:

```json
{ "affected": 42 }
```

`affected` is how many entries were removed. `0` is a complete answer, not a failure: it means the window held nothing, which is what an over-narrow `from`/`to` or a `moduleName` that matched no entry produces. Report the number rather than "done" — it is the only evidence of what the call actually took, and the entries are gone either way.

## Reading the session traffic listing

`AdminJournalTrafficController_getTraffic` — `GET /journal/traffic` — answers `{ "items": [...], "total": <number> }` and is gated on `journal.get`, the same key as the journal listing. It records sign-ins rather than changes: one record per admin sign-in, carrying `sessionId`, `adminId`, `login`, `email`, `loggedInAt`, `loggedOutAt`, `sessionDurationMs`, `closeReason`, `userAgent` and `ip`.

The window is the trap. With no `from` and no `to` the call covers **the last 24 hours only**, not everything on the instance, so a sign-in from last week is absent from the default answer rather than missing from the record. Ask for it:

```text
GET /journal/traffic?limit=10&offset=0&from=2026-01-01T00:00:00.000Z&to=2026-01-20T00:00:00.000Z
```

A `from` later than the `to`, or a span wider than 31 days, answers `400`. Narrow further with `adminId`, or with `status` — `active` for sessions still open, `closed` for sessions that have ended. Records come back newest sign-in first.

`loggedOutAt: null` marks a session that is still open, and `sessionDurationMs` for such a record is counted up to the moment of the read, so it grows between two calls that return the same `sessionId`.

## What closes a session record

`closeReason` says how a session ended, and it is the one field that explains an unexpected `401` on a token that worked a minute ago:

| Value | What happened |
|---|---|
| `logout` | the admin signed out |
| `admin_revoked` | sessions were revoked for every admin at once |
| `reauth` | a newer sign-in by the same admin superseded this session |
| `null` | the session is still open, or its token expired without being closed |

One admin holds **one** open session. Signing in again supersedes the previous session, closes its record as `reauth` and invalidates the access token that session was issued with — so two clients sharing one admin account take turns rather than working in parallel, and the loser sees `401`. Use a separate admin account per client instead of re-signing-in to recover a token.

Refreshing a token does not add a record. It extends the open session, keeps its `loggedInAt`, and returns a new access token — so the count of records answers "how many times did this admin sign in", not "how often was a token issued".

## Clearing session traffic is a separate call

`AdminJournalTrafficController_clear` — `DELETE /journal/traffic` — removes session-traffic records, and it is gated on `journal.delete`, the same key as the journal clear. Without the key the call answers `403 Forbidden resource` and the message does not name it; with no token at all it answers `401`.

Its window rules are **not** the journal's. `from` and `to` are each optional here, and whichever one you leave off is left unbounded — omit both and the call takes every closed session on the instance. No `400` will stop you. Send both, always.

```text
DELETE /journal/traffic?from=2026-01-01T00:00:00.000Z&to=2026-01-31T23:59:59.999Z
```

What you do send is checked: a `from` later than the `to`, or a span wider than 31 days, answers `400`. Narrow it further with `adminId`. A `status` parameter is accepted and then ignored — sessions that are still open are never removed, so this call cannot sign a working admin out.

The count comes back under a different name than the journal clear:

```json
{ "deleted": 0 }
```

`deleted: 0` means the window held nothing, exactly as `affected: 0` does on `JournalController_clear`.

## Not the same record as the servers audit log

Both answer "what changed", and mixing them up leads to reading the wrong one:

| | The instance journal | This server's audit log |
|---|---|---|
| Kept by | the instance | this server |
| Covers | every change, however it was made | only calls made through this server |
| Read with | `JournalController_list` | reading the file |
| Cleared by | `JournalController_clear` | rotating the file yourself |

A change made in the admin panel is in the journal and not in the audit log. A call this server refused before sending is in the audit log and not in the journal — nothing reached the instance to record.

→ `mcp/docs/server/audit-log#reads-are-not-recorded`

## Common mistakes

- **Expecting a clear with no window to work.** `from` and `to` are required.
- **Reading a `403` on the clear as a bad window.** The permission is checked first.
- **Treating `affected: 0` as an error.** The window was empty.
- **Widening the window to be safe.** Wider deletes more, and the entries do not come back.
- **Expecting the traffic clear to reject a missing window.** It accepts one and takes everything closed.
- **Reading an empty traffic listing as no sign-ins.** With no window it covers the last 24 hours.
- **Signing in again to recover a token.** It invalidates the one the other client is using.
- **Looking for a panel change in this server's audit log.** It is in the journal.
- **Using the journal to recover a deleted entity.** It records that the change happened, not the entity.

→ `mcp/docs/server/confirm-and-dry-run` · `mcp/docs/api/admins-and-permissions#permissions-are-dotted-keys`
