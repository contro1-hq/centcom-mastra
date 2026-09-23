# centcom-mastra

Human-in-the-loop approval skill for Mastra agents and workflows, adding Contro1 approval gates, escalation paths, and audit-ready logs to production AI actions.

Website: https://contro1.com

Documentation: https://contro1.com/docs/mastra-human-approval

Repository description:

Human-in-the-loop approval skill for Mastra agents and workflows, adding Contro1 approval gates, escalation paths, and audit-ready logs to production AI actions.

<!-- contro1:connect:start - generated from contro1.com/docs/connect-an-agent -->
## Connect your Mastra agent to Contro1

A Mastra agent runs in your own code, so it connects with an Agent Credential: a key bound to one agent, so every call is attributed to it and nothing in a request can change which agent it is.

1. Register the agent: contro1 init --name "<name>" --framework mastra, and finish the setup link it prints (purpose and owner).
2. Connect the application account under Apps, if it is not connected yet.
3. Give the agent the Actions it needs under Access. It starts with none.
4. Create an Agent Credential for it (Settings, API keys) and store it as CONTRO1_API_KEY in your secret manager.
5. Call Actions from your tools as below. The same credential creates approval requests for work your own code does.

Full guide: [Connect an agent: every path, in full](https://contro1.com/docs/connect-an-agent)

### Run an application Action from a Mastra tool

Contro1 holds the account and makes the call, so the record is what Contro1 observed. The tool returns what the Action produced; when a person has to approve first, it waits and then returns the result. It never re-submits: a retry could send a second email.

Before writing the input, read the exact input_schema with get_action_contract, or from the Action on the Access page.

```typescript
// npm install @contro1/sdk@^1.5.0
import { createTool } from "@mastra/core/tools";
import { z } from "zod";
import { CentcomClient, ActionsApi, needsHumanResolution } from "@contro1/sdk";

const actions = new ActionsApi(new CentcomClient({ apiKey: process.env.CONTRO1_API_KEY! }));

async function runAction(action_id: string, input: Record<string, unknown>) {
  const out = await actions.invoke({
    action_id, input,
    authority_mode: "agent_principal",
    account_mode: "shared",
    idempotency_key: crypto.randomUUID(),
  });
  if (out.invocation.state === "executed") return out.result;
  if (out.invocation.state === "awaiting_approval") {
    const settled = await actions.waitForInvocation(out.invocation.invocation_id);
    if (needsHumanResolution(settled)) throw new Error("Outcome unknown; a person must check. Do not retry.");
    return actions.getResult(settled.invocation_id);
  }
  throw new Error(`Not run: ${out.invocation.state}`);
}

export const listRecentEmails = createTool({
  id: "list-recent-emails",
  description: "List the most recent emails in the team mailbox.",
  inputSchema: z.object({ max_results: z.number().int().min(1).max(50).default(5) }),
  execute: async ({ context }) => runAction("gmail.message.list", { max_results: context.max_results }),
});
```

Everything below this section covers the other half: asking a person before a step your own code runs, using the same credential.

<!-- contro1:connect:end -->

## Installation / Usage

Copy `skills/centcom-mastra.md` into the skill library used by your coding agent or implementation assistant.

Use it when adding Contro1 approval gates to Mastra agents, tools, or workflows. The first version of this repo is intentionally skill-only so teams can apply the pattern without waiting for a dedicated SDK package.

## What this skill helps with

- Creating approval requests before risky Mastra tool calls or workflow steps.
- Optional Control Map previews for high-risk role routing and quorum workflows.
- Using `external_request_id` for idempotent tool and step review.
- Using `correlation_id` to keep a Mastra run or workflow timeline together.
- Handling signed callback verification before resuming execution.
- Producing audit-ready evidence for approved, denied, timed-out, and autonomous actions.

## Security note

Production approvals must go through Contro1 APIs and signed webhooks.
