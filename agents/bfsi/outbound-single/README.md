# Outbound EMI Payment Reminder Agent

**Domain:** BFSI / Consumer Fintech
**Direction:** Outbound
**Agent name (in prompt):** Ravi
**Fictional lender:** Finbridge

---

## What This Agent Does

Ravi makes pre-due-date outbound calls to Finbridge customers who have an upcoming EMI payment. The call has one objective: remind the customer about the upcoming due and capture a clear payment intent so the servicing team knows what to expect.

The three possible outcomes of any call:

| Outcome | What the caller says | Action |
|---|---|---|
| Pay now | "I want to pay now" / "send me the link" | Send a payment link via SMS, confirm on call |
| Confirm pay-by date | "I will pay by [date]" | Log the commitment date |
| Hardship / human needed | "I cannot pay", distress signals, complex query | Transfer to human servicing agent |

Calls that go to voicemail are handled with a short, scripted message.

---

## Compliance and Tone Note

This agent calls customers before the due date. The payment is not yet overdue. The tone must be helpful and informational, never threatening or pressuring.

Key compliance posture:
- Do not mention consequences for non-payment unless the customer asks.
- Do not call or imply the customer has missed a payment.
- Do not offer payment waivers, restructuring, or hardship terms yourself. Route to a human for that.
- Identity verification must be passed before any account detail is shared, even though this is an outbound call.
- If the person who answers is not the account holder, end the call without disclosing any financial detail.

---

## How to Build This in ElevenLabs

1. **Create a new Conversational AI agent** in the ElevenLabs dashboard.
2. **System prompt:** Paste the content of `system-prompt.md` into the System Prompt field.
3. **First message:** Use a dynamic first message that injects the customer's first name via a variable. Example: "Hello, may I speak with {{customer_first_name}}?"
   - ElevenLabs supports variable injection in the first message via conversation init parameters.
4. **Tools:** Register all tools from `tools.md` as Server Tools. Match parameter names exactly.
5. **Voice:** Select a warm, professional Indian-English male voice. Set stability to 0.60, similarity to 0.75.
6. **Language model:** GPT-4o or Claude 3.5 Sonnet. This is a simpler single-flow agent; GPT-4o mini may also work.
7. **Outbound call trigger:** Use the ElevenLabs Conversations API to initiate the call, passing call metadata as custom dynamic variables at conversation init (see below).
8. **Voicemail detection:** Add the `voicemail_detection` system tool on the agent. Set its `voicemail_message` parameter to the scripted message in the `VOICEMAIL HANDLING` section of `system-prompt.md`. Do not use prompt-based voicemail detection logic.
9. **Custom dynamic variables** to pass at conversation init (required): `customer_first_name`, `due_amount`, `due_date`, `account_last_four` (to help the customer identify which account this call is about). These are not auto-populated by ElevenLabs; you must pass them via the Conversations API.

---

## Files in This Folder

| File | Purpose |
|---|---|
| `README.md` | This file |
| `system-prompt.md` | Full single-flow outbound prompt |
| `tools.md` | Tool schemas for all tools used in this agent |
