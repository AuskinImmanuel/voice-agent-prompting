# Workflow Graph: DriveWay Motors Inbound Agent

This document defines the complete node and edge graph for the ElevenLabs Agent Workflow. Build this exactly in the Workflow editor.

---

## Node Definitions

| Node ID | Type | Name | Description |
|---|---|---|---|
| `N00` | `start` | `call_start` | Entry point. Fires on call connect. |
| `N01` | `override_agent` | `intent_router` | Listens to the caller's first statement and routes to the matching intent subagent. Additional prompt: see below. No tools. |
| `N02` | `override_agent` | `buying_subagent` | Handles buying and inventory browsing. See `subagents/01-buying.md`. |
| `N03` | `override_agent` | `selling_subagent` | Handles sell inquiries and valuation requests. See `subagents/02-selling.md`. |
| `N04` | `override_agent` | `booking_subagent` | Books test drives, hub visits, and home inspections. See `subagents/03-booking.md`. |
| `N05` | `override_agent` | `deal_status_subagent` | Retrieves deal or booking status. See `subagents/04-deal-status.md`. |
| `N06` | `override_agent` | `post_sale_subagent` | Handles post-sale document, insurance, and warranty queries. See `subagents/05-post-sale.md`. |
| `N07` | `phone_number` | `human_transfer` | Transfers to a live DriveWay agent. Set the target phone number in the node config. |
| `N08` | `end` | `call_end` | Ends the call after the closing sequence is complete. |

---

## Intent Router Node (N01) -- Additional Prompt

```
After the caller's first statement, silently determine their intent and route to the matching subagent. Never announce the routing. Do not say "I am transferring you" or label intent categories out loud.

Route when:
- Caller wants to buy, browse, or find a car -> buying_subagent
- Caller wants to sell their car, get a valuation, or find out what their car is worth -> selling_subagent
- Caller wants to book a test drive, hub visit, or home inspection -> booking_subagent
- Caller mentions a booking reference, deal ID, or wants to check an existing order -> deal_status_subagent
- Caller has a post-purchase question (documents, RC transfer, invoice, insurance, warranty, Form 35) -> post_sale_subagent

If intent is unclear after the caller's first statement, ask exactly one question:
"Are you looking to buy a car, sell one, or is this about an existing booking?"

Wait for the answer, then route.
```

---

## Edge Definitions

### From start to intent router

| From | To | Condition |
|---|---|---|
| `N00` (call_start) | `N01` (intent_router) | Always (on call connect) |

### From intent router to subagents

| From | To | Condition |
|---|---|---|
| `N01` (intent_router) | `N02` (buying_subagent) | Caller intent is buying or inventory browsing |
| `N01` (intent_router) | `N03` (selling_subagent) | Caller intent is selling or valuation |
| `N01` (intent_router) | `N04` (booking_subagent) | Caller intent is booking an appointment |
| `N01` (intent_router) | `N05` (deal_status_subagent) | Caller intent is checking deal or booking status |
| `N01` (intent_router) | `N06` (post_sale_subagent) | Caller intent is a post-sale document or support query |

### Cross-intent handoffs between subagents

These edges handle the natural flows where one intent leads to another (e.g., valuation leads to booking a home inspection, buying inquiry leads to booking a test drive).

| From | To | Condition |
|---|---|---|
| `N02` (buying_subagent) | `N04` (booking_subagent) | Caller agrees to book a test drive |
| `N03` (selling_subagent) | `N04` (booking_subagent) | Caller agrees to schedule a home inspection |
| `N05` (deal_status_subagent) | `N04` (booking_subagent) | Caller wants to reschedule an existing appointment |
| `N06` (post_sale_subagent) | `N03` (selling_subagent) | Caller asks to sell the car they bought from DriveWay |

### Escalation to human

| From | To | Condition |
|---|---|---|
| `N02` (buying_subagent) | `N07` (human_transfer) | Caller explicitly requests a human agent |
| `N03` (selling_subagent) | `N07` (human_transfer) | Caller explicitly requests a human agent |
| `N04` (booking_subagent) | `N07` (human_transfer) | Caller explicitly requests a human agent |
| `N05` (deal_status_subagent) | `N07` (human_transfer) | Caller explicitly requests a human agent |
| `N06` (post_sale_subagent) | `N07` (human_transfer) | Caller explicitly requests a human agent OR complaint requires immediate escalation |

### Call end

| From | To | Condition |
|---|---|---|
| `N02` (buying_subagent) | `N08` (call_end) | Caller confirms nothing more is needed, closing sequence complete |
| `N03` (selling_subagent) | `N08` (call_end) | Caller confirms nothing more is needed, closing sequence complete |
| `N04` (booking_subagent) | `N08` (call_end) | Caller confirms nothing more is needed, closing sequence complete |
| `N05` (deal_status_subagent) | `N08` (call_end) | Caller confirms nothing more is needed, closing sequence complete |
| `N06` (post_sale_subagent) | `N08` (call_end) | Caller confirms nothing more is needed, closing sequence complete |
| `N07` (human_transfer) | `N08` (call_end) | Transfer initiated (call leaves agent hands) |

---

## Notes

- Each `override_agent` node inherits the base agent's full system prompt. Its `additional_prompt` layers intent-specific instructions on top.
- Tool access per node is controlled by `additional_tool_ids`. The base agent has all tools registered; the workflow restricts which are active per node.
- The `phone_number` node (N07) is a system-level transfer. The agent does not speak after the transfer initiates.
- The `end` node (N08) is reached only after the agent has completed the closing sequence (summary, reference number, "anything else") and called `end_call`.
