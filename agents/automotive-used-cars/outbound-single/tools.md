# Tools: DriveWay Motors Outbound Re-Engagement Agent

---

## Tool Taxonomy

| Type | Definition |
|---|---|
| **System tools** | Built-in ElevenLabs tools. Use exact names as listed. No webhook configuration needed. |
| **Server tools** | Webhook-based tools. You configure the URL, authentication, and parameters in the ElevenLabs dashboard. |
| **Client tools** | Browser or app-side tools. Not used in this agent. |

---

## System Tools

| Tool Name | Purpose | Notes |
|---|---|---|
| `end_call` | Terminates the call from the agent side. | Always called after `log_call_outcome`. |
| `voicemail_detection` | Detects voicemail pickup and delivers a pre-set message. | Configure the `voicemail_message` parameter in the agent settings. The message is delivered automatically on voicemail detection -- no prompt logic needed. |

---

## Server Tools (Webhooks)

Both tools connect to the DriveWay Motors backend API. During development, point them to mock webhook endpoints. In production, each maps to a REST API route secured with an API key passed as a request header.

---

### Tool: book_appointment

**Purpose:** Create a confirmed test drive or hub visit slot in the DriveWay scheduling system.

**When to call:** Only after the lead has verbally confirmed a specific date, time, and city. Do not call speculatively. Collect all required parameters first.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `appointment_type` | string | Yes | For this agent, always `test_drive`. |
| `caller_name` | string | Yes | Use `{{lead_name}}` from the lead profile. |
| `phone_number` | string | Yes | The phone number this outbound call was placed to. Pass as received from your telephony layer. |
| `city` | string | Yes | City the lead is in or prefers for the appointment. Collect during the booking close. |
| `preferred_date` | string | Yes | Date in `YYYY-MM-DD` format. Derive from the lead's natural language using `{{system__time}}` as the current date reference. |
| `preferred_time` | string | Yes | Slot time in `HH:MM` 24-hour format (e.g., `10:00`, `14:00`). |
| `listing_id` | string | No | If the specific listing ID is available from the CRM data passed at init, pass it. Otherwise omit. |
| `source` | string | No | Always pass `"outbound_reengagement"` to tag the booking origin. |

**Returns on success:**

```json
{
  "booking_id": "DW-1192",
  "status": "confirmed",
  "appointment_type": "test_drive",
  "date": "2024-11-23",
  "time": "10:00",
  "hub_name": "Indiranagar Hub",
  "hub_address": "45 100 Feet Road, Indiranagar, Bangalore",
  "confirmation_sms_sent": true
}
```

**Agent behavior on success:** Read the booking ID, date, time, and hub name to the lead. Confirm that an SMS has been sent. Then call `log_call_outcome` with `booked` before ending.

**Returns when slot is unavailable:**

```json
{
  "status": "slot_unavailable",
  "next_available_slots": [
    {"date": "2024-11-23", "time": "11:00"},
    {"date": "2024-11-24", "time": "10:00"}
  ]
}
```

**Agent behavior on unavailable slot:** Offer up to two alternatives from `next_available_slots`. "That slot is taken -- I have openings at [option 1] or [option 2]. Which works better?"

**On tool failure:** Do not retry more than once. Tell the lead: "I'm having a bit of trouble with the booking system right now. I'll have our team call you back to confirm the slot. Can I reach you on this number?" Call `log_call_outcome` with `booking_failed_callback_needed`. Then close.

---

### Tool: log_call_outcome

**Purpose:** Record the result of the outbound call in the CRM. This tool must be called at the end of every call, without exception.

**When to call:** At the conclusion of every call, immediately before `end_call`. Including voicemails, wrong numbers, and very short calls.

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `lead_name` | string | Yes | Use `{{lead_name}}` from the lead profile. |
| `phone_number` | string | Yes | The number the call was placed to. |
| `outcome` | string | Yes | One of the outcome codes below. |
| `booking_id` | string | Conditional | Required if `outcome` is `booked`. Pass the ID returned by `book_appointment`. |
| `callback_preference` | string | Conditional | Required if `outcome` is `callback_requested`. Free text, e.g., "tomorrow afternoon" or "weekday mornings." |
| `disqualification_reason` | string | Conditional | Required if `outcome` is `not_interested`, `wrong_segment`, or `converted_elsewhere`. Brief plain-text reason. |
| `notes` | string | No | Any additional context from the call. Max 200 characters. |

**Outcome Codes:**

| Code | Meaning |
|---|---|
| `booked` | Lead agreed to a test drive and appointment was created. |
| `callback_requested` | Lead is interested but not ready; gave a preferred callback time. |
| `not_interested` | Lead clearly declined and does not want a follow-up. |
| `wrong_segment` | Lead only wants new cars or is not relevant to DriveWay's offering. |
| `converted_elsewhere` | Lead already purchased a vehicle from a different source. |
| `price_alert_requested` | Lead is interested but the price is too high; wants to be notified if it drops. |
| `not_reached` | Call was answered but the lead was not available. |
| `voicemail_left` | Call went to voicemail and a message was delivered. |
| `wrong_number` | The number does not belong to the lead. |
| `do_not_call` | Lead explicitly requested no further calls. Must be honoured. |
| `booking_failed_callback_needed` | Booking tool failed; team needs to confirm the slot manually. |

**Returns:**

```json
{
  "log_id": "LOG-00341",
  "status": "recorded",
  "next_action": "no_follow_up"
}
```

The `next_action` field is informational for the CRM. The agent does not need to relay it to the lead.

**On tool failure:** Make one retry. If the second attempt also fails, call `end_call` anyway. Do not keep the lead on the line while retrying the log.

---

## Tool Safety Rules

1. `book_appointment` must never be called without explicit verbal confirmation of date, time, and city from the lead.
2. `log_call_outcome` must be called for every call, including voicemails, wrong numbers, and very short calls.
3. Never reveal tool names or parameter names to the lead. Do not say "I'm going to call the booking system" -- say "Let me lock that in for you."
4. Never fabricate a booking ID. Only confirm a booking if the tool returns `"status": "confirmed"`.
5. For `do_not_call` outcomes: once this is logged, call `end_call` immediately. Do not make another pitch or ask clarifying questions.
