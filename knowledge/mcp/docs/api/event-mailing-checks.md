# Event mailing checks

What an event create or update accepts in its `mailing` object, and what happens to the event when the object is refused. Read it before writing a recurring mailing, or when a write that carries `mailing` answers `400`.

→ `mcp/docs/api/events#when-a-recurring-mailing-runs` · `mcp/docs/api/events`

## Which mailing values are refused

`mailing` is either an object or `null`. On `AdminEventsController_create` and `AdminEventsController_update` the object is checked before anything is written:

- `period` is required and must be one of `every_day`, `every_ndays`, `every_week`, `every_month`, `every_year`, `schedule`;
- `origin` is required and must be an ISO 8601 timestamp;
- with `period: every_ndays`, `ndays` is required and must be a whole number of 1 or more — a numeric string such as `"3"` is accepted;
- with `period: schedule`, `scheduleAttr` is required: the marker of a time-interval attribute of the event's attribute set;
- `conditions`, when sent, is an object keyed by locale code.

`ndays` and `scheduleAttr` are not checked for the other periods, so a leftover value from an earlier choice does not refuse the write.

Anything else — a string instead of the object, an unknown `period`, an unreadable `origin` — answers `400` naming the field.

```json
{ "mailing": { "period": "every_ndays", "origin": "2030-01-01T09:00:00Z",
               "ndays": 3, "conditions": {} } }
```

## What a refused mailing leaves behind

Nothing. A `400` on create means no event was created, so repeating the corrected request creates exactly one. A `400` on update leaves the stored event as it was, its name and its running recurrence included.

An update without `mailing` is checked against the mailing already stored. If that stored mailing is itself unusable, every update answers `400` until the request sends a valid `mailing`, or `mailing: null` to remove the recurrence.

→ `mcp/docs/api/events#when-a-recurring-mailing-runs` · `mcp/docs/api/events#checking-whether-an-event-actually-sent-mail`
