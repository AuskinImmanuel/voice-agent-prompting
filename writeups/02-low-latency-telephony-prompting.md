# Prompting for small, fast models on telephony

When I first started building phone agents on ElevenLabs, the default instinct was to reach for the biggest model available. More reasoning, fewer hallucinations. That instinct is reasonable for a chatbot or an async task. It's wrong for a live phone call.

---

## The telephony latency budget

There's a well-documented perceptual threshold: around 500ms to 1.5 seconds of silence after the caller stops speaking, they start to wonder if the call dropped. By 2 seconds, some say "hello?" By 3 seconds, some hang up.

Total response latency on a voice AI call is roughly: STT processing + model inference + TTS synthesis + audio delivery. You don't control most of that pipeline. What you do control is model inference time, and that's where prompt engineering intersects directly with call quality.

A small, fast model class like Gemini 3.x Flash-Lite sits at the low end of inference latency. It's not as capable as a frontier model on complex reasoning, but on a live call, "complex reasoning" isn't usually what's happening. The caller wants to change their delivery address, check if their package shipped, or book a table. These are information-retrieval and data-collection tasks. A fast model handles them fine, provided the prompt does the heavy lifting the model's smaller reasoning budget can't.

The trade-off is explicit: you move complexity from the model to the prompt. The model gets simpler; the prompt gets more precise.

---

## Writing terse, imperative instructions

Frontier models can infer intent from loose, principle-based instructions. Small models do better with direct commands.

Instead of:
```
Be sure to confirm critical information with the caller, especially when it comes to
dates or contact details, as mistakes can lead to incorrect bookings.
```

Write:
```
After collecting date and phone number: repeat both back. Ask "Is that correct?" Wait for yes before proceeding.
```

Same rule. The second version is half the tokens and there's nothing to interpret. "Be sure to" and "especially when it comes to" are filler the model doesn't reliably act on.

---

## Repeating rules at point of use

Small models don't always retrieve global rules when they're buried 200 tokens from where they're needed. The fix: repeat the rule at the exact step where it applies.

In `# Guardrails`:
```
Never read order IDs verbatim. Speak each digit individually.
```

And inside the order lookup flow (in `# Goal` or the relevant subagent node's instructions):
```
3. Read back the order ID digit by digit. Example: order 748392 -> "seven four eight three nine two"
```

Yes, it's repetition. You're not writing for a human reader. You're writing for a model that processes the prompt fresh on every turn and benefits from proximity of instruction to application.

---

## Procedural checklists over principles

"Use good judgment when the caller seems confused" doesn't translate reliably on a small model. Checklists do.

Example: order status flow checklist
```
Before responding to an order status question:
[ ] Did I call lookup_order with the caller's email?
[ ] Did the tool return a result?
[ ] If yes: speak status, estimated delivery, and carrier. Stop.
[ ] If no result: ask for order number instead of email. Retry once.
[ ] If still no result: tell caller to check their confirmation email and offer to connect them with support.
```

The checklist format is literal and sequential. The model doesn't have to decide what "good judgment" means. It works through the steps.

---

## Keyword-to-action tables

For anything that requires mapping caller language to a specific agent behavior, a table is more reliable than a paragraph.

```
Caller says any of these   |  Agent does
---------------------------|----------------------------------
"cancel", "don't want it"  |  Start cancellation flow
"change address", "wrong address" | Start address update flow
"where is it", "shipping"  |  Call lookup_order, speak status
"speak to someone", "agent", "human" | Transfer to support queue
```

A small model reading a lookup table like this behaves closer to a deterministic router than a reasoning system. On a live call, that's what you want for high-frequency intents.

---

## Tight flow scoping

Each flow section (or subagent node's instructions in an Agent Workflow) should start by declaring its scope and immediately excluding adjacent topics. This prevents the model from conflating flows when the caller's phrasing is ambiguous.

```
### Change delivery address

Handles: requests to update shipping address on an order that hasn't shipped yet.
Does NOT handle: refunds, cancellations, items already delivered, or address changes on shipped orders.

If the caller asks about any of those, say: "For that I'll need to connect you with our team."
```

The "does NOT handle" clause is important. Without it, a small model may try to handle adjacent requests using whatever instructions are nearest in the prompt, producing hallucinated or partially correct responses.

---

## Controlling digit and number readback

TTS engines render what the model outputs. If the model outputs "748392", TTS may read it as "seven hundred forty-eight thousand three hundred ninety-two" which is wrong for an order ID. Be explicit:

```
Number readback rules:
- Phone numbers: each digit, pauses between groups of 3. "555 ... 248 ... 9301"
- Order IDs (6-8 digits): every digit individually. "7 4 8 3 9 2"
- Dollar amounts: spell out. "$45.00" -> "forty-five dollars"
- Dates: month name always. "2025-04-15" -> "April fifteenth, twenty twenty-five"
```

State the output format explicitly. Don't rely on the model to infer it.

---

## Handling filler and silence

The caller experiences a gap between when they stop talking and when they hear audio. You can't fully close it, but you can manage the perception.

Train the model to use bridging phrases before tool calls:
```
Before calling any tool, speak a brief processing indicator first:
- "Let me pull that up."
- "One moment while I check."
- "Got it, looking that up now."
Say one of these before the tool call. Do not stay silent.
```

These lines cost milliseconds to synthesize and buy you a natural-sounding pause during the actual lookup. They're not UX polish. On a small model with tight latency, they're a core part of the call experience.

---

## Summary

Writing for a small, fast telephony model is a deliberate discipline. You give up implicit reasoning and replace it with explicit tables, checklists, point-of-use repetition, and scoped scenario blocks. The prompt does more work. The model does less. The call is faster and more reliable because inference time is genuinely shorter. The constraint is a feature: having to write explicit, mechanical instructions forces you to resolve ambiguity in the prompt rather than leaving it to model inference.

Worked examples illustrating these patterns are in the `/agents` folder of this repo. The inbound agents there are built as Agent Workflows, so you can see how flow scoping and point-of-use repetition apply within individual subagent nodes.
