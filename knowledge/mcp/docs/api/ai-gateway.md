# AI Gateway

The platform's own facility for calling language models: provider accounts that hold credentials and limits, and jobs that run asynchronously.

Do not confuse it with the MCP server you are talking to. This is a feature of the platform that an operator configures; the MCP server is how you reach the Admin API.

→ `mcp/docs/api/settings` · `mcp/docs/server/getting-started`

## Provider accounts

An account holds the connection to one model provider: its credential, which models are allowed, which is the default, and a spending limit.

Accounts are addressed by a stable marker. **No accounts are provisioned**, so an instance may have none, and the feature then does nothing rather than failing loudly.

## Never expose an account credential

An account's configuration contains a provider API key. Two rules, the same as for payment accounts:

- Do not print account configuration into a conversation, a report, a commit or a ticket. To show an account exists, show its marker and its provider.
- Do not create or change an account unless a human supplies the credential in that moment and asks you to.

→ `mcp/docs/api/payments#never-read-or-write-credentials-casually`

## Jobs are asynchronous

Work is submitted as a job and completed later. A job has a terminal state — completed, failed or cancelled — and a result you read once it reaches one.

So the shape is: submit, then poll the job by id until it is terminal, then read the result. Do not treat the submission response as the answer, and do not submit again because the first call returned quickly.

```text
cms_api_search { "query": "ai gateway" }
```

## Submitting twice is the expensive mistake

Each job consumes tokens against the account's spending limit. A duplicate submission costs real money and produces a second result nobody asked for.

If a job seems stuck, read it by id. If it is still running, wait. Cancellation exists for jobs that genuinely need stopping.

## Limits and what hitting one looks like

An account carries a spending limit and reports its usage. When the limit is reached, further jobs are refused rather than queued.

That refusal is a configuration state, not a fault. Report the account and its limit to the human; raising it is their decision.

## Choosing a model

An account declares which models it allows and which is the default. A job asking for a model outside the allowed list is refused.

Read the account before specifying a model, and prefer the default unless there is a reason not to — the allowed list is how an operator controls cost.

## Configuration is an operator decision

Which provider, which models, what limit, and whether the feature is enabled at all are all operator choices with a budget attached. An agent's role here is to read the configuration and use it, not to expand it.

If a task needs a model that is not allowed, or a limit that is not there, say so and stop.

## Common mistakes

- **Confusing this with the MCP server.** Different things entirely.
- **Printing an account credential.** Never.
- **Treating a submission as a result.** Poll the job.
- **Re-submitting a slow job.** It costs tokens and produces a duplicate.
- **Assuming the feature is configured.** No accounts are provisioned.
