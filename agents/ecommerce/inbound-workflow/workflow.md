# Workflow: Kart Street Inbound Support

This document defines the Agent Workflow graph for the Kart Street inbound support agent. Build this in the ElevenLabs **Workflow editor** on the agent's Workflow tab.

---

## Node Definitions

### N1 : Start

**Type:** `start`

**Description:** Entry point. The base agent greets the caller and listens for the first intent.

**First message:** "Thank you for calling Kart Street support. This is Maya. How can I help you today?"

---

### N2 : Intent Router

**Type:** `override_agent`

**Description:** The base agent listens and classifies the caller's intent. No additional prompt. This is the base agent operating normally until it routes to a subagent node.

> Note: In practice this is the base agent itself. No separate node is needed in the ElevenLabs UI if the start node feeds directly to intent edges. List it here for clarity.

---

### N3 : Order Status

**Type:** `override_agent`

**Description:** Handles order tracking and status inquiries.

**additional_prompt:** See `subagents/order-status.md`

**additional_tool_ids:** `lookup_order`

---

### N4 : Returns and Refunds

**Type:** `override_agent`

**Description:** Handles return eligibility checks and pickup scheduling.

**additional_prompt:** See `subagents/returns-refunds.md`

**additional_tool_ids:** `lookup_order`, `check_return_eligibility`, `schedule_pickup`

---

### N5 : Delivery Rescheduling

**Type:** `override_agent`

**Description:** Handles delivery date and address changes for in-transit orders.

**additional_prompt:** See `subagents/delivery-rescheduling.md`

**additional_tool_ids:** `lookup_order`, `reschedule_delivery`

---

### N6 : Product Catalog

**Type:** `override_agent`

**Description:** Handles product specification, availability, and compatibility questions.

**additional_prompt:** See `subagents/product-catalog.md`

**additional_tool_ids:** `search_catalog`

---

### N7 : Escalation

**Type:** `override_agent`

**Description:** Prepares the caller for transfer. Says a brief handoff line, then routes to N8.

**additional_prompt:** See `subagents/escalation.md`

**additional_tool_ids:** _(none : transfer is handled by N8)_

---

### N8 : Transfer to Human

**Type:** `phone_number`

**Description:** Native transfer node. Transfers the caller to the human support queue.

**Number:** _(enter your human support queue phone number here)_

**Note:** This node uses the native ElevenLabs `transfer_to_number` mechanism. No custom tool is needed.

---

### N9 : End

**Type:** `end`

**Description:** Terminates the call after the closing sequence. Reached from any subagent node after the caller confirms there is nothing else they need.

---

## Edge Definitions

| Edge | From | To | Condition |
|------|------|----|-----------|
| E1 | N1 (Start) | N3 (Order Status) | Caller says "where is my order," "track," "shipment," "status," "hasn't arrived," "when will I get" |
| E2 | N1 (Start) | N4 (Returns and Refunds) | Caller says "return," "refund," "exchange," "send back," "damaged," "wrong item," "defective," "don't want this" |
| E3 | N1 (Start) | N5 (Delivery Rescheduling) | Caller says "reschedule," "change delivery," "not home," "different date," "missed delivery," "change address," "different address" |
| E4 | N1 (Start) | N6 (Product Catalog) | Caller asks about a product's specs, variants, availability, price, or compatibility before purchase |
| E5 | N1 (Start) | N7 (Escalation) | Caller explicitly asks for a human ("speak to a person," "real agent," "your manager") or intent is out of scope |
| E6 | N3 (Order Status) | N4 (Returns and Refunds) | Caller shifts to return or refund topic mid-call |
| E7 | N3 (Order Status) | N5 (Delivery Rescheduling) | Caller shifts to rescheduling topic mid-call |
| E8 | N3 (Order Status) | N7 (Escalation) | Caller asks for human or issue is out of scope |
| E9 | N4 (Returns and Refunds) | N3 (Order Status) | Caller shifts to order status mid-call |
| E10 | N4 (Returns and Refunds) | N7 (Escalation) | Item not eligible and caller wants exception review, or caller requests human |
| E11 | N5 (Delivery Rescheduling) | N4 (Returns and Refunds) | Delivered order flagged as non-delivery, routing to returns |
| E12 | N5 (Delivery Rescheduling) | N7 (Escalation) | Caller requests human or issue is out of scope |
| E13 | N6 (Product Catalog) | N7 (Escalation) | Two consecutive failed catalog searches, or caller requests human |
| E14 | N7 (Escalation) | N8 (Transfer to Human) | Immediately after handoff line is spoken |
| E15 | N3 (Order Status) | N9 (End) | Caller confirms no further needs, closing sequence complete |
| E16 | N4 (Returns and Refunds) | N9 (End) | Caller confirms no further needs, closing sequence complete |
| E17 | N5 (Delivery Rescheduling) | N9 (End) | Caller confirms no further needs, closing sequence complete |
| E18 | N6 (Product Catalog) | N9 (End) | Caller confirms no further needs, closing sequence complete |

---

## Notes

- Cross-intent transitions (E6, E7, E9, E11) carry the full conversation transcript, so no data collection repeats.
- N8 (Transfer to Human) does not connect to N9 (End). The transfer ends the agent session.
- If the start node receives an ambiguous utterance that does not match any edge condition clearly, the base agent asks one clarifying question before routing.
