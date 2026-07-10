# Workflow gates deadlock. Prompt-order rules don't.

Recently I removed every workflow gate from a production voice agent, all 18 of them, in a single commit. The trigger was one live test call where the agent locked up with a caller on the line. The gates were doing exactly what the platform promised they would do. That was the problem. This writeup is about why hard scenario gates fail on live phone calls, and why the same ordering, written as a rule in the prompt, holds up better.

---

## What a must-gate promises

Multi-scenario voice platforms usually offer some form of dependency gate: mark scenario B as requiring scenario A, and the platform guarantees B cannot run until A completes. The pitch is safety through enforcement. The model can't skip identity verification and jump straight into booking, because the platform physically won't let it.

On the agent in question, an inbound line for a multi-location orthopedic group integrated with the customer's EHR, that looked like this:

```
registerPatient:        requires patientLookup [complete]
bookAppointment:        requires patientLookup [complete]
rescheduleAppointment:  requires patientLookup [complete]
cancelAppointment:      requires patientLookup [complete]
```

The agent had about 40 scenarios and 18 of these gates, and nearly everything downstream was gated on the patient lookup. It felt responsible. Verification first, always, enforced in config rather than trusted to a prompt. I would have defended it in review.

---

## The failure mode: "incomplete" is not a plan

A gate checks one thing: did the prerequisite scenario complete? It has no opinion about how the scenario ended, and no concept of what should happen if it didn't.

So ask the obvious question. What happens when verification fails? The caller gives a name and phone number, the lookup finds no match, and the scenario ends without a verified patient. As far as the platform is concerned, the prerequisite is incomplete. Every gated scenario stays locked.

Now hold that state up against the medium. In a chat product an unfinished prerequisite is annoying but survivable. The user can retry, refresh, come back tomorrow. On a phone call there is a live human on the line, and the clock runs in seconds. An agent with every path locked has nowhere legal to go. It can't register the caller, can't route them, can't book. It can only stall.

---

## The call where it happened

First live test call against the real database. The caller's name and phone number didn't match any record. That's a normal outcome, wrong numbers and misheard names are everyday telephony, and it's exactly what should route a caller into registration as a new patient.

Instead, the lookup ended unverified and the gates fired in unison. Registration blocked, because it required a completed lookup. The intent router blocked, so the agent couldn't even move the caller to a different request. Booking blocked. The one scenario built for this exact situation, registering the patient we couldn't find, was locked behind the failure it existed to handle.

Nothing malfunctioned. Every gate did precisely what it was configured to do. The design was the bug: the prerequisite could end in a state the gates didn't model, and the recovery path was on the wrong side of the wall.

---

## The fix: order moves into the prompt

I removed all 18 gates in one commit and replaced them with a single hard rule near the top of the system prompt:

```
Scenario order (hard rule):
1. Run patientLookup first, before any other scenario.
2. Stay in patientLookup until it reaches a terminal state:
   verified match found, OR no match after one retry.
   No match IS terminal. Do not loop on verification.
3. On no match: continue as a new patient. Go to registerPatient.
   Never say "I can't verify you" or "let's try that again."
4. Only after step 2 is terminal: route to whatever the caller
   actually asked for (booking, rescheduling, cancellation, anything).
```

Two lines carry most of the weight. "No match IS terminal" gives failure a defined ending, so a failed verification is a finished step rather than a stuck one. And step 3 gives that ending a destination: the lookup-fail path now flows silently into registration instead of dead-ending. The caller who can't be found becomes a new patient, and the call keeps moving.

That commit was one of 5 fix commits (plus 1 dev spec) that came out of 4 live test calls. None of the later calls deadlocked.

---

## Why a soft rule degrades where a gate deadlocks

The difference isn't strictness. It's what happens at the edges.

A gate is evaluated by code that can't see the call. It gets a boolean, complete or not, and enforces it regardless of what the caller is doing. When reality produces a state the config didn't anticipate, the gate holds its wall, and the wall is the failure.

A prompt rule is read by the model on every turn with the full conversation in view. When reality goes sideways, the model can route around it. A caller who interrupts verification to ask whether the clinic takes their insurance gets an answer, then gets pulled back to step 2. That flexibility is exactly what you don't want around a payment, and exactly what you do want around conversation order.

Compare worst cases. The worst case for the prompt rule is that the model bends the order, does something slightly early, and you tighten the wording in the next revision. The worst case for the gate is a frozen call with a live person inside it. One of those is a bug you fix. The other is a caller you lost.

---

## The residual risk, and how to pin it

The honest cost of a soft rule: a small, fast model in the Gemini 3.x Flash-Lite class can skip a global instruction that sits a couple hundred tokens away from where it matters. I covered this in the low-latency writeup, and the fix is the same here: repeat the rule at point of use.

The ordering rule lives at the top of the system prompt, and each downstream scenario block restates its own precondition at its entry:

```
### bookAppointment
Precondition: patientLookup reached a terminal state on this call.
If it has not run yet, run it first, then return here.
```

Note the shape. The scenario reminds, it doesn't block. If the model arrives early, the instruction points it back to the lookup and then back here. There is always a legal next move. In the live test calls after the rewrite, ordering held without a single gate left in the config.

---

## Gates for irreversible actions only

The rule of thumb I took away: reserve hard gates for actions you can't take back. A booking write, a charge, a record mutation, anything where "the model did it early" costs more than a stalled call. Even then, prefer the gate at the tool boundary ("call bookAppointment ONLY after the caller explicitly confirms date, time, and provider"), and where it really matters, a server-side check behind the tool.

Conversation order is not irreversible. A question asked out of sequence is recoverable in one turn. So order lives in the prompt, where the model can hold it firmly on the happy path and loosely enough everywhere else to survive contact with real callers. If a gate can trap a live caller with no legal move, it shouldn't exist.

---

The inbound sample agents in the `/agents` folder of this repo are built this way: scenario ordering and preconditions live in the base prompt and the per-node instructions, and the only hard "ONLY after" conditions sit on action tools. That split is deliberate, and after that first test call, permanent.
