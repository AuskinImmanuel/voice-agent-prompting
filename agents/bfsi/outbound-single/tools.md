# Tool Schemas: Outbound EMI Payment Reminder Agent

---

## Tool Taxonomy

| Category | Tools |
|---|---|
| System tools | `voicemail_detection`, `end_call`, `transfer_to_number` |
| Server tools (webhooks) | `verify_identity`, `send_payment_link`, `log_payment_intent`, `log_call_outcome` |

**System tools** are built into ElevenLabs and do not require registration. Reference them by exact name.
**Server tools** are webhook-based tools you register in the ElevenLabs dashboard under Agent > Tools > Add Tool. Match parameter names exactly.

For this outbound agent, the customer's account details (account ID, registered mobile) are passed as custom dynamic variables at conversation init via the ElevenLabs Conversations API. Tools use these variables alongside any caller-provided inputs.

---

## System Tools

### `voicemail_detection`

Detects answering machines and voicemail greetings. Configure a `voicemail_message` parameter with the scripted message from the `VOICEMAIL HANDLING` section in `system-prompt.md`. The tool delivers the message automatically and signals the agent to end the call.

Do not use prompt-based voicemail detection logic. Use this tool exclusively.

---

### `end_call`

Ends the call cleanly. No parameters.

---

### `transfer_to_number`

Transfers the caller to a PSTN number.

| Parameter | Type | Required | Description |
|---|---|---|---|
| `number` | string | Yes | Destination phone number in E.164 format. |

---

## Server Tools

---

## Tool: `verify_identity`

**Description:** Confirms that the person speaking is the Finbridge account holder by matching their date of birth against the account record. On outbound calls, the registered mobile number is already known from the custom dynamic variables passed at conversation init; only the date of birth is collected from the caller.

**When to call:** Immediately after the account holder confirms they are on the line, before sharing any account-specific detail (due amount, account reference, etc.).

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `account_id` | string | Yes | The account ID passed at conversation init as a custom dynamic variable. |
| `date_of_birth` | string | Yes | Date of birth as provided by the caller, in YYYY-MM-DD format. Convert spoken dates before passing. |

**Returns:**

```json
{
  "verified": true,
  "session_token": "xyz789abc",
  "customer_first_name": "Karan"
}
```

Or on failure:

```json
{
  "verified": false,
  "reason": "dob_mismatch"
}
```

**Notes:**
- On `verified: false`, do not retry on an outbound call. End the call and log the outcome. One attempt only.
- The `session_token` is used by all subsequent tools in this session.
- Do not share the session token with the caller.

---

## Tool: `send_payment_link`

**Description:** Generates a secure one-time payment link and sends it to the customer's registered mobile number via SMS. The link pre-fills the account and due amount so the customer only needs to complete authentication and confirm.

**When to call:** When the customer expresses intent to pay now and asks for a link.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `account_id` | string | Yes | Account ID passed at conversation init as a custom dynamic variable. |
| `amount` | number | Yes | Amount to pre-fill in the payment link, in rupees. Use the `due_amount` call variable. |
| `validity_hours` | number | No | How long the link is valid. Defaults to 24 hours if not specified. Pass 24 unless the system requires a different value. |

**Returns:**

```json
{
  "sms_sent": true,
  "link_validity_hours": 24,
  "masked_mobile": "XXXXXX9812"
}
```

Or on failure:

```json
{
  "sms_sent": false,
  "reason": "mobile_not_reachable"
}
```

**Notes:**
- If `sms_sent` is false, tell the customer to call the support line for an alternate way to pay. Do not retry on the same call.
- Do not read the payment link URL aloud. The SMS handles delivery.
- The `masked_mobile` can be used to confirm delivery: "The link has been sent to your number ending in [last four]."

---

## Tool: `log_payment_intent`

**Description:** Records the customer's payment intent or a special flag against their account. This is the primary data output of every completed outbound call. The servicing team uses these logs to track expected collections and follow-up cases.

**When to call:** At the conclusion of any intent-bearing conversation path (pay now, pay by date, hardship, do-not-call, disputed amount).

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. Required if verification was passed. |
| `account_id` | string | Yes | Account ID passed at conversation init as a custom dynamic variable. Pass even if verification was not passed, for unverified-caller logs. |
| `intent` | string | Yes | One of: `"payment_link_sent"`, `"pay_by_date"`, `"hardship_transfer"`, `"do_not_call_requested"`, `"payment_disputed_on_outbound"`, `"acknowledged_will_pay"`. |
| `committed_date` | string | No | Required when `intent` is `"pay_by_date"`. The date the customer committed to, in YYYY-MM-DD format. |
| `notes` | string | No | Optional plain-text context from the conversation. One to two sentences max. Used by human agents who follow up. |

**Returns:**

```json
{
  "logged": true,
  "log_id": "OLOG-20241112-00872"
}
```

**Notes:**
- Always call this before ending the call (except for third-party answered calls where no account detail was shared).
- For `intent: "hardship_transfer"`, call this before transferring, not after. The receiving agent will then have the flag visible on pickup.
- For `intent: "do_not_call_requested"`, the backend will suppress future outbound calls to this customer. Confirm to the caller that the preference has been noted, but do not promise a specific suppression timeline.

---

## Tool: `log_call_outcome`

**Description:** Logs the final disposition of the outbound call for CRM and compliance records. This is separate from payment intent logging and captures the call-level result.

**When to call:** At the end of every call, including calls that did not reach the account holder.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `account_id` | string | Yes | Account ID passed at conversation init as a custom dynamic variable. |
| `session_token` | string | No | Pass if `verify_identity` was called and passed. |
| `outcome` | string | Yes | One of: `"payment_link_sent"`, `"committed_pay_by_date"`, `"hardship_transferred"`, `"human_transfer"`, `"verification_failed"`, `"voicemail_left"`, `"third_party_answered"`, `"do_not_call_requested"`, `"customer_acknowledged"`, `"no_answer"`. |
| `call_duration_secs` | number | No | Approximate call duration in seconds. Read from `{{system__call_duration_secs}}` if available. |
| `flags` | array | No | Special flags. Options: `"hardship"`, `"disputed_amount"`, `"do_not_call"`, `"verification_failed"`. |

**Returns:**

```json
{
  "logged": true,
  "call_log_id": "CL-OUT-20241112-04871"
}
```

**Notes:**
- This tool should be the last tool called before ending the call.
- For voicemail: set `outcome: "voicemail_left"`.
- For third-party answered (not the account holder): set `outcome: "third_party_answered"`. No payment intent log is needed in this case.
- `outcome` values that also have a payment intent logged: `"payment_link_sent"`, `"committed_pay_by_date"`, `"hardship_transferred"`. Both tools should be called; this tool is for the call-level record, `log_payment_intent` is for the account-level record.
