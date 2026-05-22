# Outbound Cart Recovery Agent (Single-Flow)

Generic e-commerce outbound agent. Fictional brand: **Kart Street**.

---

## What This Agent Does

Places outbound calls to customers who left items in their cart without completing the purchase. The single goal: recover the order by offering a time-limited discount and helping the customer complete the purchase if they want to.

This is a single-flow agent. There is no intent switching. Every call follows the same spine: greet, surface the abandoned items, handle the one or two most common objections, make the offer, confirm the order or close gracefully.

---

## Call Goal

Get the caller to confirm they want to complete the purchase. Apply a discount code and confirm the order via `apply_discount_code`. If the caller is not ready, log the outcome and end cleanly.

---

## Voicemail Handling

Voicemail detection uses the native ElevenLabs `voicemail_detection` system tool. When voicemail is detected, the tool plays the configured `voicemail_message` automatically and the agent calls `log_call_outcome` with `outcome: voicemail`. No discount code is included in the voicemail message.

---

## Files

```
outbound-single/
  README.md
  system-prompt.md
  tools.md
```

---

## Custom Dynamic Variables

The following variables are **custom dynamic variables** injected at conversation init via the ElevenLabs Conversations API. They are not auto-populated by ElevenLabs. Your backend must pass them when initiating each outbound call.

| Variable | Example Value |
|----------|---------------|
| `{{customer_first_name}}` | Priya |
| `{{customer_id}}` | C-10042 |
| `{{cart_item_1}}` | Wireless Earbuds Pro |
| `{{cart_item_count}}` | 2 |
| `{{discount_code}}` | SAVE10 |
| `{{discount_value}}` | 10 |
| `{{cart_id}}` | (optional) |

These variables are referenced in the system prompt and first message. Pass them as `dynamic_variables` in the Conversations API request body when creating the conversation.

---

## How to Build in ElevenLabs

### 1. Create the Agent

- Go to **ElevenLabs Conversational AI > Agents > New Agent**
- Set agent type: **Outbound**
- Voice: warm, friendly, mid-energy (this is a recovery call, not a complaint line)
- Language: English

### 2. System Prompt

Paste the full contents of `system-prompt.md` into the **System Prompt** field.

### 3. First Message (Outbound)

ElevenLabs outbound agents speak first. Set the first message to:

```
Hi, is this {{customer_first_name}}?
```

The agent's opening after confirmation is defined in `system-prompt.md` under the OPENING section.

### 4. Voicemail Detection

Enable the `voicemail_detection` system tool on the agent. Set the `voicemail_message` parameter to:

```
Hi, this is a message for {{customer_first_name}} from Kart Street. You left some items in your cart and I wanted to reach out. We have a special offer for you on those items. Please visit kartstreetonline dot com to complete your order. Thanks, have a great day.
```

The tool handles detection and playback automatically. No prompt-based detection logic is needed.

### 5. Tools

Create each server tool listed in `tools.md` under **Agent > Tools > Add Tool**. Enable `end_call` from the system tool list.

### 6. Telephony-Layer Campaign Setup

Outbound dialing campaigns are configured at the telephony layer (your SIP provider, outbound dialer, or Twilio Programmable Voice setup), not inside ElevenLabs. ElevenLabs does not have a native outbound dialing or campaign scheduler.

Configure your telephony layer to:
- Call only during permitted hours (9 AM to 8 PM, caller's local time zone)
- Limit to two attempts per contact per abandoned cart event
- Stop calling a contact if they complete the purchase through any channel (use a webhook callback to suppress further attempts)
- Pass the required custom dynamic variables when initiating each call via the Conversations API

### 7. Test

Test at least these three call paths before going live:
1. Happy path: caller agrees, discount applied, order confirmed.
2. Objection path: caller says "I'll think about it." Agent handles the objection and closes or logs.
3. Voicemail path: `voicemail_detection` triggers, message plays, outcome logged.

---

## Design Notes

- The discount code is pre-generated per call and injected as a custom dynamic variable. The agent does not generate codes on the fly.
- The agent does not process payment. It confirms intent, applies the code, and directs the caller to complete checkout via the app or website. The cart is pre-populated on the platform side.
- If the caller raises issues outside the scope of this agent (order complaint, account problem), the agent acknowledges and offers to connect them to support via `transfer_to_number`.
