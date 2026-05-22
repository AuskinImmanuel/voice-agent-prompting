# Writing tools for voice agents

Tool use is where voice agents go from scripted phone trees to genuinely useful. It's also where I've seen the most production bugs. An agent that invents a parameter value, calls a lookup at the wrong moment, or reads a raw API response out loud is worse than no tool at all. Getting tool definitions right is not a backend concern. It's a prompting concern.

---

## The three tool categories in ElevenLabs

**System tools** are built into the platform. You enable them in the agent config; you don't implement them. The ones you'll use most on phone agents: `end_call`, `transfer_to_number`, `transfer_to_agent`, `voicemail_detection`, `language_detection`, `skip_turn`, `play_keypad_touch_tone`, and `knowledge_base_rag`. The platform handles their mechanics. Your job in the prompt is to tell the agent when to call them.

```
If the caller explicitly asks to speak with a person, call transfer_to_number immediately. Do not ask why.
If voicemail_detection fires at the start of the call, call end_call without speaking.
```

**Server tools** are webhooks you register with ElevenLabs. The platform calls your HTTP endpoint, and the response is injected back into the agent's context. You define the name, description, and parameters. This is where your business logic lives: order lookups, appointment bookings, CRM writes. The examples in this writeup (`lookup_order`, `book_slot`) are server tools.

**Client tools** are invoked on the client device. The agent emits a structured payload; the client app handles execution. Useful for UI actions or client-side logging. Less common in pure telephony.

---

## Anatomy of a well-defined server tool

Every server tool needs four things in the prompt: what it does, when to call it, what inputs it requires and where they come from, and what to do with the response. Here's a minimal complete example:

```
Tool: lookup_order

Purpose: Retrieves the current status and estimated delivery date for a customer's order.

Call this when: The caller asks about order status, shipping, delivery date, or where their package is.

Do NOT call this until you have: either the caller's email address OR their order ID number.
If you have neither, collect one of them first.

Parameters:
  email (string, optional): caller's email address as spoken. Lowercase, no spaces.
  order_id (string, optional): order ID number from the caller. Digits only, no dashes.
  At least one of email or order_id is required.

Response fields you will receive:
  status: one of "processing", "shipped", "out_for_delivery", "delivered", "cancelled"
  estimated_delivery: date string in YYYY-MM-DD format, may be null
  carrier: shipping carrier name, may be null
  tracking_number: alphanumeric string, may be null
```

The critical things here: the "do NOT call until" clause, the explicit parameter source ("as spoken"), and the enumerated response fields. Each of those elements exists to prevent a specific failure mode.

---

## The two-phase pattern: lookup then act

Most server tool interactions are two steps: first look something up (read-only), then act on what you found (write/mutate). Keep them as separate tools; don't conflate them. Example: a restaurant booking agent.

```
Tool: check_availability
Purpose: Check open time slots for a given date and party size.
Call this: After collecting date and party size from the caller.
Parameters:
  date (string): in YYYY-MM-DD format
  party_size (integer): number of guests

Tool: book_slot
Purpose: Confirms a reservation in the system.
Call this: ONLY after (1) check_availability returned a matching slot AND
           (2) the caller explicitly confirmed the date, time, and party size.
Parameters:
  date (string): YYYY-MM-DD
  time (string): HH:MM in 24-hour format
  party_size (integer)
  guest_name (string): caller's name as provided
  phone (string): callback number as provided
```

Both are server tools. The gates on `book_slot` are not optional decoration. A small model will call an action tool early if you don't constrain it. I've seen agents book reservations before the caller confirmed a time because the data was already in context. An explicit "ONLY after" condition closes that gap.

---

## Hydrating the response: speaking tool output correctly

When a server tool returns data, the agent needs to translate it into natural speech. Without explicit instructions, it reads raw field values aloud ("Your status is out_underscore_for_underscore_delivery") or adds hedge language you didn't ask for. Define the mapping explicitly:

```
After receiving lookup_order response, speak:

status field mapping (say the phrase, never the raw value):
  "processing"        -> "Your order is still being processed."
  "shipped"           -> "Your order has shipped."
  "out_for_delivery"  -> "Your order is out for delivery today."
  "delivered"         -> "Your order was delivered."
  "cancelled"         -> "This order has been cancelled."

If estimated_delivery is not null: add "Estimated delivery is [month name] [day]."
If carrier and tracking_number are not null: add "You can track it with [carrier] using tracking number [read digits individually]."
If estimated_delivery is null: do not mention delivery date at all.
```

The last line matters as much as the others. Silence about a null field is a deliberate instruction. Without it, some models hallucinate a delivery date or volunteer "I don't have an estimated date" when the caller didn't ask.

---

## Projections: only return the fields you need

If your server tool returns 40 fields and the agent only speaks 4, those extras sit in context on every subsequent turn, adding overhead and giving the model material to mention things the caller didn't ask about.

Return projected responses with only the fields the agent uses. If you can't control the API response, say so in the tool definition:

```
Ignore these fields: internal_id, warehouse_code, agent_notes, batch_id.
Only use: status, estimated_delivery, carrier, tracking_number.
```

---

## Common failure modes

**Invented parameter values.** Happens when the tool is available but required data hasn't been collected. The model fills in guesses. Fix: "Do NOT call this tool until you have collected [X]." Make the gate explicit.

**Action tool called too early.** The model calls a booking or mutation tool before the caller confirms. Fix: "Call this tool ONLY AFTER the caller explicitly confirms." Explicitly confirms means a clear yes, not the absence of a no.

**Raw result read aloud.** The model reads a date string as digits, a status code verbatim, or adds hedge language not in the response. Fix: field mapping tables (see hydration above) and TTS rules in `# Tone`.

**Repeated lookup calls.** The model calls the lookup again even though the result is already in context. Fix: "Call lookup_order only once per conversation. If you already have the status, use it from context."

---

## A note on tool names and descriptions

The name and description are the model's primary signal for when to call a tool. Use names that describe the action: `lookup_order` not `get_order_by_email_or_id_v2`. One sentence: what it does and when to reach for it. Ambiguous names produce ambiguous call behavior. System tools have platform-defined names you cannot change.

Worked examples showing full tool definitions alongside the agent prompts that use them are in the `/agents` folder of this repo. The inbound agents there are built as Agent Workflows, where system tools like `transfer_to_number` and `end_call` appear alongside custom server tools in each subagent node's `additional_tool_ids`.
