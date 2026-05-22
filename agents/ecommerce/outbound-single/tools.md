# Tools: Kart Street Cart Recovery Agent

---

## Tool Taxonomy

| Tool | Category | Notes |
|------|----------|-------|
| `apply_discount_code` | Server (webhook) | |
| `log_call_outcome` | Server (webhook) | |
| `voicemail_detection` | System | Native ElevenLabs tool; no setup required |
| `transfer_to_number` | System | Native ElevenLabs tool; no setup required |
| `end_call` | System | Native ElevenLabs tool; no setup required |

---

## Server Tools

### `apply_discount_code`

**Purpose:** Apply a pre-generated discount code to the customer's cart and confirm it is active.

**When to Call:**
- When the customer agrees to complete the purchase.
- When the customer says "maybe" and you are pre-applying the code for convenience.

Do not call this tool for customers who say they are not interested or for voicemail outcomes.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `customer_id` | string | Yes | Internal customer ID passed in at call init as a custom dynamic variable. |
| `discount_code` | string | Yes | The pre-generated code for this call. Use the value from `{{discount_code}}`. |
| `cart_id` | string | No | Optional cart ID if passed as a custom dynamic variable. Allows the platform to apply the code to the exact cart. |

**Sample Response Shape:**

```json
{
  "applied": true,
  "discount_code": "SAVE10",
  "discount_value_pct": 10,
  "cart_total_before": 2999,
  "cart_total_after": 2699,
  "expires_at": "2024-05-19T21:00:00Z",
  "checkout_url": "https://kartstreetonline.com/cart/checkout?token=abc123"
}
```

**Voice Rendering:**
- If `applied` is true: "Done. The discount is applied to your cart. You can complete the checkout on the app or at kartstreetonline dot com."
- Do not read out `cart_total_after` as a number unless the customer specifically asks. Direct them to the cart to see the final total.
- Do not read the `checkout_url`. It is not phone-friendly.

**On Failure:** "I wasn't able to apply the code right now. I'll give you the code directly instead: [read code character by character]. You can enter it at checkout. It's valid for twenty-four hours."

---

### `log_call_outcome`

**Purpose:** Record the outcome of the call. Mandatory for every call, regardless of result.

**When to Call:** At the end of every call, before `end_call`. Also call immediately for wrong-number situations.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `customer_id` | string | Yes | Internal customer ID from the custom dynamic variables passed at call init. |
| `outcome` | string | Yes | One of the outcome codes listed below. |
| `notes` | string | No | Short free-text note (max 200 characters). Use for unusual situations only. |

**Outcome Codes:**

| Code | Meaning |
|------|---------|
| `converted` | Customer agreed to complete the purchase and discount was applied. |
| `interested_followup` | Customer did not confirm right now but expressed interest. Code pre-applied. |
| `not_interested` | Customer explicitly declined. |
| `voicemail` | Call went to voicemail. Logged after `voicemail_detection` handles the message. |
| `wrong_number` | Person who answered is not the intended customer. |
| `wrong_contact` | Customer confirmed their identity but says they did not abandon a cart. |
| `no_answer_callback` | Customer was busy or unavailable mid-call but did not opt out. |
| `opt_out` | Customer asked to be removed from outbound calls. |
| `transferred_to_support` | Call transferred to customer support team. |
| `call_error` | Technical issue prevented normal call flow. |

**No sample response shape.** This is a fire-and-forget logging call. Treat any non-error response as success.

**On Failure:** Do not tell the customer there was a logging issue.

---

## System Tools

### `voicemail_detection`

**Type:** System tool (native ElevenLabs)

**Purpose:** Detects when the call is answered by a voicemail system and automatically plays the configured voicemail message.

**When to Use:** Enable on the agent. The tool fires automatically when a voicemail greeting is detected. No prompt-based detection logic ("detect the beep," "if you hear a recording") is needed or correct.

**Configuration:** Set the `voicemail_message` parameter in the agent's tool settings to:

```
Hi, this is a message for {{customer_first_name}} from Kart Street. You left some items in your cart and I wanted to reach out. We have a special offer for you on those items. Please visit kartstreetonline dot com to complete your order. Thanks, have a great day.
```

**After voicemail plays:** The agent calls `log_call_outcome` with `outcome: voicemail`, then calls `end_call`.

---

### `transfer_to_number`

**Type:** System tool (native ElevenLabs)

**Purpose:** Transfer the call to a live human agent when the customer raises a support issue outside the scope of this agent.

**When to Use:** When the customer reports a problem unrelated to the cart (wrong order received, account issue, delivery complaint) and agrees to be transferred.

**Before calling:** Say: "Connecting you now. Take care."

**After calling:** The session transfers to the configured support number. Log `transferred_to_support` via `log_call_outcome` before the transfer if possible.

**Setup:** Configure the destination phone number in the agent's `transfer_to_number` tool settings. No custom webhook is needed.

---

### `end_call`

**Type:** System tool (native ElevenLabs)

**Purpose:** Cleanly terminate the outbound call.

**When to Call:** Only after `log_call_outcome` has been called. The two always run in sequence: log first, then end.

**Parameters:** None. Enable in agent tool settings.

---

## Custom Dynamic Variables Reference

These variables are passed into the agent at call init via the ElevenLabs Conversations API and are available throughout the conversation.

| Variable | Type | Description |
|----------|------|-------------|
| `{{customer_first_name}}` | string | Customer's first name |
| `{{customer_id}}` | string | Internal customer ID (used in tool calls) |
| `{{cart_item_1}}` | string | Name of the primary item in the cart |
| `{{cart_item_count}}` | integer | Total number of items in the cart |
| `{{discount_code}}` | string | Pre-generated discount code for this call |
| `{{discount_value}}` | integer | Discount percentage (e.g., 10 for 10 percent off) |
| `{{cart_id}}` | string | Optional cart ID for direct code application |
