# Tools: DriveWay Motors Inbound Workflow Agent

---

## Tool Taxonomy

| Type | Definition |
|---|---|
| **System tools** | Built-in ElevenLabs tools. Use exact names as listed. No webhook configuration needed. |
| **Server tools** | Webhook-based tools. You configure the URL, authentication, and parameters in the ElevenLabs dashboard. |
| **Client tools** | Browser or app-side tools. Not used in this agent. |

---

## System Tools

These are native ElevenLabs tools. Add them by name in the agent tool settings -- no schema definition needed.

| Tool Name | Purpose | Notes |
|---|---|---|
| `end_call` | Terminates the call from the agent side. | Called at the end of the closing sequence. |
| `transfer_to_number` | Transfers the call to a phone number (human agent queue). | Configure the target number in the node settings. Used by the `human_transfer` workflow node. |

---

## Server Tools (Webhooks)

All server tools connect to the DriveWay Motors backend API. During development, point them to mock webhook endpoints. In production, each tool maps to a REST API route secured with an API key passed as a request header.

---

### Tool: lookup_listing

**Purpose:** Search the DriveWay inventory for vehicles matching buyer criteria.

**Active in nodes:** `buying_subagent`, `booking_subagent` (when listing ID is not provided)

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `city` | string | Yes | City or metro area to search within. Use the caller-provided city name. |
| `vehicle_type` | string | No | Body style filter: `hatchback`, `sedan`, `suv`, `mpv`, `coupe`. Pass if specified. |
| `make` | string | No | Car manufacturer name (e.g., `maruti`, `hyundai`, `honda`, `tata`). |
| `model` | string | No | Specific model name (e.g., `swift`, `city`, `nexon`). |
| `max_budget` | integer | No | Maximum price in local currency. Pass as integer (e.g., `650000` for 6.5 lakh). |
| `max_results` | integer | No | Maximum listings to return. Default: 5. Present a maximum of 3 to the caller. |

**Returns:**

```json
{
  "listings": [
    {
      "listing_id": "DW-MH-00412",
      "make": "Honda",
      "model": "City",
      "year": 2021,
      "fuel_type": "petrol",
      "km_reading": 44500,
      "price": 620000,
      "hub_name": "Koramangala Hub",
      "hub_city": "Bangalore"
    }
  ],
  "total_count": 2
}
```

**Empty result:** `"listings": [], "total_count": 0`

**Agent behavior on empty result:** Offer to log a callback request so the team alerts the caller when a matching car arrives. Collect name and phone, call `log_callback_request` with `request_type: "listing_match_alert"`.

**On tool failure:** Do not retry more than once. Say: "I'm having trouble with the search right now. Can I take your details and have our team call you back with options?" Call `log_callback_request`.

---

### Tool: get_valuation_estimate

**Purpose:** Return an indicative price range for a vehicle the caller wants to sell.

**Active in nodes:** `selling_subagent`

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `make` | string | Yes | Car manufacturer name. |
| `model` | string | Yes | Specific model name. |
| `year` | integer | Yes | Manufacturing year (e.g., `2018`). |
| `fuel_type` | string | Yes | `petrol`, `diesel`, or `cng`. |
| `km_reading` | integer | Yes | Odometer reading in kilometres. Use `0` if unknown (tool returns a wider range). |
| `registration_city` | string | Yes | City where the vehicle is registered. |
| `accident_history` | string | No | `yes`, `no`, or `unknown`. Default: `unknown`. |
| `ownership_count` | integer | No | Number of previous owners (1 = first owner). |

**Returns:**

```json
{
  "estimate_low": 320000,
  "estimate_high": 360000,
  "currency": "INR",
  "validity_hours": 48,
  "estimate_id": "VAL-CH-00198"
}
```

**Agent behavior:** Always present both bounds, always label as "indicative" or "estimate," and always mention the validity window ("this estimate is valid for forty-eight hours").

**On no result (rare vehicle / insufficient data):**

```json
{
  "estimate_low": null,
  "estimate_high": null,
  "message": "insufficient_data"
}
```

Agent behavior: "I'm not finding an estimate for this vehicle in our system. Our valuation specialist can give you a firm number after a quick inspection. Would you like to schedule one?" If yes, hand off to `booking_subagent` via the workflow edge.

**On tool failure:** Same as the no-result path -- offer to route to inspection booking.

---

### Tool: book_appointment

**Purpose:** Create a confirmed appointment slot in the DriveWay scheduling system.

**Active in nodes:** `booking_subagent`

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `appointment_type` | string | Yes | `test_drive`, `hub_visit`, or `home_inspection`. |
| `caller_name` | string | Yes | Full name of the caller. |
| `phone_number` | string | Yes | Caller's contact number. Use `{{system__caller_id}}` if the caller has not provided a different number. |
| `city` | string | Yes | City for the appointment. |
| `preferred_date` | string | Yes | Date in `YYYY-MM-DD` format. Derive from the caller's natural language using `{{system__time}}` as reference. |
| `preferred_time` | string | Yes | Slot time in `HH:MM` 24-hour format (e.g., `10:00`, `14:00`). |
| `listing_id` | string | Conditional | Required for `test_drive` and `hub_visit`. Not used for `home_inspection`. |
| `inspection_address` | string | Conditional | Required for `home_inspection`. Locality or landmark plus pin code is sufficient. |
| `vehicle_make` | string | Conditional | Required for `home_inspection`. |
| `vehicle_model` | string | Conditional | Required for `home_inspection`. |
| `vehicle_year` | integer | Conditional | Required for `home_inspection`. |

