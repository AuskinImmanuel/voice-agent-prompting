# Scaling a prompt down: lean base plus scenario cards

Writeup 02 was about sentence-level technique for small fast models: terse imperatives, checklists, lookup tables. This one is about what happens when that isn't enough. I recently rebuilt an inbound scheduling agent for a multi-location orthopedic group, about 40 scenarios wired into the customer's EHR, whose per-turn context had crept to about 20K tokens. The target was a small fast model in the Gemini 3.x Flash-Lite class, which degrades well before that. Sentence tightening alone wasn't going to close that gap. The prompt had to change shape: a lean base plus one card per scenario.

---

## The problem: a prompt that outgrew its model

The agent books, reschedules, cancels, and confirms appointments, registers new patients, and answers the long tail of front-desk questions. Each of those is a scenario with its own data collection, tool calls, and branches. Scenarios accrete one at a time over months of customer calls, each one harmless on its own. Nobody decides to write a 20K-token prompt. You just wake up one day owning one.

Two costs land at once. The first is latency: per-turn tokens are per-turn silence, and the caller hears all of it. The second is worse. A small fast model doesn't degrade gracefully as instructions pile up. It skips steps, blends adjacent flows, and stops retrieving rules that sit far from where they apply. A frontier model reads a 20K prompt and mostly behaves. The class I was targeting does not, and that was on purpose: I moved roughly nine agents onto that class in four weeks, all for latency. This agent was the largest of them.

---

## The shape

On any given turn, the model now reads four things, in this order:

```
1. Identity         (2-3 sentences)
2. Scenario order   (hard numbered rule: lookup first, stay until
                     terminal, then route; never book without a
                     patient record in context)
3. The active card  (the one scenario the call is in right now)
4. Guardrails       (one block, read last)
```

Identity, ordering, and guardrails are the base. Every line there is paid for on every turn, which is why it stays short. The 40 cards live outside it, and the model only ever sees the card for the scenario it's in. That's what makes scenario count nearly free: adding scenario 41 doesn't slow down turn twelve of a cancellation call.

The ordering rule is the piece of the base I'd defend hardest. Sequencing used to live in platform gates, and the first live test call deadlocked the whole agent when a failed verification left the prerequisite scenario incomplete. I removed all 18 gates and moved scenario order into the prompt. That failure got its own writeup (05), so the short version here: order belongs in the prompt, where the model can bend it around a real caller instead of freezing on them.

---

## Anatomy of a scenario card

Four ingredients, in this order, and nothing else:

1. A scope line: what the card handles and what it does not.
2. Numbered steps, one action per line. An action is observable: ask this, call this tool, say this, route there.
3. Condition-to-action branches, written as "If X: do Y" lines under the step they modify.
4. One few-shot example of a real exchange.

```
### rescheduleAppointment
Handles: moving an existing appointment.
Does NOT handle: cancellations, new-patient bookings.

1. Call getAppointments.
2. If none found: say "I don't see an upcoming appointment for
   you." Offer to book one instead.
3. Read the appointment back: day, date, time, office.
4. Ask what day works better. Collect one target day.
5. Call getSlots for that day.
6. If no openings: offer the two nearest days that have them.
7. Read the chosen slot back. Ask "Should I move it?" Wait for a
   clear yes.
8. Move the appointment. Confirm the new day and time. Stop.

Example:
Caller: "I need to move my Thursday appointment."
Agent: "Sure. I see Thursday the 14th at 2:10 at our Main Street
office. What day works better for you?"
```

The discipline is in what's missing. No paragraph on why we confirm before moving anything. No tone reminders. No repeated guardrails. The test for every line is mechanical: if this line disappeared, would any turn's output change? If not, it goes.

---

## One guardrails block, read last

The old prompt had global rules copied into scenario after scenario: number read-back formats, verification phrasing, things the agent must never say in failure states. Writeup 02 argues for repeating rules at point of use, and I stand by that, but point of use means the one step where a rule applies. A rule that applies to every turn of every scenario has no single point of use. Copying it into each card multiplies tokens and, worse, versions. I found cards carrying slightly different wordings of the same read-back rule, and the model treated the drift as meaningful.

So global rules live in exactly one block, at the end of what the model reads, after the active card. On a small model that position is not cosmetic: the end of the context is the instruction nearest the output, and I've watched rules start landing simply because they moved there. The split is clean: in the card if one scenario needs a rule, in the trailing block if they all do.

---

## What got cut, and why it was safe

Three categories account for almost all the tokens that left, none of them behavior:

**Prose and rationale.** Why-paragraphs for future maintainers, moved to a spec document outside the prompt. The model doesn't act on explanations, and no human reads them mid-call.

**Duplicate rules.** The global-rule copies described above, collapsed into the one trailing block. Consolidation, not deletion.

**Style padding.** "Be sure to," "it's important that," goal restatements at the top of every flow. Filler a small model doesn't act on; writeup 02 covers it line by line.

Every removed line was a duplicate (kept once), an explanation (relocated), or filler (deleted). The before and after have the same scenarios, tools, and branches. Nothing the agent does on a call was touched.

---

## The result

Per-turn context went from about 20.4K tokens to 6.4K. Same agent, same scenario count, still about 40. Turns got faster because there is genuinely less to read, and the agent holds its place in a flow the way it never reliably did at 20K.

The live test calls that followed still surfaced real failures, but of a telling kind: a tool contract wanting exact casing, a read-back phrasing rule. Sharp, specific, one commit each (writeup 06 walks through them). What the calls did not surface was the diffuse kind, the model losing track of which scenario it's in, which is what a 20K prompt produces on this model class.

---

## When not to do this

Three cases:

- **You're on a frontier model and can afford to be.** Frontier models retrieve distant rules and tolerate prose. If your flow needs that reasoning and your latency budget covers it, carding everything up buys less and costs maintainability.
- **Low scenario count.** Under roughly ten scenarios, a single flat prompt in the six-section structure from writeup 01 is already small. Cards add structure without token savings that matter.
- **No one is on the line.** Batch and async tasks don't pay latency in caller patience, and a big prompt costs pennies there. Optimize for readability instead.

The pattern earns its keep at the intersection: many scenarios, small model, live caller. That's most of production telephony.

---

The sample inbound agents in the `/agents` folder of this repo are built in this shape: a short base agent plus one subagent file per intent. Smaller scale, same architecture.
