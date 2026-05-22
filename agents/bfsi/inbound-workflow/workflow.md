# Agent Workflow: Inbound Consumer Lending Support (Priya)

This document defines the full node and edge graph for the ElevenLabs Agent Workflow. Build this in the Workflow editor tab on the base agent.

---

## Overview

```
start
  |
  v
[tool: verify_identity]
  |
  |-- verified: false (attempt 1) --> [say: retry_prompt] --> [tool: verify_identity (retry)]
  |                                         |
  |                                         |-- verified: false (attempt 2) --> [phone_number: human_queue]
  |
  |-- verified: true
  |
  v
[override_agent: intent_router] (base agent, listens for intent)
  |
  |-- intent = loan_status       --> [override_agent: loan_status_subagent]       --> [end]
  |-- intent = card_support      --> [override_agent: card_support_subagent]      --> [end]
  |-- intent = payment_emi       --> [override_agent: payment_emi_subagent]       --> [end]
  |-- intent = dispute_problem   --> [override_agent: dispute_subagent]           --> [end]
  |-- intent = human_escalation  --> [phone_number: human_queue]
  |-- fraud detected (any node)  --> [phone_number: fraud_queue]
```

---

## Nodes

### Node 1: `start`

**Type:** `start`
**Description:** Entry point when the call connects. The base agent greets the caller.
**Agent behaviour:** Delivers the first message. "Thank you for calling Finbridge. This is Priya. How can I help you today?" Listens for the caller's opening statement.
**Exits to:** Node 2 (verify_identity tool node).

---

### Node 2: `verify_identity`

**Type:** `tool`
**Tool name:** `verify_identity`
**Description:** Forces the base agent to collect the caller's registered mobile number and date of birth, then call the `verify_identity` server tool.
**Agent behaviour before tool call:** "Before I pull up your account, I need to confirm a couple of details for your security. Could I have the mobile number linked to your Finbridge account?" Waits. Then: "And your date of birth?" Waits. Then calls the tool.
**Exits to:**
- Edge A: `verified == true` -- go to Node 4 (intent router)
- Edge B: `verified == false` AND attempt count == 1 -- go to Node 3 (retry say)

---

### Node 3: `say` (verification_retry_prompt)

**Type:** `say`
**Description:** Informs the caller the details did not match and prompts a second attempt.
**Content:** "The details I have do not match what we have on file. Could you try again? I will need the mobile number registered with your account and your date of birth."
**Exits to:** Node 2 (verify_identity, second attempt).
**On second failure from Node 2:** go to Node 9 (human_queue transfer).

> Implementation note: track attempt count in conversation context. After the first `verified: false` response, route through Node 3 and back to Node 2. If Node 2 returns `verified: false` a second time, route to Node 9.

---

### Node 4: `override_agent` (intent_router)

**Type:** `override_agent`
**Description:** Post-verification routing. The base agent identifies the caller's intent and routes to the appropriate subagent node.
**Additional prompt:** See `base-agent.md` `# Goal` section (routing logic is already present in the base agent). No separate additional_prompt needed on this node; the base agent handles routing naturally via conversation.
**Exits to:**
- Edge: caller intent is loan / application / "my loan" -- go to Node 5
- Edge: caller intent is card / "my card" / activate / statement -- go to Node 6
- Edge: caller intent is EMI / payment / due date / outstanding -- go to Node 7
- Edge: caller intent is dispute / wrong charge / fraud / "I did not make this" -- go to Node 8
- Edge: caller says "speak to a person" / "human" / "manager" / scope is unclear -- go to Node 9
- Edge: fraud urgency detected at any point -- go to Node 10

---

### Node 5: `override_agent` (loan_status_subagent)

**Type:** `override_agent`
**Description:** Handles loan application status and active loan account queries.
**Additional prompt and tool IDs:** See `subagents/01-loan-status.md`.
**Exits to:** Node 11 (end) on resolution. Node 9 (human_queue) on escalation.

---

### Node 6: `override_agent` (card_support_subagent)

