# Tools: Inbound Consumer Lending Support Agent

---

## Tool Taxonomy

| Category | Tools |
|---|---|
| System tools | `end_call`, `transfer_to_number` |
| Server tools (webhooks) | `verify_identity`, `get_loan_status`, `get_card_details`, `get_payment_info`, `raise_dispute`, `log_payment_intent`, `log_call_outcome` |

**System tools** are built into ElevenLabs and do not require registration. Reference them by exact name.
**Server tools** are webhook-based tools you register in the ElevenLabs dashboard under Agent > Tools > Add Tool. Point each to your backend endpoint. Match parameter names exactly.

Register `verify_identity` and `log_call_outcome` on the base agent. Register the intent-specific tools on the relevant subagent nodes as listed in `additional_tool_ids` in each subagent file.

`verify_identity` must be called before any account-specific tool. Do not pass account data to any other tool until `verify_identity` returns `verified: true`.

---

## System Tools

### `end_call`

Ends the call cleanly. Call this at the end of any resolved conversation after the closing line.

No parameters.

---

### `transfer_to_number`

Transfers the caller to a PSTN number (human support queue, fraud queue, etc.).

| Parameter | Type | Required | Description |
|---|---|---|---|
| `number` | string | Yes | The destination phone number in E.164 format. |

Use this for all human escalations and fraud escalations. This is the correct tool for transferring to a human; do not use any other tool for this purpose.

---

## Server Tools

### `verify_identity`

**Description:** Verifies the caller's identity against the Finbridge customer database using two data points: the mobile number linked to the account and the date of birth on file. Returns a verified or not-verified status and a session token used by downstream tools.

**When to call:** At the start of any conversation where account-specific information will be shared. Must be called and return `verified: true` before calling any other account-level tool.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `mobile_number` | string | Yes | The mobile number the caller provides, in E.164 format (e.g., +919XXXXXXXXX). Strip spaces and formatting before passing. |
| `date_of_birth` | string | Yes | Date of birth in YYYY-MM-DD format. Convert spoken dates to this format before passing. |

**Returns:**

