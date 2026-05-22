# Tools: Kart Street Inbound Support Agent

---

## Tool Taxonomy

| Tool | Category | Notes |
|------|----------|-------|
| `lookup_order` | Server (webhook) | |
| `check_return_eligibility` | Server (webhook) | |
| `schedule_pickup` | Server (webhook) | |
| `reschedule_delivery` | Server (webhook) | |
| `search_catalog` | Server (webhook) | |
| `transfer_to_number` | System | Native ElevenLabs tool; no setup required |
| `end_call` | System | Native ElevenLabs tool; no setup required |

---

## Server Tools

### `lookup_order`

**Purpose:** Retrieve order details including current status, courier tracking number, estimated delivery date, and line items.

**When to Call:** When the caller wants to check order status or track a shipment. Also called by the delivery rescheduling subagent to confirm the order's current status before attempting a reschedule.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | Yes | The order ID as spoken by the caller. Strip spaces and hyphens before sending. Example: "KS00742" |
| `phone_last4` | string | No | Last four digits of the phone number on the account. Use for disambiguation if the caller does not have their order ID. |

**Sample Response Shape:**

```json
{
  "order_id": "KS00742",
  "status": "in_transit",
  "courier": "BlueDart",
  "tracking_number": "BD7839201",
  "estimated_delivery": "2024-05-18",
  "items": [
    { "name": "Wireless Earbuds Pro", "qty": 1, "price": 1499 }
  ],
  "delivery_address": "14 Rajpath Colony, Bangalore 560001"
}
```

**Voice Rendering:** Map status codes to spoken language:
- `in_transit` -> "on its way to you"
- `out_for_delivery` -> "out for delivery today"
- `delivered` -> "marked as delivered"
- `processing` -> "still being packed and prepared"
- `cancelled` -> "cancelled"

**On first failure:** "Let me try looking that up again."
**On second failure:** "I'm not able to pull up that order right now. Can I log a callback request for you?"

---

### `check_return_eligibility`

**Purpose:** Check whether a specific order line item is eligible for return and return the allowed resolution types (refund, exchange, or neither).

**When to Call:** When a caller wants to return or exchange an item, before collecting any pickup details.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | Yes | Order ID |
| `item_name` | string | Yes | Name of the item as returned by `lookup_order`. Use the exact string from the order. |
| `reason` | string | Yes | Caller's stated reason. Use one of: `wrong_item`, `damaged`, `defective`, `not_needed`, `size_issue`, `other` |

**Sample Response Shape:**

```json
{
  "eligible": true,
  "window_closes": "2024-05-25",
  "allowed_resolutions": ["refund", "exchange"],
  "pickup_required": true,
  "notes": "Item must be unused and in original packaging."
}
```

**Business Rules:**
- If `eligible` is false, tell the caller why (use the `notes` field) and offer to escalate.
- If `pickup_required` is true, proceed to `schedule_pickup` after the caller confirms their resolution choice.
- Never promise a refund timeline beyond "three to seven business days after the item is picked up."

**On failure:** "I'm having trouble checking that. I'll flag this order for our returns team and they will call you back within twenty-four hours."

---

### `schedule_pickup`

**Purpose:** Schedule a reverse-logistics pickup for a return item.

**When to Call:** After `check_return_eligibility` confirms eligibility and the caller has chosen a resolution (refund or exchange).

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | Yes | Order ID |
| `item_name` | string | Yes | Item being returned |
| `resolution` | string | Yes | Either `refund` or `exchange` |
| `pickup_date` | string | Yes | Caller's preferred pickup date. Format: YYYY-MM-DD |
| `pickup_address` | string | Yes | Full pickup address. Confirm with caller before submitting. |
| `contact_phone` | string | Yes | Contact number for the pickup team. Confirm with caller. |

**Sample Response Shape:**

```json
{
  "pickup_confirmed": true,
  "pickup_date": "2024-05-20",
  "pickup_slot": "10 AM - 1 PM",
  "request_id": "RTN-83921",
  "instructions": "Keep item in original box. Hand to courier agent."
}
```

