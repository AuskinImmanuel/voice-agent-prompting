# voice-agent-prompting

How I write production voice agents: the prompting architecture, the model choices, and a set of worked sample agents you can build yourself.

I'm Auskin Immanuel. I build conversational voice agents for a living, on ElevenLabs, the kind that answer or place real phone calls at scale. This repo is the part of that craft I can share openly: the techniques, not any one employer's prompts. Everything here is written fresh, generic, and safe to read.

## Why this repo exists

Most "prompt engineering" advice is written for chat. Voice on a phone line is a different problem. There is no scrollback, the caller talks over you, latency is brutal, and the call can end at any second. The way you write a prompt has to change for that. This repo is what I've learned doing it.

Two things live here: short writeups on the how, and full sample agents that put the how into practice. The writeups now cover the whole arc, from prompt architecture through live pilot calls and multi-location routing. If you'd rather hear it than read it, my site has a voice agent built with everything in this repo: [auskinimmanuel.github.io](https://auskinimmanuel.github.io), click the orb and talk to Lia.

## Writeups

- [Prompting architecture for production voice agents](writeups/01-prompting-architecture-for-voice-agents.md): the anatomy of a voice-agent system prompt, point-of-use rules vs global rules, lookup tables over abstract reasoning, and when to use a single-prompt agent vs a multi-intent workflow.
- [Prompting for small, fast models on telephony](writeups/02-low-latency-telephony-prompting.md): why a fast, low-latency model (the Gemini 3.x Flash-Lite class) is usually the right call for live phone calls, and how you write prompts differently for one.
- [Writing tools for voice agents](writeups/03-writing-tools-for-voice-agents.md): defining tools the agent calls, the two-phase lookup-then-act pattern, handling the response so the agent speaks it correctly, and the common failure modes.
- [Scaling a prompt down: lean base plus scenario cards](writeups/04-lean-base-and-scenario-cards.md): restructuring a large multi-scenario agent for a small fast model, a short base prompt plus one card per scenario, a single trailing guardrails block, and cutting per-turn context from about 20K to 6.4K tokens.
- [Workflow gates deadlock. Prompt-order rules don't.](writeups/05-gates-deadlock-prompt-order-wins.md): why platform must-gates freeze a live call when a prerequisite scenario ends incomplete, the removal of all 18 gates that fixed it, and how to carry scenario order as a hard numbered rule in the prompt instead.
- [The live test-call loop: four calls, five commits, one spec](writeups/06-live-test-call-loop.md): why dashboard testing lies, the run-read-group-commit loop, four failure classes from live test calls (platform deadlocks, speech discipline, contract fidelity, stale deploys), and what belongs in a dev spec instead of the prompt.
- [Location mapping and human handoff for multi-branch voice agents](writeups/07-location-mapping-and-human-handoff.md): deriving the caller's branch instead of reading them a menu, fallback ladders built on measured distance rather than list order, walling off sites you can only transfer to, and the keyed anchor-phrase contract behind every human handoff.

## Sample agents

Six sample agents, built natively for the ElevenLabs platform, across three verticals. Per vertical there is one inbound agent and one outbound agent. All generic and non-branded.

Inbound agents are built as native ElevenLabs Agent Workflows: a graph of a start node, one subagent node per caller intent (each an `override_agent` node with its own instructions), and edges routing between them. Outbound agents are single ElevenLabs agents with one system prompt and one linear flow.

| Vertical | Inbound (Agent Workflow) | Outbound (single agent) |
|---|---|---|
| Automotive / used cars | [inbound](agents/automotive-used-cars/inbound-workflow/) | [outbound](agents/automotive-used-cars/outbound-single/) |
| E-commerce | [inbound](agents/ecommerce/inbound-workflow/) | [outbound](agents/ecommerce/outbound-single/) |
| BFSI / fintech | [inbound](agents/bfsi/inbound-workflow/) | [outbound](agents/bfsi/outbound-single/) |

Each inbound folder has a README, `workflow.md` (the node and edge graph), `base-agent.md` (the 6-section base system prompt), a `subagents/` folder (one file per intent), and `tools.md`. Each outbound folder has a README, `system-prompt.md`, and `tools.md`. They are written so you can build them in the ElevenLabs dashboard and make real calls. See [agents/README.md](agents/README.md) for the full index.

## A note on what's not here

I don't publish anything from work. No employer prompts, no client names, no real call data. The techniques are general craft and the sample agents are written from scratch for this repo in domains I don't work in. That's deliberate.

## Reach me

[LinkedIn](https://www.linkedin.com/in/auskin-immanuel/)
