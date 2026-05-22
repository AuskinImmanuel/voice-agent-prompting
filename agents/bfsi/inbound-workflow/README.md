# Inbound Consumer Lending Support Agent (Workflow)

**Domain:** BFSI / Consumer Fintech
**Direction:** Inbound
**Agent name:** Priya
**Fictional lender:** Finbridge

---

## What This Agent Does

Priya handles inbound support calls for Finbridge's consumer lending and financial products line. Callers reach her for five categories of need:

| Intent | What the caller wants |
|---|---|
| Loan status | Track a loan application or active loan account |
| Card support | Activate a card, query credit limit, request a statement |
| Payment / EMI | Check due date, outstanding amount, payment history |
| Dispute / problem | Report an incorrect charge, failed transaction, fraud concern |
| Human escalation | Sensitive matter or anything outside scope |

---

## Identity Verification Gate

No account-specific information is shared before the caller passes a two-point identity check. The base agent collects:

1. The mobile number registered with the account
2. The date of birth on the account

Both must match records via the `verify_identity` server tool. If verification fails twice, the call is transferred to a human agent via `transfer_to_number`.

This gate is enforced in the base agent's `# Guardrails` section and applies to every intent subagent. Do not remove it.

---

## Architecture: ElevenLabs Agent Workflow

This agent is built as a native **Agent Workflow** in the ElevenLabs dashboard, not a single flat system prompt. The workflow is a graph of nodes and edges. Each intent is handled by a dedicated `override_agent` subagent node that inherits the base agent config and adds intent-specific instructions and tools.

See `workflow.md` for the full node and edge graph.
See `base-agent.md` for the base agent system prompt.
See `subagents/` for the per-intent subagent files.
See `tools.md` for all tool schemas.

---

## Custom Dynamic Variables

The following variables must be passed at conversation init via the ElevenLabs Conversations API. They are **not** auto-populated by the platform.

| Variable | Description |
|---|---|
| _(none for inbound)_ | The caller's phone number is available via the system variable `{{system__caller_id}}`. No additional custom variables are required for this inbound agent at baseline. |

If your deployment needs additional injected context (e.g., a CRM lookup result pre-filled before the call connects), you can add custom variables following the `{{snake_case}}` convention and reference them in `base-agent.md` under `# Environment`.

---

## How to Build This in the ElevenLabs Dashboard

1. **Create the base agent.** In the Conversational AI section, create a new agent. Paste the content of `base-agent.md` as the system prompt.
2. **Add tools to the base agent.** Register `verify_identity` and `log_call_outcome` as Server Tools on the base agent (see `tools.md`).
3. **Open Workflow editor.** Switch to the Workflow tab on the agent.
4. **Build the node graph** following `workflow.md`:
   - Add a `start` node.
   - Add a `tool` node for `verify_identity`.
   - Add an intent-routing branch.
   - Add five `override_agent` subagent nodes (one per intent file in `subagents/`).
   - Add a `phone_number` node pointing to your human support queue.
   - Add `end` nodes where appropriate.
5. **Configure each subagent node.** For each `override_agent` node, paste the `additional_prompt` from the corresponding file in `subagents/` and list the `additional_tool_ids` shown there.
6. **Voice:** Select a warm, calm Indian-English female voice (for example, Aria or a custom cloned voice). Set stability to 0.55, similarity to 0.75.
7. **First message:** "Thank you for calling Finbridge. This is Priya. How can I help you today?"
8. **Language model:** GPT-4o or Claude 3.5 Sonnet recommended for multi-intent routing accuracy.
9. **Turn detection:** Set sensitivity to 0.5 (default). Increase slightly if callers tend to pause mid-sentence.

---

## Files in This Folder

| File | Purpose |
|---|---|
| `README.md` | This file |
| `workflow.md` | Full node and edge graph for the Agent Workflow |
| `base-agent.md` | Base agent system prompt (6-section format) |
| `subagents/01-loan-status.md` | Override agent: loan application and account status |
| `subagents/02-card-support.md` | Override agent: card activation, limit, statement, block |
| `subagents/03-payment-emi.md` | Override agent: EMI due date, payment history, hardship |
| `subagents/04-dispute-problem.md` | Override agent: dispute filing and fraud reporting |
| `subagents/05-human-escalation.md` | Override agent: escalation to human agent |
| `tools.md` | Server tool and system tool schemas |
