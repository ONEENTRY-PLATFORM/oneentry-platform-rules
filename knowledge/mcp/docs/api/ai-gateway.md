# AI Gateway

The platform's own facility for calling language models: provider accounts that hold credentials and limits, and jobs that run asynchronously.

Do not confuse it with the MCP server you are talking to. This is a feature of the platform that an operator configures; the MCP server is how you reach the Admin API.

→ `mcp/docs/api/ui-component-configs#the-constructor-agent-and-its-result` · `mcp/docs/api/settings` · `mcp/docs/server/getting-started`

## Provider accounts

An account holds the connection to one model provider: its credential, which models are allowed, which is the default, and its limits.

Accounts are addressed by a stable marker. **No accounts are provisioned**, so an instance may have none, and the feature then does nothing rather than failing loudly.

`AdminAiSettingsController_listRouters` and `AdminAiSettingsController_getRouter` read them; `AdminAiSettingsController_getRouterUsage` reports what an account has consumed against its limits.

## Who may create change or delete an account

Writing an account needs one of three permissions, one per operation:

- `settings.aiAccounts.create` — `AdminAiSettingsController_createRouter`
- `settings.aiAccounts.update` — `AdminAiSettingsController_updateRouter`
- `settings.aiAccounts.delete` — `AdminAiSettingsController_deleteRouter`

Without it the write answers `403`. Reading accounts, usage, allowed models and the model catalog needs no permission, so any admin can see which accounts exist and pick a model.

Admins who could already change general settings were given these keys when they appeared; others must ask for them.

→ `mcp/docs/api/admins-and-permissions#asking-for-a-grant`

## Never expose an account credential

An account's configuration contains a provider API key. Two rules, the same as for payment accounts:

- Do not print account configuration into a conversation, a report, a commit or a ticket. To show an account exists, show its marker and its provider.
- Do not create or change an account unless a human supplies the credential in that moment and asks you to.

A read returns `apiKey` masked — a few characters at each end, stars between — and `hasApiKey` saying whether one is set. The masked value is not the key: never send it back.

On update, omit `apiKey` to keep the stored key, send a string to replace it, and send `null` only to delete it.

→ `mcp/docs/api/payments#never-read-or-write-credentials-casually`

## Jobs are asynchronous

Work is submitted as a job and completed later. The submission answers `201` with `jobId` and `status`, usually `queued`.

Poll `AdminAiGatewayController_getJob` until `status` is terminal — `completed`, `failed` or `cancelled` — then read `output`. `error` is set only on `failed`. Do not treat the submission response as the answer, and do not submit again because the first call returned quickly.

`AdminAiGatewayController_cancelJob` stops a job that is still running. Cancelling one that already completed or failed answers `409`.

## Submitting twice is the expensive mistake

Each job consumes tokens against the account's limits. A duplicate submission costs real money and produces a second result nobody asked for.

Agent submissions accept an optional `idempotencyKey`. A second submission with the same key returns the existing job instead of starting another, so send one whenever a retry is possible.

If a job seems stuck, read it by id. If it is still running, wait.

## Limits on an account

| Field | Bounds | Meaning |
|---|---|---|
| `tokenLimit` | integer ≥ 0, or `null` | Tokens the account may spend; `null` is no limit |
| `costLimitUsd` | number ≥ 0, or `null` | US dollars the account may spend; 10 when omitted on create, `null` is no limit |
| `agentRunTimeoutSec` | integer 1 to 2147483 | Seconds one agent run may take; 300 by default; `null` is refused with `400` |
| `agentRunCostLimitUsd` | number 0 to 999999, or `null` | US dollars one agent run may spend; `null` is no limit |

`resetTokenUsage: true` and `resetCostUsage: true` on update start the matching counter again from zero. There is no limit on the number of steps or tokens in one run apart from these.

## Refused with 429 because the account limit is used up

Once consumed tokens or cost reach `tokenLimit` or `costLimitUsd`, every new submission against that account answers `429` and no job is created. `isExhausted: true` in the usage read says the same thing in advance.

That refusal is a configuration state, not a fault. Report the account and its usage to the human; raising or resetting the limit is their decision.

## A completed job with an empty result ran out of budget

When one agent run reaches `agentRunTimeoutSec` or `agentRunCostLimitUsd`, the job still ends `completed`, not `failed`. Its `output` carries `warningCode: "exhausted"` and no usable result. The spend is counted all the same.

The cost ceiling is checked between model rounds, so a run can pass it by the cost of one round.

Tell the human the run was cut short and by which limit. Do not resubmit the same request unchanged — it will stop at the same point. Narrow the request, or ask for a higher per-run limit.

## Choosing a model

An account declares `allowedModels` and a `defaultModel`. A job uses the model in the request, otherwise the default, otherwise the first allowed one.

A model outside a non-empty allowed list is not refused at submission: the job is accepted and then ends `failed` with the reason in `error`. Read `AdminAiSettingsController_getRouterModels` before naming a model, and prefer the default — the allowed list is how an operator controls cost.

An unknown account marker behaves the same way: accepted, then `failed`.

## Configuration is an operator decision

Which provider, which models, what limits, and whether the feature is enabled at all are all operator choices with a budget attached. An agent's role here is to read the configuration and use it, not to expand it.

If a task needs a model that is not allowed, or a limit that is not there, say so and stop.

## Common mistakes

- **Confusing this with the MCP server.** Different things entirely.
- **Printing an account credential, or sending back the masked one.** Never.
- **Treating a submission as a result.** Poll the job.
- **Re-submitting a slow job.** Send an `idempotencyKey`, and read the job by id.
- **Reading `completed` as success.** Check `warningCode` first.
- **Retrying a `429`.** The account is out of budget; nothing changes until a human acts.
- **Assuming the feature is configured.** No accounts are provisioned.
