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
- Use Control Map when high-risk approvals need a preview for required roles, quorum, separation of duties, and fallback routing.
- Verify signed Contro1 callbacks before resuming a suspended workflow or executing a delayed action.
- Treat rejected, cancelled, timed_out, invalid signatures, and unknown request IDs as fail-closed for production actions.

## Required configuration

```bash
CENTCOM_API_KEY=cc_live_your_key
CENTCOM_BASE_URL=https://api.contro1.com/api/centcom/v1
CENTCOM_WEBHOOK_SECRET=whsec_your_secret
CENTCOM_CALLBACK_URL=https://your-app.example.com/webhooks/contro1
```

## 1. Short example: ask for approval

Put this at the top of a risky Mastra tool before the real action runs:

```typescript
const { reason, ...toolInputWithoutReason } = toolInput as { reason?: string; [key: string]: unknown };

const request = await createContro1Request({
  type: 'approval',
  question: `Approve ${toolId}?`,
  context: {
    action: { tool: toolId, input: toolInputWithoutReason },
    machine_observed: { tool_id: toolId, run_id: runId },
    agent_reported: { justification: reason },
  },
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

## Send context the reviewer can trust

Build the approval `context` inside the Mastra tool, at the point where it receives `toolInput` - not by asking the agent to explain itself in a later step. Three sources feed it: the exact `toolInput` the tool was called with, copied verbatim as a machine-observed fact; the run or trigger that started the agent (the `runId`, or whatever kicked off the request); and the agent's own justification, which is only trustworthy if it rides through the gate with the call itself. Make `reason` a required field on a risky tool's input schema so the agent must produce it as part of the same call Contro1 is reviewing, instead of the tool asking "why" after the fact.

Keep provenance separated inside `context`: verbatim tool input and run facts under `machine_observed`, the model-authored `reason` under `agent_reported`. Two rules follow: `agent_reported` text must never change `required_role`, routing, or which `approval_policy` applies - it only gives the human reviewer color, since a prompt-injected agent can produce a very persuasive justification for a bad tool call. And if a high-risk tool call arrives without its required `reason`, fail closed (reject the call) instead of forwarding a guess to the operator. See https://contro1.com/docs/requests-api for the full pattern.

## 2. Full production example

A production tool should create the request, wait for a signed decision, and then perform the action:

```typescript
async function guardedDeploy(context: { version: string; environment: string; reason: string }) {
  const runId = process.env.MASTRA_RUN_ID ?? crypto.randomUUID();
  const toolId = 'deploy-release';

  if (!context.reason) {
    // Fail closed: a deploy is high-risk, so a missing agent-reported reason is
    // rejected instead of being sent to a human to guess.
    throw new Error('guardedDeploy requires a `reason` explaining why this deploy is needed');
  }

  const request = await createContro1Request({
    type: 'approval',
    question: `Deploy ${context.version} to ${context.environment}?`,
    context: {
      action: { tool: toolId, input: { version: context.version, environment: context.environment } },
      machine_observed: { tool_id: toolId, run_id: runId },
      agent_reported: { justification: context.reason },
    },
    callback_url: process.env.CENTCOM_CALLBACK_URL,
    required_role: 'developer',
    external_request_id: `mastra:${runId}:${toolId}:${context.version}`,
    correlation_id: runId,
    metadata: { integration: 'mastra', tool_id: toolId },
  });

  const decision = await waitForSignedContro1Decision(request.id);
  if (!decision.response?.approved) throw new Error('Deploy rejected by operator');

  const result = await deploy(context.version, context.environment);

  return result;
}
```

## 3. If routing fails, check who is available

Most integrations do not need Control Map in the normal approval path. If a request cannot be routed, times out unexpectedly, or your app wants to show a clear operational error, call Control Map to see who is currently available.

```typescript
const preview = await fetch(`${process.env.CENTCOM_BASE_URL}/requests/control-map`, {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${process.env.CENTCOM_API_KEY}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({
    approval_requirements: {
      required_roles: ['developer'],
      required_approvals: 1,
    },
    metadata: {
      integration: 'mastra',
      workflow_id: workflowId,
      step_id: stepId,
    },
  }),
}).then((r) => r.json());

console.log(preview.satisfiable);        // can this request be routed?
console.log(preview.on_shift_capacity);  // who is currently available?
console.log(preview.fallback_reviewers); // who can receive fallback routing?
console.log(preview.warnings);           // why routing may fail
```

## 4. Workflow approval pattern

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

## 5. Log autonomous and post-approval actions

Every approval request already stores the human decision in Contro1. Use audit records for low-risk actions that did not need approval, and optionally after an approved action completes if you want to record what your Mastra system actually did next.

```typescript
await logContro1Action({
  action: 'mastra.search_completed',
  summary: 'Searched internal docs for refund policy',
  source: { integration: 'mastra', workflow_id: 'support-agent', run_id },
  outcome: 'success',
  severity: 'info',
  correlation_id: runId,
});
```

For a post-approval action result, link the audit record back to the approval request:

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

## 6. Get evidence

Use evidence when compliance or incident review needs proof of what happened.

```typescript
const evidence = await getContro1(`/requests/${requestId}/evidence`);
const timeline = await getContro1(`/cases/${runId}`);
```

- Request evidence shows one reviewed action: context, policy, reviewer, decision, callback, timestamps, and final response.
- Case timeline shows all approvals and audit records that share the same `correlation_id`.

## Reference links

- Website: https://contro1.com
- Documentation: https://contro1.com/docs/mastra-human-approval
- Repo: https://github.com/contro1-hq/centcom-mastra
- Mastra docs: https://mastra.ai/docs
- Contro1 webhooks: https://contro1.com/docs/webhooks
