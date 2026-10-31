# How it works

Checked against OpenClaw 2026.10.4.

## The hook

The plugin registers one `before_tool_call` handler at priority 100 (`plugins/hooks/tool-policy.md`). For a call it blocks, it returns `{ block: true, blockReason }`, which is terminal: lower-priority handlers are skipped and the tool doesn't run. OpenClaw also blocks the call when a `before_tool_call` handler throws or times out (`plugins/hooks/reference.md:93`). Tool Search and Code Mode keep plugin `before_tool_call` hooks for the calls they make on the model's behalf (`tools/tool-search.md:321-323`, `tools/code-mode/internals.md:63-64`).

No conversation-access grant is needed: `before_tool_call` sees only the tool name, its parameters and run metadata.

## What counts as a webhook route

The scheduling tool is `cron`, with the alias `automations`. Its delivery schema has four ways to post to a URL (`createCronDeliverySchema` and `createCronFailureAlertSchema` in the built cron tool):

| Field | Webhook when |
| --- | --- |
| `delivery.mode` | `"webhook"` |
| `delivery.failureDestination.mode` | `"webhook"` |
| `delivery.completionDestination` | present (its only mode is `"webhook"`) |
| `failureAlert.mode` | `"webhook"` |

The schemas allow extra properties, and the tool also accepts job fields at the top level of the call. So instead of checking those four paths, the plugin scans the whole parameter tree and blocks when it finds:
- any string value equal to `webhook` (ignoring case and surrounding space);
- a non-null value under any key whose name contains `webhook`;
- a `to` that holds a URL: anything with `://`, or anything `URL.parse` reads as `http:` or `https:`, such as `https:host/path`. The host accepts that form as a webhook URL (`normalizeHttpWebhookUrl`, 2026.10.5). A dotted key ending in `to`, such as `delivery.failureDestination.to`, counts as `to`, because the host nests dotted keys on `update` (`nestDottedCronKey`, 2026.10.5).

A URL elsewhere, such as in a reminder's text, isn't a route and passes.

## Failing closed

- Parameters that aren't an object are blocked.
- An `action` outside the list the scheduling tool had in 2026.10.4 (`status`, `list`, `get`, `add`, `update`, `remove`, `run`, `runs`, `next_check`, `wake`) is blocked. A new action in a later version stays blocked until the list in `src/rules.ts` is updated.
- A call with no agent id is checked. OpenClaw documents missing context fields as unproven (`plugins/hooks/tool-policy.md`).

## What it can't see

- Jobs created outside an agent's tools: the `openclaw cron` CLI, the Control UI and Gateway RPC callers. To catch those as well, check `openclaw cron list --json --all` for webhook routes from outside the gateway.
- Existing jobs. The plugin acts when a job is created or changed, not on jobs already stored.
