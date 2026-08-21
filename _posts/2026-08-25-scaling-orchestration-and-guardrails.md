---
layout: post
title: "Part 5: Scaling Up: Orchestration, Guardrails, Observability"
date: 2026-08-25 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [orchestration, guardrails, observability]
---

 
*Part 4 was state and infrastructure. This part is the harder scaling problem: keeping an LLM-driven system trustworthy as it does more. One idea organises everything here, and it crystallised slowly over months of fixes — the split between what the system guarantees and what the model merely interprets.*
 
## The line
 
Every safety-critical property in this system is a structural guarantee: enforced in code, holding no matter what the LLM outputs.
 
Writes are split into **preview and submit** — preview computes and describes the action, submit executes it, and submit only runs after an explicit confirmation state the host tracks. The model cannot "accidentally" write; the pathway doesn't exist. Identity is **injected**, never asserted: the verified `AccessContext` rides into every domain call from the host, so the model structurally cannot claim to be someone else. A **gate** stops every domain tool until verification succeeds — enforced in the orchestrator, not requested in a prompt. And confirmations follow a **contract**: an explicit yes across a turn boundary, with repeated unclear replies ticking a handoff counter until the conversation escalates to a human instead of looping forever.
 
Look at what's left for the LLM: intent classification, argument extraction, narration. Every one of them recoverable. A misroute produces a clarifying question, not a bad write. That's not an accident — it's the design. The model gets exactly the jobs where being wrong is cheap.
 
Honesty demands the other list too, as of the time of writing: timeouts, retry policy, per-session budget ceilings, and durable state beyond Redis TTLs were all still thin. Writing the weakness list down felt bad, and was half the roadmap.
 
## Two modes, one seam
 
For this system I built two orchestration modes:
 
1. **Mode A** (debug): plan, execute, then narrate — slower, but every step inspectable.
2. **Mode B** (production): a single merged LLM call plans and responds ; fewer round-trips, fewer LLM calls, and therefore faster replies.
The seam matters more than either mode. Because both live behind the same interface, switching is configuration. A bug needs step-by-step visibility? Flip to A, watch it think, flip back. Ship on B.
 
## Finding our place on the map
 
The core idea behind both modes comes from Hughes' series. His **V1 orchestrator**, the agentic loop, ran 17 to 34 LLM calls for the same request: high variance. His **V2 goal-aware** pattern: about 21 calls, low variance. His **V3 planner + DAG**: about 3 calls, zero variance: plan once, execute mechanically. His decision tree says conversational interfaces want V1's adaptability, and predictable batch work wants V3's discipline.
 
A multi-turn healthcare support conversation, though, needs both at once, conversational *and* predictable, which is why this system landed in the territory his Part 3 calls hybrid **adaptive planning**. Every turn runs a small plan: router plus planner, one merged call in mode B. Then the plan executes mechanically, validated, deduplicated, no agentic loop, no per-step "should I call this?" soul-searching. Per-turn cost is flat, like V3. But re-planning happens *every turn*, which restores V1's adaptability: the caller changes direction mid-conversation, and the next turn simply plans differently, with the session's accumulated state carried along in Redis.
 
To make sure both modes can sit behind the same interface so we change only the mode when we need to, never the whole system (yeah, plug-and-play again), I made their input and output identical:
 
1. **Mode A**: 1 LLM call to pick the domain(s), then, for *each* domain, 1 LLM call to choose the tool and extract its arguments. The result of the round is `domain + tool + args`.
2. **Mode B**: 1 LLM call to pick the domain(s), the tool(s), *and* the arguments in a single shot. The result of the round is also `domain + tool + args`.
Same shape in, same shape out so everything downstream (execution, narration, notes) neither knows nor cares which mode produced the plan.
 
At first glance, mode B looks like the obvious choice for a low latency system like an AI voice agent or call centre. But there's a catch: as the number of domains and tools grows, the single merged prompt grows with it, and a bloated context window is exactly where hallucination creeps in. Mode A, by splitting the decision into smaller, tightly-scoped calls, keeps each prompt lean. That's the real reason I built both (not as debug-vs-production), but as two points on a latency-versus-reliability trade-off I can move between when a conversation gets hairy. How the system *detects* that it should switch, and how it recovers, is a story for the next parts.
 
## The subsystem I didn't build
 
Not every turn needs tools. For example the user asks "what was that order number again?", the answer is sitting right there in conversation history. My first design for this was a whole separate answering subsystem, and it got pushed back on, correctly. The history was *already in the prompt*; the only gap was that the planner didn't know it was allowed to skip tools. The shipped fix is an `answer_from_context` flag, with narration-with-history as the safety net so a misfired flag degrades gracefully instead of failing.
 
I keep the memory of that review around as a warning: over-engineering usually announces itself as a new class.
 
## Notes written by code
 
Every conversation produces a structured note: for the human agents who pick up escalations, and for the audit trail, which in healthcare support matters clinically as well as operationally. The design choice: the note is built **deterministically from tool results**. Not generated. Not summarised by the model. Two rules govern it: facts come from tool outputs, so the note cannot hallucinate, because no generation is involved; and user vocabulary only — order IDs and names, never internal DB keys. If it can't be shown to the caller, it doesn't go in the note.
 
It took me a while to see what this actually was: context engineering's core discipline, applied in reverse. The same care about what enters the model's context applies to what leaves the system as a record. (Part 6 tells the story of what this note does for the model itself.)
 

## Tracing every step of system
 
Plenty of platforms trace LLM-powered systems these days: Langfuse, LangSmith, MLflow. I went with MLflow, and it traces every turn: router decision, plan, tool calls with timing, narration. One fix from this chapter is worth its line, tracing initialisation moved to lazy/background, because a synchronous init on the hot path is latency the user feels. Observability must never break the latency budget it exists to measure.
 
Believe me, a good tracing setup saves more time than you'd expect: when something misbehaves, the trace usually tells you *which* of router, plan, or narration went wrong before you've even opened the code. And there's a second payoff I didn't anticipate: nothing demonstrates the system's quality to your team and stakeholders quite like the numbers in the tracing dashboard. Latency percentiles, tool-call success rates, escalation counts ; harder to argue with than any demo.

## The lessons file
 
Here is the compressed version for future-me. Guarantees in code, interpretation in the LLM, never trade across that line for convenience. Latency budgets shape architecture more than most pattern article. Tracing is the cheapest way to actually *know* your system: understand the flow of every turn (what went in, what came out, which tools ran) and the LLM stops being a black box you pray to and becomes a system you can debug.

## References

- Chris Hughes, *From Orchestrator to Goal-Aware* (Part 2 — orchestration variance): https://chris-hughes10.github.io/posts/multi-agent-part2/
- Chris Hughes, *Planning Multi-Agent Workflows* (Part 3 — planner + DAG, hybrid adaptive planning): https://chris-hughes10.github.io/posts/multi-agent-part3/#hybrid-approaches
- Anthropic, *Building effective agents*: https://www.anthropic.com/research/building-effective-agents
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- MLflow LLM tracing: https://mlflow.org/docs/latest/llms/tracing/index.html

*Next: Part 6 — the context window deserved its own post: a forgetting bug, a naive fix, a U-shaped curve, and the note that can't be evicted.*
