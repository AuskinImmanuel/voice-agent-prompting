# Prompting architecture for production voice agents

I've shipped a dozen or so production voice agents on ElevenLabs Conversational AI. Before the first one went live, I assumed writing a voice prompt was roughly like writing a chat prompt, maybe a little shorter. That assumption caused bugs in about the first three calls we listened to. Voice is a fundamentally different medium, and the prompt architecture has to account for that from the start.

---

## Why voice changes everything about prompt structure

In a chat interface, a user can scroll back, re-read the bot's last message, and take their time. On a phone call, the interaction is ephemeral and linear. The caller heard what they heard, and they're already formulating a response before you're done speaking. If the caller interrupts (and they will), the agent has to handle partial context and move on.

Latency matters too. Every instruction in your prompt costs tokens, and on a live call, token processing happens on every turn. Bloated prompts directly translate to longer response latency, which translates to awkward silences, which translates to callers saying "hello?" or hanging up. Structure your prompt for the machine's reading speed.

---

## The anatomy of a production system prompt

ElevenLabs defines a canonical six-section structure for agent system prompts. Each section has a specific heading and purpose:

**`# Personality`.** Who the agent is and what role it plays. Keep this to two or three sentences. This is the agent's identity: name, company, purpose. No backstory.

```
You are Maya, a booking agent for Creston Dining. You handle incoming calls from guests who want to make, modify, or cancel reservations.
```

**`# Environment`.** Channel context, date/time injection, and caller context. Often skipped by people new to voice prompts. If your agent needs to know it's Tuesday evening when deciding whether to offer same-day slots, inject `{{system__time}}` here (ElevenLabs injects current date and time together in this single variable). Custom variables passed at conversation init via the Conversations API also belong here.

**`# Tone`.** How the agent sounds: pacing, warmth, formality, and voice output rules. TTS reads whatever the model outputs, so this section also carries number-readback rules, abbreviation handling, and processing filler phrases.

**`# Goal`.** One paragraph, three bullets at most. Primary objective, success definition, and escalation threshold. North star for the model when a caller does something unexpected.

**`# Guardrails`.** What the agent must never do, say, or reveal: forbidden phrases, off-limits topics, data integrity rules, escalation triggers. Also the right place for universal rules that apply to every turn: one question at a time, never re-ask what you already have.

**`# Tools`.** Definitions of every external capability the agent can call: when to call each one, required inputs, and how to speak the response.

---

## Point-of-use rules vs. global rules

If a rule only matters in a specific situation, put it at that situation, not in `# Guardrails` alone. A rule saying "never tell the caller how many open slots are left" may not surface during the booking flow if it lives only 200 tokens away at the top. Repeat it inside the relevant node's instructions so the model sees it when it needs it.

`# Guardrails` is for things that apply everywhere: don't make promises you can't keep, don't reveal internal system names, one apology maximum per call. Situational rules live at the point of use.

---

## Literal lookup tables vs. abstract reasoning

Abstract rules ("handle status codes appropriately") leave too much room for model interpretation. Lookup tables close that gap.

```
Status code mapping -- speak the phrase, never say the code:
  "CONFIRMED"  -> "Your reservation is confirmed."
  "WAITLIST"   -> "You're on the waitlist for that time."
  "CANCELLED"  -> "That reservation has been cancelled."
  "NOT_FOUND"  -> "I don't see a reservation under that name."
```

The model doesn't have to reason about what "appropriately" means. It just matches the value it got from the tool and speaks the mapped phrase. On a live call, that determinism matters.

---

## Forbidden-phrase lists

There are phrases that sound fine in text but are wrong for voice, legally risky, or just inconsistent with the brand. I maintain an explicit list:

```
NEVER SAY:
- "I'll definitely get that done for you" (guarantee you can't keep)
- "Let me transfer you to our system" (reveals internal tooling)
- "One second" (implies a literal one-second wait that won't happen)
- "Unfortunately" (before a refusal -- leads to over-apologetic tone)
```

Negative lists like this work better than positive replacements in most cases, because the model knows the entire space of acceptable phrasings and can pick naturally within it.

---

## Single-agent prompts vs. Agent Workflows

For simple agents with one or two use cases, everything lives in a single system prompt. A restaurant booking line that only books and cancels? One prompt with a clear `# Goal` and `# Tools`, done.

When an agent handles several distinct workflows that each have their own data-collection steps, tool calls, and branching logic, cramming all of it into one flat prompt creates two problems: the prompt gets long and slow to process, and the model starts blending flows in ways you didn't intend.

ElevenLabs' native answer to this is **Agent Workflows**: a graph of nodes connected by edges. The node types you'll use most:

- `start`: entry point of the workflow.
- `override_agent` (subagent node): loads a specialized sub-prompt alongside the base agent. Carries `additional_prompt` (the per-intent instructions) and `additional_tool_ids` (the tools that node can call). This is the primary building block for multi-intent routing.
- `tool`: executes a specific tool call at a defined point in the graph.
- `phone_number`: transfers the call to a number.
- `end`: terminates the workflow.

The main agent (at `start`) handles routing. When it identifies the caller's intent, the workflow transitions to the right `override_agent` node, which carries only the instructions and tools for that flow. Adding a new intent means adding a new node, not touching existing ones.

Rule of thumb: if adding a new flow doesn't require touching any existing flow, it's a good candidate for its own `override_agent` node. If the flows share a lot of state or need to pass data mid-call, a single flat prompt with clear section headers is often simpler.

---

## A word on section ordering

The model processes top to bottom on every turn. Put `# Personality` and `# Goal` at the top. Put `# Guardrails` before `# Tools`. Don't bury TTS rules at the very end. I've seen prompts where number-readback rules were on line 300 of a 320-line prompt and the agent consistently mispronounced order IDs. Moving them up fixed it. Sequence is real.

---

Worked examples for each pattern described here are in the `/agents` folder of this repo. The inbound agents there are built as Agent Workflows, so you can see the node-and-edge structure alongside the prompts.
