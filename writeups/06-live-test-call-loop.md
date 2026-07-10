# The live test-call loop: four calls, five commits, one spec

I recently took a rebuilt inbound scheduling agent for a multi-location orthopedic group into pilot: about 40 scenarios, scheduling tools wired into the customer's EHR, a small fast model. Four live test calls produced five fix commits and one written spec for the dev team. This writeup is about the loop that turns a phone call into that output, and the four failure classes those calls surfaced.

---

## Why dashboard testing lies

The agent was clean in the platform's test console before call one. Then the first real call deadlocked.

Console testing is a simulation with four conveniences production doesn't have. You type instead of speak, so there's no transcription noise. Tool calls hit whatever environment the console points at, usually not the live database with its real rejection behavior. There's no latency pressure. And you are the caller, so you unconsciously feed the agent inputs it can handle.

A real phone call removes all four cushions at once: live DB, live tools, live latency, a real line with speech recognition doing what speech recognition does. Every one of my four calls failed on something the console had no way to show me.

---

## The loop

The loop I run is short and strict:

1. Make one call end to end, on a real phone line, against the live system. Play a plausible caller, not a friendly one.
2. Pull the transcript and the tool logs. Read every turn, including the ones that went fine.
3. Group failures by class, not by symptom. A pile of bad read-backs might be one missing rule.
4. Ship one commit per failure class, with the call that motivated it named in the message.
5. Anything the platform or backend owes you goes into a written spec, not a prompt workaround.

The one-commit-per-class rule matters most. Naming the class forces you to write the missing rule; patching symptoms one at a time never does.

---

## Call 1: the platform deadlock

Call 1 never reached a booking. I failed the phone-and-name verification on purpose, the patient lookup ended incomplete, and the platform's required-workflow gates blocked registration, the intent router, and booking. The agent was alive, polite, and completely stuck. The same call exposed a fallback node with an empty tools list: the escape hatch couldn't transfer anyone anywhere.

The fix was removing all 18 platform ordering gates and moving scenario order into the prompt as a hard numbered rule. That failure got its own writeup, number 05 in this repo, so the short version: must-pass gates fail closed when a prerequisite ends dirty, and a live caller can't wait while you notice. One failure class, one commit.

---

## Calls 2 and 3: speech discipline on a small model

Calls 2 and 3 got through the flow but sounded wrong, and on a small fast model (the Gemini 3.x Flash-Lite class from writeup 02), "sounded wrong" is a missing rule, not a model mood. Three failures landed in this class.

First, correction loops. I corrected a phone number read-back and the agent read the wrong number back again. I corrected it again. It started a third read-back. That loop is where callers hang up:

```
Two-strike rule for corrected read-backs:
If the caller corrects the same read-back twice, do NOT read it back a third time.
Say: "Let me get this right. Could you say the full number once more, slowly?"
Capture it fresh, then read the new value back once.
```

Second, digit-grouping fidelity. Callers give numbers in groups. Read the number back as one flat stream, or regrouped, and they can't check it against what they said, even when the digits are right. The rule: mirror the caller's grouping.

Third, phrasing. Verification questions read verbatim sound like a form being administered, so the rule now is natural wording for the same fields. And failure states got a forbidden-phrase list. Mine had said "still not finding you," which makes the caller feel like the problem. The agent must never say "re-verify," "try again," "still not finding," or "unable to verify." Give it the exact line to say instead. One class, one commit.

---

## Calls 3 and 4: contract fidelity

Call 3 also produced a string of createPatient rejects, and every one traced back to bytes. The live contract wanted camelCase `departmentId` sent as a string; the agent sent lowercase `departmentid`. The address had to be one slash-joined string, street/city/state/zip; the agent sent it comma-formatted. The reasoning was fine. The bytes were wrong, and casing and joining decided accept versus reject.

The fix has two parts, both repo discipline rather than clever prompting. Part one: byte-exact copies of the live system's real tool responses now live in the repo, with the code formatter fenced off. I write response-handling rules against the real strings, not my memory of them. Part two: canonical-value KBs. Every parameter that must match a backend enum (insurance plans, body parts, treatment types, location codes) gets a lookup table, and the prompt's rule is copy, don't compose:

```
departmentId: string, from the location table in the registration KB. Copy exactly.
address: one string, slash-joined: street/city/state/zip
  Example: "412 Oakmont St/Springfield/IL/62704"
insurance: one canonical plan name from the registration KB, copied character
  for character. Never paraphrase a plan name.
```

A small model is a good copyist and a bad improviser. Give it something to copy.

---

## Call 4: the stale-deploy trap

Call 4 was supposed to confirm the contract fix. Instead the agent sent filler strings, literally "not discussed," as the values for email and phone. And the failure response came back containing the literal text `{error}`: the placeholder naming the failing field was never substituted, so the agent guessed the wrong field and asked me to correct data that was fine.

Then the tool logs showed the real problem: the live system was running an older version of my agent, with none of the latest commits deployed. Most of that transcript was fiction, test data from a version that no longer existed in my repo.

The lesson is now step zero of the loop: before trusting a transcript, confirm the deploy state of the thing you just called. Diff a known-changed line if you have to. A stale-build test call doesn't just waste the call; it manufactures confident wrong conclusions.

---

## What goes in the dev spec instead of the prompt

Not everything a test call surfaces is a prompting problem, and the worst habit in this job is prompting around platform defects. The spec from these calls asks for server-side per-parameter validation before a request hits the EHR, rejecting early with the failing field named; a fix for the `{error}` placeholder substitution; and a server-side filler-text blacklist ("not discussed," "unknown," "N/A"), because a prompt-side ban on filler is a mitigation, not a guarantee. The empty-tools fallback node from call 1 went in too.

The dividing line: if the model behaved wrong, that's a prompt commit. If the model behaved fine and the system still failed, that's a spec. A workaround in the prompt hides the defect from everyone and pays tokens on every turn to keep hiding it.

---

## The ratio

Five commits and one spec from four calls is a good ratio. Each commit closed a failure class, not a symptom, and the spec put real platform defects on the record instead of burying them in my prompt. None of it would have surfaced in the console.

If you build voice agents, the live test-call loop isn't the step after the work. It is the work. Write the prompt, make the call, read the transcript like it's evidence, and let the failures tell you what to commit. The loop is the job.