**Voice Rendering:**
- Read the pickup date and slot: "Your pickup is scheduled for the twentieth of May, between ten AM and one PM."
- Read the request ID character by character: "Your return request number is R T N dash eight three nine two one."
- Read any instructions from the tool response.

**On failure:** "I wasn't able to book the pickup right now. I'm logging this as a priority and our team will confirm your pickup slot within two hours via SMS."

---

### `reschedule_delivery`

**Purpose:** Change the delivery date or delivery address for an order that is still in transit.

**When to Call:** When a caller wants to change their delivery date or update the delivery address.

**Pre-call check:** Confirm from `lookup_order` that status is `in_transit` or `processing`. If the status is `out_for_delivery` or `delivered`, do not call this tool.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `order_id` | string | Yes | Order ID |
| `new_date` | string | No | New preferred delivery date. Format: YYYY-MM-DD. Either this or `new_address` must be provided. |
| `new_address` | string | No | New delivery address (full). Either this or `new_date` must be provided. |

**Sample Response Shape:**

```json
{
  "rescheduled": true,
  "new_delivery_date": "2024-05-21",
  "confirmation_code": "DLVR-55021",
  "note": "Address changes are subject to courier availability. You will receive an SMS confirmation within 2 hours."
}
```

**Business Rules:**
- Do not reschedule if the package is already out for delivery. Say: "Once the package is out for delivery, it can't be rescheduled for that day. If you miss the delivery, the courier will attempt again the next working day."
- Always read back a full new address before submitting and get explicit confirmation.

**On failure:** "I wasn't able to make that change right now. I recommend calling the courier directly. Would you like me to give you the courier's tracking number so they can help?"

---

### `search_catalog`

**Purpose:** Search the product catalog to answer questions about specs, variants, availability, and compatibility.

**When to Call:** When a caller asks about a product's specifications, color or size options, stock status, or compatibility with something they already own.

**Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `query` | string | Yes | Natural-language search query close to the caller's own words. |
| `filter_category` | string | No | Optional category filter. Examples: `electronics`, `clothing`, `home_appliances`, `footwear` |

**Sample Response Shape:**

```json
{
  "results": [
    {
      "product_id": "PRD-1042",
      "name": "Wireless Earbuds Pro",
      "available": true,
      "price": 1499,
      "variants": ["black", "white"],
      "specs": {
        "battery_life": "24 hours with case",
        "connectivity": "Bluetooth 5.3",
        "water_resistance": "IPX4"
      },
      "compatibility_notes": "Works with any Bluetooth-enabled device."
    }
  ]
}
```

**Voice Rendering:**
- Never read raw JSON. Summarize the top result in two to three natural sentences.
- If multiple results: "I found a few options. The closest match is the [name]. Would you like details on that one?"
- If no results: "I'm not finding that in our catalog right now. It may be out of stock or listed under a different name. Would you like me to check a different search?"

**On failure:** "I'm having trouble searching the catalog at the moment. Can I connect you to a product specialist?"

---

## System Tools

### `transfer_to_number`

**Type:** System tool (native ElevenLabs)

**Purpose:** Transfer the call to a live human agent at a configured phone number.

**When to Use:** When the escalation subagent (N7) has delivered the handoff line and the workflow routes to the `phone_number` node (N8). This is the only mechanism for human escalation. There is no custom `transfer_to_agent` tool.

**Setup:** No manual tool creation needed. Configure the destination number directly in the `phone_number` workflow node.

**Before the transfer completes**, the escalation subagent says: "Let me connect you to a team member right away. Please hold for just a moment."

---

### `end_call`

**Type:** System tool (native ElevenLabs)

**Purpose:** Cleanly terminate the call after the closing sequence is complete.

**When to Use:** Only after the full closing sequence (summary, reference number if applicable, "anything else?" check, goodbye).

**Parameters:** None. Enable in agent tool settings; no configuration required.
