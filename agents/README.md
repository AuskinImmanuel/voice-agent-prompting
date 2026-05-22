# Sample agents

Six voice agents, built natively for the ElevenLabs platform. Three verticals, two agents each: an inbound agent and an outbound agent.

The pattern behind the pairing:

- **Inbound is an Agent Workflow.** A caller dialing a support line can want any of several things. The workflow is a graph: a start node, one `override_agent` subagent node per intent (each with its own instructions and tools), and edges connecting them. ElevenLabs routes the caller to the right subagent node based on what they say.
- **Outbound is a single agent.** An outbound call is a campaign with one job: confirm a delivery, recover a cart, remind about a payment. One linear objective, so one prompt, no routing.

All agents are generic and non-branded. The companies are invented. Pick any folder and you have enough to build the agent in the ElevenLabs dashboard and place real calls.

## Index

### Automotive / used cars
- [Inbound, Agent Workflow](automotive-used-cars/inbound-workflow/): a used-car marketplace support line (buying, selling, booking a test drive or inspection, deal status, post-sale documents).
- [Outbound, single agent](automotive-used-cars/outbound-single/): re-engage a lead who got an online quote, move them toward a booking.

### E-commerce
- [Inbound, Agent Workflow](ecommerce/inbound-workflow/): a retail support line (order status, returns and refunds, delivery rescheduling, product questions, escalation).
- [Outbound, single agent](ecommerce/outbound-single/): cart-abandonment re-engagement.

### BFSI / fintech
- [Inbound, Agent Workflow](bfsi/inbound-workflow/): a consumer-lending support line (loan status, card support, payment and EMI queries, disputes, escalation). Identity is verified before any account-specific information is shared.
- [Outbound, single agent](bfsi/outbound-single/): an EMI / payment reminder call, respectful and compliant.

## How each folder is laid out

**Inbound (Agent Workflow):**
```
inbound-workflow/
  README.md          what the workflow does, and how to build it in ElevenLabs
  workflow.md        the node and edge graph: start node, subagent nodes, edges
  base-agent.md      the base 6-section system prompt loaded on every node
  subagents/         one file per caller intent, each an override_agent subagent node
  tools.md           tool definitions: name, description, parameters, when-to-call
```

**Outbound (single agent):**
```
outbound-single/
  README.md          what the agent does, and how to build it in ElevenLabs
  system-prompt.md   the full system prompt
  tools.md           tool definitions: name, description, parameters, when-to-call
```

For the thinking behind these, see the [writeups](../writeups/).
