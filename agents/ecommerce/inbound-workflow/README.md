# Inbound Customer Support Agent (Agent Workflow)

Generic e-commerce inbound support line. Fictional brand: **Kart Street**.

---

## What This Agent Does

Handles the five most common inbound customer support intents for an online retailer. Using ElevenLabs Agent Workflows, the base agent classifies the caller's intent and hands off to the appropriate subagent node. Each subagent node carries its own additional instructions and tool set. A caller whose need shifts mid-call is handled naturally because all subagents share the base agent's context and the full conversation transcript.

---

## Intent Map

| Intent | Subagent File | Tools Used |
|--------|--------------|------------|
| Order status / tracking | `subagents/order-status.md` | `lookup_order` |
| Returns and refunds | `subagents/returns-refunds.md` | `check_return_eligibility`, `schedule_pickup` |
| Delivery rescheduling | `subagents/delivery-rescheduling.md` | `lookup_order`, `reschedule_delivery` |
| Product / catalog questions | `subagents/product-catalog.md` | `search_catalog` |
| Escalation to human | `subagents/escalation.md` | `transfer_to_number` (system tool) |

---

## File Structure

```
inbound-workflow/
  README.md
  workflow.md
  base-agent.md
  tools.md
  subagents/
    order-status.md
    returns-refunds.md
    delivery-rescheduling.md
    product-catalog.md
    escalation.md
```

---

## Custom Dynamic Variables

The following variables are **custom dynamic variables** injected at conversation init via the ElevenLabs Conversations API. They are not auto-populated by ElevenLabs. Your backend must pass them when creating each conversation.

| Variable | Type | Description |
|----------|------|-------------|
| `{{support_hours}}` | string | Human-readable support hours, e.g. "9 AM to 9 PM, Monday to Saturday" |
| `{{brand_name}}` | string | Brand name spoken by the agent, e.g. "Kart Street" |

If you do not use dynamic variables, hardcode these values directly in `base-agent.md`.

---

## How to Build in ElevenLabs

### 1. Create the Base Agent

- Go to **ElevenLabs Conversational AI > Agents > New Agent**
- Voice: warm, neutral, mid-paced (recommended)
- Language: English
- Paste the contents of `base-agent.md` into the **System Prompt** field
- Set the first message: `"Thank you for calling Kart Street support. This is Maya. How can I help you today?"`

### 2. Add Tools to the Base Agent

Create each server tool listed in `tools.md` under **Agent > Tools > Add Tool**. Use the parameter schemas exactly as defined.

The system tools (`transfer_to_number`, `end_call`) are native to ElevenLabs and do not need to be created manually. Enable them in the agent's tool settings.

### 3. Open the Workflow Editor

- In the agent, go to the **Workflow** tab
- Build the graph described in `workflow.md`
- For each `override_agent` node, paste the matching subagent file's `additional_prompt` into the node's prompt field and select the tool IDs listed in `additional_tool_ids`

### 4. Configure the Escalation Node

The escalation node uses a native `phone_number` node type. Enter the human support queue phone number in the node's number field. No custom tool setup is needed.

### 5. Test Calls

Run at least one test call per intent before going live. Verify:
- Correct subagent node is activated for each intent phrase
- Tools are called with correct parameters
- Agent confirms data before submitting any request
- Agent does not fabricate order details
- Escalation transfers to the configured phone number
- Closing sequence runs once, at the end of the final topic

---

## Design Notes

- One question at a time. No subagent stacks two questions in a single turn.
- Data collected in an earlier intent (order ID, item name) is available in the conversation transcript. Subagents must check the transcript before asking for information again.
- The closing sequence runs only once, at the end of the caller's final topic.
- All monetary amounts are spoken as words ("fourteen hundred and ninety-nine rupees," not "Rs 1499").
