# Journal retention, the append-only switch and journal export

How long the instance keeps its three admin journals, the switch that makes them append-only for good, and how to copy the action journal into another system without losing or repeating a record.

Read this before changing the retention settings, when a journal clear answers `403` to an admin who holds `journal.delete`, or when someone asks for the journal to be shipped to an external log store.

→ `mcp/docs/api/journal#reading-the-journal` · `mcp/docs/api/content-api-journal#clearing-the-error-journal-takes-an-optional-window` · `mcp/docs/api/settings#general-settings`

## Which settings decide how long the journal is kept

Retention lives in the general settings, under `user.journalRetention`, and is written with `PUT /api/admin/settings-general`. Writing it needs `settings.general.update`.

| Field | What it takes |
|---|---|
| `journalDays` | days the action journal is kept |
| `trafficDays` | days the admin session traffic is kept |
| `contentApiErrorsDays` | days the public API error journal is kept |
| `autoCleanupEnabled` | boolean; whether the scheduled cleanup removes records older than those periods |
| `immutable` | boolean; makes all three journals append-only, permanently |

The day counts are whole numbers, `0` or more; `0` never means "delete everything": the cleanup does not clear on a zero period. A fraction, a negative number, a string in place of a number or of a boolean answers `400` and saves nothing.

```json
{
  "user": {
    "journalRetention": {
      "journalDays": 365,
      "trafficDays": 90,
      "contentApiErrorsDays": 30,
      "autoCleanupEnabled": true
    }
  }
}
```

Read the record first and send only the keys you are changing, as with every general setting. Leave `immutable` out of a retention change unless the operator has asked for it.

## Making the journal append-only

`immutable: true` inside `user.journalRetention` turns the admin journals into an append-only record. From that moment:

- `DELETE /api/admin/journal`, `DELETE /api/admin/journal/traffic` and `DELETE /api/admin/journal/content-api-errors` answer `403`, whatever window you send;
- the scheduled cleanup removes nothing, whatever `autoCleanupEnabled` and the day counts say;
- new entries keep arriving and every read works as before.

The switch cannot be undone through the API. **Never turn it on yourself.** Ask the operator first, say plainly that it is one-way, and turn it on only on an explicit yes. It exists for projects whose contract forbids deleting audit records.

## Why clearing the journal answers 403 with journal delete

With the append-only switch on, a clear answers `403` with the message `The journal is immutable: its records cannot be deleted`, even for an admin who holds `journal.delete`. No key, role or window changes that answer, so do not grant permissions, retry or reshape the window to get past it. Report that the journal on this instance is append-only.

The permission is still checked first. An admin without `journal.delete` gets the ordinary `403 Forbidden resource`, and only once the key is in place does the append-only refusal appear.

To check which case you are in, read `user.journalRetention.immutable` from `GET /api/admin/settings-general`.

## The append-only switch cannot be turned off

Once `immutable` is `true`, the instance holds it there:

| You send | What happens |
|---|---|
| `"immutable": false` | `400`; nothing in the request is saved |
| a `journalRetention` object without `immutable` | saved, `immutable` stays `true`, `autoCleanupEnabled` is set to `false` |
| `"journalRetention": null` | same as above |

So the day counts can still be edited, but cleanup stays off and the flag stays on. Read the record back after any retention write and report the `autoCleanupEnabled` you find there, not the one you sent. Do not try another body shape to switch it off. There is none.

## Exporting the journal to another system

The action journal can be copied out by paging `GET /api/admin/journal` forward in time with a cursor. Reading needs `journal.get`.

```text
GET /api/admin/journal?from=2026-11-03T10:15:00.000Z&sortByCondition=createdAt&sortByDirection=ASC&limit=200
```

- Keep the `createdAt` of the last record the receiver accepted as the cursor, and send it as `from`.
- `from` is inclusive, so the first records of a page can be ones you already sent. Skip a record whose `createdAt` equals the cursor and whose `id` was already delivered.
- `limit` goes up to 200. A page shorter than the limit means you have caught up. Wait, then ask again.
- Move the cursor only after the receiver confirms with a `2xx`. A failed delivery leaves the cursor where it was, so the next run sends the same records again rather than skipping them.

The admin panel ships a workflow template named "Journal export" that runs this loop on a 60-second timer. Start from it instead of building the flow by hand.

→ `mcp/docs/api/workflows#run-a-flow-without-waiting-for-its-timer`

## Common mistakes

- **Turning on `immutable` as part of a routine settings change.** It is one-way. Ask the operator first.
- **Sending `immutable: false` to undo it.** It answers `400` and saves nothing.
- **Granting more keys after a `403` on a clear.** On an append-only journal no key opens it.
- **Expecting cleanup after a retention write on an append-only journal.** `autoCleanupEnabled` comes back `false`.
- **Advancing the export cursor before the receiver answers.** A failed delivery then loses records.
- **Dropping every record at the cursor time.** Only skip ids you have already delivered.

→ `mcp/docs/api/journal#the-clear-is-gated-on-journal-delete` · `mcp/docs/api/settings#changing-any-setting`
