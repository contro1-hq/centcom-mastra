# centcom-mastra

Human-in-the-loop approval skill for Mastra agents and workflows, adding Contro1 approval gates, escalation paths, and audit-ready logs to production AI actions.

Website: https://contro1.com

Documentation: https://contro1.com/docs/mastra-human-approval

Repository description:

Human-in-the-loop approval skill for Mastra agents and workflows, adding Contro1 approval gates, escalation paths, and audit-ready logs to production AI actions.

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
