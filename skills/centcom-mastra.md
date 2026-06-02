---
name: centcom-mastra
description: Add Contro1 human approval gates to Mastra agents, tools, and workflows with signed callbacks and audit-ready logs.
user_invocable: true
---

# Contro1 + Mastra Skill

Use this skill when integrating Contro1 approval workflows into a Mastra codebase.

## Rules

- Put approval inside a Mastra tool when the agent chooses whether to call that tool.
- Put approval in a workflow step when the risky action is deterministic and known in advance.
- Keep low-risk tools autonomous; log them with Contro1 audit records when evidence is required.
- Use the Mastra run ID or workflow run ID as `correlation_id`.
- Use the exact tool ID or step ID in `external_request_id` so retries are idempotent.
- Verify signed Contro1 callbacks before resuming a suspended workflow or executing a delayed action.
- Treat rejected, cancelled, timed_out, invalid signatures, and unknown request IDs as fail-closed for production actions.

## Required configuration

```bash
CENTCOM_API_KEY=cc_live_your_key
CENTCOM_BASE_URL=https://api.contro1.com/api/centcom/v1
CENTCOM_WEBHOOK_SECRET=whsec_your_secret
CENTCOM_CALLBACK_URL=https://your-app.example.com/webhooks/contro1
```

## Tool approval pattern

Wrap destructive Mastra tools before they execute:

```typescript
const request = await createContro1Request({
  type: 'approval',
  question: `Approve ${toolId}?`,
  context: toolInput,
  callback_url: process.env.CENTCOM_CALLBACK_URL,
  required_role: 'manager',
  external_request_id: `mastra:${runId}:${toolId}`,
  correlation_id: runId,
  metadata: {
    integration: 'mastra',
    tool_id: toolId,
  },
});
```

Resume the action only after a verified approved decision. Denials and timeouts stop the action.

## Workflow approval pattern

For deterministic workflows, insert an approval step between prepare and execute:

```typescript
const approvalPayload = {
  type: 'approval',
  question: 'Approve production CRM write-back?',
  context: reviewContext,
  callback_url: process.env.CENTCOM_CALLBACK_URL,
  required_role: 'manager',
  external_request_id: `mastra:${workflowRunId}:crm-write`,
  correlation_id: workflowRunId,
  metadata: {
    integration: 'mastra',
    workflow_id: 'lead-enrichment',
    step_id: 'crm-write',
  },
};
```

## Audit logging

After an approved action runs, log the follow-up in the same case:

```typescript
await logContro1Action({
  action: 'mastra.tool_completed',
  summary: `Completed ${toolId} after approval`,
  source: { integration: 'mastra', workflow_id: workflowId, run_id: runId },
  outcome: 'success',
  correlation_id: runId,
  in_reply_to: { type: 'request', id: requestId },
});
```

## Reference links

- Website: https://contro1.com
- Documentation: https://contro1.com/docs/mastra-human-approval
- Repo: https://github.com/contro1-hq/centcom-mastra
- Mastra docs: https://mastra.ai/docs
- Contro1 webhooks: https://contro1.com/docs/webhooks