```json
{
  "verified": true,
  "session_token": "abc123xyz",
  "customer_name": "Anjali Mehta",
  "account_ids": ["LAcc-00123", "CCard-00456"]
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
- If `verified` is false, allow one retry. On a second failure, transfer to human via `transfer_to_number`.
- The `session_token` must be passed to all subsequent tool calls in the same conversation.
- Do not share the session token with the caller.

---

### `get_loan_status`

**Description:** Retrieves the status of a loan application or active loan account for a verified customer.

**When to call:** In the loan status subagent, after identity is verified.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `query_type` | string | Yes | One of: `"application"` (pre-disbursement) or `"account"` (active or closed loan). |
| `reference_number` | string | No | Loan application ID or loan account number if the caller provides one. |

**Returns (application):**

```json
{
  "reference_number": "APP-20240315-00892",
  "product": "Personal Loan",
  "status": "under_review",
  "status_label": "Under Review",
  "documents_pending": [],
  "last_updated": "2024-11-10"
}
```

**Returns (account):**

```json
{
  "account_id": "LAcc-00123",
  "product": "Personal Loan",
  "outstanding_principal": 145000,
  "next_emi_amount": 8500,
  "next_due_date": "2024-12-05",
  "emis_remaining": 18,
  "status": "active"
}
```

**Notes:**
- `status` values for applications: `under_review`, `documents_pending`, `approved`, `disbursed`, `rejected`, `cancelled`.
- For `documents_pending`, the `documents_pending` array contains the list of required documents.
- Never infer or guess the status. Use only what the tool returns.

---

### `get_card_details`

**Description:** Retrieves credit card information for a verified customer, including status, credit limit, statement summary, and supports card blocking.

**When to call:** In the card support subagent, after identity is verified.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `query_type` | string | Yes | One of: `"activation_status"`, `"limit"`, `"statement"`, `"card_status"`, `"block_card"`. |
| `statement_month` | string | No | Month and year in YYYY-MM format. Required only when `query_type` is `"statement"`. Defaults to the most recent cycle if omitted. |
| `card_last_four` | string | No | Last four digits of the card, if the caller provides them for disambiguation. |
| `block_reason` | string | No | Required when `query_type` is `"block_card"`. One of: `"lost"`, `"stolen"`, `"suspected_fraud"`. |

**Returns (limit):**

```json
{
  "card_id": "CCard-00456",
  "total_limit": 200000,
  "available_credit": 68500,
  "last_four": "4921"
}
```

**Returns (statement):**

```json
{
  "statement_month": "2024-10",
  "total_spends": 31450,
  "transaction_count": 12,
  "minimum_due": 3145,
  "total_due": 31450,
  "due_date": "2024-11-15",
  "statement_email_sent": false
}
```

**Returns (block_card):**

```json
{
  "success": true,
  "card_id": "CCard-00456",
  "blocked_at": "2024-11-12T14:32:00Z",
  "replacement_dispatch_eta": "2024-11-18"
}
```

**Notes:**
- Confirm caller intent explicitly before calling with `query_type: "block_card"`. This action is irreversible.
- If `statement_email_sent` is false, offer to send. Do not assume it was already sent.

---

### `get_payment_info`

**Description:** Retrieves upcoming payment details, confirms payment status, handles payment link generation, and looks up auto-debit mandate status.

**When to call:** In the payment / EMI subagent, after identity is verified.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `query_type` | string | Yes | One of: `"upcoming"`, `"status"`, `"trace"`, `"auto_debit_status"`, `"payment_link"`. |
| `product_type` | string | No | One of: `"loan"`, `"credit_card"`. Required for `"upcoming"` and `"payment_link"`. |
| `date_range` | object | No | `{ "from": "YYYY-MM-DD", "to": "YYYY-MM-DD" }`. Used for `"status"` query to find recent payments. |
| `reference_number` | string | No | Bank reference or UTR number from caller. Used for `"trace"` query. |
| `payment_amount` | number | No | Amount the caller wants to pay. Used for `"payment_link"` query. |

**Returns (upcoming):**

```json
{
  "product_type": "loan",
  "account_id": "LAcc-00123",
  "next_due_date": "2024-12-05",
  "amount_due": 8500,
  "days_until_due": 23
}
```

**Returns (payment_link):**

```json
{
  "link_sent_via_sms": true,
  "link_validity_hours": 24,
  "amount": 8500
}
```

**Notes:**
- For `"payment_link"`, the tool sends the SMS automatically. Do not read the URL aloud.
- For `"trace"`, if the transaction is not found the tool returns `found: false`. Escalate to human if trace fails.

---

### `raise_dispute`

**Description:** Looks up transactions and raises dispute tickets for unauthorised charges, incorrect amounts, merchant disputes, failed transactions, and fraud reports.

**When to call:** In the dispute / problem subagent, after identity is verified.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `action` | string | Yes | One of: `"lookup"`, `"lookup_fuzzy"`, `"create"`, `"status"`. |
| `transaction_date` | string | No | Date of transaction in YYYY-MM-DD format. Required for `"lookup"` and `"create"`. |
| `transaction_amount` | number | No | Amount in rupees. Required for `"lookup"` and `"create"`. |
| `merchant_name` | string | No | Merchant name or description if the caller knows it. |
| `dispute_reason` | string | No | Required for `"create"`. One of: `"unauthorised"`, `"incorrect_amount"`, `"merchant_dispute"`, `"failed_transaction"`, `"emi_dispute"`, `"fraud"`. |
| `caller_description` | string | No | A short plain-text summary of what the caller reported. Include for `"create"` to improve dispute quality. |
| `reference_number` | string | No | Existing dispute reference number. Required for `"status"` action. |
| `category` | string | No | Additional category tag. Use `"limit_increase_request"` when logging a credit limit increase interest from the card support subagent. |

**Returns (create):**

```json
{
  "dispute_id": "DSP-20241112-00341",
  "status": "raised",
  "expected_resolution_days": 7,
  "notification_channel": "sms_and_email"
}
```

**Returns (status):**

```json
{
  "dispute_id": "DSP-20241112-00341",
  "status": "under_investigation",
  "raised_on": "2024-11-12",
  "expected_resolution_date": "2024-11-19",
  "last_updated": "2024-11-13"
}
```

**Notes:**
- Always confirm transaction details with the caller before calling with `action: "create"`.
- For `dispute_reason: "fraud"`, escalate to the fraud queue via `transfer_to_number` after the ticket is created.
- The `lookup` action does not create a ticket. Use it to confirm the transaction exists before asking the caller to confirm a dispute.

---

### `log_payment_intent`

**Description:** Records the caller's payment intent or a special flag (e.g., hardship) against the account for downstream team action.

**When to call:** In the payment / EMI subagent before transferring a hardship caller. Also used to log "will pay by date" commitments.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | Yes | Token returned by `verify_identity`. |
| `intent` | string | Yes | One of: `"hardship_transfer"`, `"pay_by_date"`, `"payment_link_sent"`, `"mandate_renewal_requested"`. |
| `committed_date` | string | No | If `intent` is `"pay_by_date"`, the date the caller committed to, in YYYY-MM-DD format. |
| `notes` | string | No | Relevant notes from the conversation to pass to the receiving team. |

**Returns:**

```json
{
  "logged": true,
  "log_id": "LOG-00892"
}
```

---

### `log_call_outcome`

**Description:** Logs the final outcome of the call for QA and CRM purposes. Call this at the end of any call that is not a clean resolution (e.g., transfer, failed verification, unanswered query).

**When to call:** Before calling `transfer_to_number` or `end_call` in non-resolved paths.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `session_token` | string | No | Pass if available (i.e., identity was verified at some point). |
| `outcome` | string | Yes | One of: `"transferred_human"`, `"transferred_fraud"`, `"verification_failed"`, `"callback_scheduled"`, `"out_of_hours"`, `"caller_disconnected"`. |
| `scenario` | string | No | The intent the caller was in when the call ended. One of: `"loan_status"`, `"card_support"`, `"payment_emi"`, `"dispute"`, `"escalation"`. |
| `escalation_reason` | string | No | Brief plain-text reason for the escalation. One to two sentences. |
| `flags` | array | No | Special flags. Options: `"hardship"`, `"fraud"`, `"abusive_caller"`, `"transfer_failed"`. |

**Returns:**

```json
{
  "logged": true,
  "call_log_id": "CLOG-20241112-04421"
}
```