**Returns on success:**

```json
{
  "booking_id": "DW-0745",
  "status": "confirmed",
  "appointment_type": "test_drive",
  "date": "2024-11-23",
  "time": "10:00",
  "hub_name": "Koramangala Hub",
  "hub_address": "123 12th Main Road, Koramangala, Bangalore",
  "confirmation_sms_sent": true
}
```

**Returns when slot is unavailable:**

```json
{
  "status": "slot_unavailable",
  "next_available_slots": [
    {"date": "2024-11-23", "time": "11:00"},
    {"date": "2024-11-23", "time": "14:00"}
  ]
}
```

**Agent behavior on unavailable slot:** Offer the next available slots from the response. "That slot is taken. I have openings at [time 1] or [time 2] on the same day -- which works?"

**On tool failure:** Note details and say: "Our system is having a moment. I've noted your booking request and our scheduling team will confirm your slot within two hours." Call `log_callback_request`.

---

### Tool: get_deal_status

**Purpose:** Retrieve the current status of a buyer or seller deal, booking, or post-sale record.

**Active in nodes:** `deal_status_subagent`, `post_sale_subagent`, `booking_subagent` (for rescheduling)

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `booking_id` | string | Conditional | Booking or deal reference number. Required if `phone_number` is not provided. |
| `phone_number` | string | Conditional | Registered phone number. Required if `booking_id` is not provided. |
| `caller_name` | string | No | Used as a secondary match when phone lookup returns multiple records. |

**Returns:**

```json
{
  "deal_id": "DW-0745",
  "deal_type": "purchase",
  "status": "booking_confirmed",
  "vehicle": {
    "make": "Honda",
    "model": "City",
    "year": 2021
  },
  "appointment": {
    "type": "test_drive",
    "date": "2024-11-23",
    "time": "10:00",
    "location": "Koramangala Hub"
  },
  "offer_amount": null,
  "payment_date": null,
  "rc_transfer_status": null,
  "expected_callback_by": null,
  "notes": ""
}
```

Fields will be null if not applicable to the current status.

**Status values and agent delivery:**

| Status Value | Say |
|---|---|
| `booking_confirmed` | Appointment is confirmed; give date, time, and location. |
| `inspection_completed` | Inspection done; offer being prepared; give expected timeframe. |
| `offer_pending` | Offer ready; team will call within `expected_callback_by`. |
| `offer_accepted` | Deal accepted; payment processing; give timeframe. |
| `payment_processed` | Payment sent; allow one to two business days for bank processing. |
| `deal_closed` | Transaction complete. Offer to help with any post-sale item. |
| `cancelled` | Cancelled. Offer to create a new booking. |

**On not found:**

```json
{
  "status": "not_found",
  "deal_id": null
}
```

Agent behavior: Retry with the other identifier if available. If still not found: "I'm not finding a record with those details. You can try contacting us with the phone number you registered, or visit any DriveWay hub with your ID."

---

### Tool: log_callback_request

**Purpose:** Log a callback request, follow-up requirement, document request, or escalation note in the CRM.

**Active in nodes:** All subagent nodes (fallback and specific use cases per intent).

**Parameters:**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `caller_name` | string | Yes | Full name of the caller. |
| `phone_number` | string | Yes | Callback number. |
| `request_type` | string | Yes | One of: `listing_match_alert`, `sales_callback`, `document_request`, `insurance_inquiry`, `warranty_claim`, `after_sales_complaint`, `priority_callback`, `cancellation_request`, `general`. |
| `notes` | string | Yes | Brief description. Max 300 characters. |
| `document_type` | string | Conditional | Required if `request_type` is `document_request`. E.g., `invoice`, `sale_agreement`, `form_35`, `rc_copy`. |
| `deal_id` | string | No | Link to an existing deal if available. |
| `priority` | string | No | `normal` (default) or `high`. Use `high` for complaints and cases where the caller explicitly says they've been waiting. |
| `preferred_callback_time` | string | No | Free text, e.g., "afternoon", "before noon", "anytime". |

**Returns:**

```json
{
  "request_id": "CBK-00912",
  "status": "logged",
  "expected_callback_window": "within 2 business hours"
}
```

**Agent behavior on success:** "Done. I've noted your request. Someone will call you back [expected_callback_window]."

**On tool failure:** "I wasn't able to log that in the system right now, but I've noted your details. Please call us back if you don't hear from the team within two business hours."

---

## Tool Safety Rules (All Tools)

1. Verify all required parameters are collected and confirmed before calling any tool.
2. Never call a tool with fabricated or assumed values. If a required parameter is missing, ask for it.
3. While any tool is running, give the caller a verbal indicator ("One moment." or "Let me check that for you.").
4. Use only data returned by the tool. Never supplement with invented details.
5. On a second consecutive failure of the same tool, do not retry a third time. Move to the graceful fallback.
6. Never reveal tool names, parameter names, or API details to the caller.