**Type:** `override_agent`
**Description:** Handles card activation, credit limit queries, statement requests, card issue troubleshooting, and card blocking.
**Additional prompt and tool IDs:** See `subagents/02-card-support.md`.
**Exits to:** Node 11 (end) on resolution. Node 9 (human_queue) on escalation. Node 8 (dispute_subagent) if fraud is mentioned during card block.

---

### Node 7: `override_agent` (payment_emi_subagent)

**Type:** `override_agent`
**Description:** Handles upcoming payment info, payment status checks, auto-debit issues, payment link dispatch, and hardship cases.
**Additional prompt and tool IDs:** See `subagents/03-payment-emi.md`.
**Exits to:** Node 11 (end) on resolution. Node 9 (human_queue) on hardship or unresolved escalation.

---

### Node 8: `override_agent` (dispute_subagent)

**Type:** `override_agent`
**Description:** Handles transaction disputes, incorrect charges, failed transactions, and fraud / account compromise reports.
**Additional prompt and tool IDs:** See `subagents/04-dispute-problem.md`.
**Exits to:** Node 11 (end) on resolved dispute. Node 10 (fraud_queue) on fraud / account takeover. Node 9 (human_queue) on unresolved or overdue dispute.

---

### Node 9: `phone_number` (human_queue)

**Type:** `phone_number`
**Description:** Transfers the caller to the Finbridge human support queue.
**Transfer number:** _(your internal human support PSTN number)_
**Agent behaviour before transfer:** See `subagents/05-human-escalation.md` for the correct pre-transfer script.
**Warm transfer note:** Pass caller verified status, the intent node they were in, a one-line summary of the issue, and any reference numbers generated.
**Exits to:** Call leaves the agent after transfer.

---

### Node 10: `phone_number` (fraud_queue)

**Type:** `phone_number`
**Description:** Transfers the caller directly to the Finbridge fraud specialist queue.
**Transfer number:** _(your internal fraud team PSTN number)_
**Agent behaviour before transfer:** "Given the nature of this, I am connecting you to our fraud team right now. Please do not share any OTPs with anyone, including anyone claiming to be from Finbridge." Call `log_call_outcome` with `outcome: "transferred_fraud"` and `flags: ["fraud"]` before the transfer completes.
**Exits to:** Call leaves the agent after transfer.

---

### Node 11: `end`

**Type:** `end`
**Description:** Call resolution. The subagent has answered the caller's question and confirmed there is nothing else needed.
**Agent behaviour before end:** "Is there anything else I can help you with today?" If no: "Thank you for calling Finbridge. Have a good day." Then `end_call`.

---

## Edge Summary

| From | To | Condition |
|---|---|---|
| start | verify_identity | Always (first step) |
| verify_identity | intent_router | verified == true |
| verify_identity | verification_retry_prompt | verified == false, attempt 1 |
| verification_retry_prompt | verify_identity | Retry prompt delivered |
| verify_identity (attempt 2) | human_queue | verified == false |
| intent_router | loan_status_subagent | Loan / application intent |
| intent_router | card_support_subagent | Card intent |
| intent_router | payment_emi_subagent | EMI / payment intent |
| intent_router | dispute_subagent | Dispute / fraud intent |
| intent_router | human_queue | Explicit human request or out-of-scope |
| loan_status_subagent | end | Resolved |
| loan_status_subagent | human_queue | Escalation (appeal, distress, complex) |
| card_support_subagent | end | Resolved |
| card_support_subagent | human_queue | Escalation |
| card_support_subagent | dispute_subagent | Fraud mentioned during card block |
| payment_emi_subagent | end | Resolved |
| payment_emi_subagent | human_queue | Hardship or unresolved |
| dispute_subagent | end | Dispute raised and resolved |
| dispute_subagent | fraud_queue | Fraud / account takeover detected |
| dispute_subagent | human_queue | Unresolved or overdue dispute |
| human_queue | (exit) | Transfer complete |
| fraud_queue | (exit) | Transfer complete |
| end | (exit) | end_call |

---

## System Tools Used in This Workflow

| Tool | Node | Purpose |
|---|---|---|
| `end_call` | end (Node 11) | Ends the call after resolution |
| `transfer_to_number` | human_queue (Node 9) | Transfers to human support |
| `transfer_to_number` | fraud_queue (Node 10) | Transfers to fraud team |
