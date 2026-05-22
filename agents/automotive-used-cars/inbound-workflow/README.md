# Inbound Agent Workflow: DriveWay Motors

## What This Agent Does

This is the main inbound phone line for DriveWay Motors, a fictional generic used-car marketplace. A single ElevenLabs agent built as a native Agent Workflow handles every reason a customer might call: browsing inventory, getting a sell/valuation estimate, booking a test drive or home inspection, checking an existing deal, and post-sale document queries.

The workflow routes the caller to the correct intent subagent (an `override_agent` node) based on their opening statement. The caller never hears internal routing labels.

---

## Custom Dynamic Variables

These variables are injected at conversation init via the ElevenLabs Conversations API. They are NOT auto-populated by ElevenLabs -- your backend must pass them when initiating the call.

| Variable | Type | Example | Notes |
|---|---|---|---|
| `{{business_hours}}` | string | `Monday to Saturday, 9 AM to 7 PM` | Used in the after-hours guardrail |
| `{{agent_display_name}}` | string | `Priya` | Agent persona name for the greeting |

`{{system__caller_id}}` is a system variable (auto-populated by ElevenLabs) providing the caller's phone number on inbound calls. It is used by tools that need a phone number without asking the caller.

---

## Intent Map

| Intent | Subagent Node | Primary Tools |
|---|---|---|
| Buying / browsing inventory | `buying_subagent` | `lookup_listing`, `log_callback_request` |
| Selling / valuation | `selling_subagent` | `get_valuation_estimate` |
| Book appointment | `booking_subagent` | `book_appointment`, `lookup_listing` |
| Deal or booking status | `deal_status_subagent` | `get_deal_status`, `log_callback_request` |
| Post-sale documents | `post_sale_subagent` | `get_deal_status`, `log_callback_request` |

---

## How to Build This in the ElevenLabs Workflow Editor

### Step 1: Create the base agent

1. In the ElevenLabs dashboard, open the agent you want to use as the base (or create a new one).
2. Paste the contents of `base-agent.md` into the system prompt field.
3. Set the first message to: `"Thank you for calling DriveWay Motors. This is {{agent_display_name}}. How can I help you today?"`
4. Add all five server tools from `tools.md` to this base agent (`lookup_listing`, `get_valuation_estimate`, `book_appointment`, `get_deal_status`, `log_callback_request`).
5. Add the `end_call` and `transfer_to_number` system tools.

### Step 2: Open the Workflow editor

Go to your agent, click the "Workflow" tab, and open the visual graph editor.

### Step 3: Add the start node

The workflow always begins with a `start` node. This node fires on call connect. No configuration is needed here beyond linking to the first routing step.

### Step 4: Add the intent router node

Add an `override_agent` node named `intent_router`. In its `additional_prompt`, instruct the agent to listen to the caller's opening statement and route (via the workflow edges defined in Step 7) to the matching subagent. This node does NOT carry tools -- it just listens and routes.

### Step 5: Add one override_agent node per intent (5 nodes)

For each subagent in `subagents/`:

1. Create an `override_agent` node with the name matching the file (e.g., `buying_subagent`).
2. Copy the `additional_prompt` block from the subagent file into the node's additional prompt field.
3. Add only the tools listed in `additional_tool_ids` for that node. Tools listed here are layered on top of the base agent's tool set for that node.

### Step 6: Add terminal nodes

- Add a `phone_number` node for human escalation. Set the phone number to DriveWay's live agent queue. Name it `human_transfer`.
- Add an `end` node named `call_end`.

### Step 7: Connect the edges

Wire edges exactly as defined in `workflow.md`. Each edge has a condition (the intent the router detected) and points to a target node. Within each subagent node, add edges for:
- Escalation requests (lead to `human_transfer`)
- Booking success or call close (lead to `call_end`)
- Cross-intent handoffs (e.g., selling subagent can hand to booking subagent)

### Step 8: Add custom dynamic variables

In the agent settings, declare `{{business_hours}}` and `{{agent_display_name}}` as custom dynamic variables. Your telephony layer must pass these at conversation init.

### Step 9: Test

Use the "Test Agent" panel. Simulate each intent trigger phrase and verify:
- Routing reaches the correct subagent node without announcing internal labels
- Each subagent calls only the tools in its `additional_tool_ids`
- Closing sequence fires before `end_call`
- Human escalation transfers correctly via the `phone_number` node

---

## File Structure

```
inbound-workflow/
  README.md               (this file)
  workflow.md             (node and edge graph)
  base-agent.md           (base agent system prompt, 6-section format)
  tools.md                (server tools + system tools)
  subagents/
    01-buying.md
    02-selling.md
    03-booking.md
    04-deal-status.md
    05-post-sale.md
```
