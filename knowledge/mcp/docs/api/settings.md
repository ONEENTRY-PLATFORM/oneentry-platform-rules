# Settings

Instance-wide configuration: general settings, plan limits, discount behaviour, notification sender details, usage counters. Each is a **single record** — there is no meaningful second one.

Several of these paths are permanently confirm-gated, and the ones that are not still affect the whole instance.

→ `mcp/docs/api/baseline-data#settings-records-are-singletons` · `mcp/docs/server/allow-levels`

## Settings records are singletons

General settings, plan limits, usage counters, discount settings and event notification settings each exist exactly once, provisioned with the instance.

**Update them. Never create them.** A create succeeds silently, leaves two records, and the platform reads only one — which makes the change appear to do nothing and leaves a second record nobody knows about.

## General settings

Instance-level presentation and behaviour: date formatting, image handling and similar. Small in scope, wide in effect — a date format change is visible everywhere at once.

Read the record, change the one field, dry run, send, read back.

## Plan limits are not yours to set

Limits on requests, storage, record counts, admin seats and upload sizes describe what the instance is entitled to. They are managed by whoever provisions the instance, and this server keeps that whole path permanently confirm-gated.

Read them when you need to explain a limit — "creates are failing because the record limit is reached" is a genuinely useful diagnosis. Do not edit them to make a limit go away.

→ `mcp/docs/api/baseline-data#every-needless-record-consumes-instance-quota`

## The record limit and what it feels like

When an instance reaches its total record limit, **every** create fails, including ones unrelated to whatever filled the quota. The symptom is confusing precisely because it is global.

Two things follow: check the counters before concluding that a specific create is broken, and treat accidental duplicates as something to clean up rather than tolerate.

## Discount settings

Whether discounts stack is an instance setting, not a property of a discount. Any promise about what a customer will pay depends on it.

Read it before configuring discounts, and mention it when explaining a total to a human.

→ `mcp/docs/api/discounts#stacking`

## Notification sender settings

The sender identity used for outbound notifications. Changing it changes what customers see on every message the instance sends afterwards.

An address that does not pass the receiving side's checks results in messages that quietly do not arrive. Treat a change here as something to verify with a real send, not just a successful write.

## Captcha and system settings

Some system settings — captcha keys among them — are permanently confirm-gated. They involve credentials issued by an external service, and rotating them affects every form on the site at once.

Read them to confirm whether a mechanism is configured. Leave changes to an operator who has the new values in hand.

## Changing any setting

1. Read the record first, in full.
2. Change one field.
3. Dry run and show the human the current value alongside the new one.
4. Send, read back, and confirm nothing else moved.

Settings are the area where a well-meaning "tidy up" does the most damage, because a single record affects everything and nothing points at it to warn you.

## Common mistakes

- **Creating a settings record.** Silently duplicates.
- **Raising a plan limit to fix a failure.** Not yours; diagnose instead.
- **Ignoring the stacking setting** when predicting a discounted total.
- **Changing a sender address without verifying delivery.**
- **Sending a whole settings object back with one field changed** and overwriting fields you never read.
