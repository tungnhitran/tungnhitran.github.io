---
layout: post
title: "Part 5: Scaling Up — Orchestration, Guardrails, Observability"
date: 2026-08-25 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [orchestration, guardrails, observability]
---

*Part 4 was state and infrastructure. This part is the harder scaling problem: keeping an LLM-driven system trustworthy as it does more. One idea organises everything here, and it crystallised slowly over months of fixes: the split between what the system guarantees and what the model merely interprets.*

## The line

Every safety-critical property in this system is a structural guarantee — enforced in code, holding no matter what the LLM outputs.

Writes are split into **preview and submit**: preview computes and describes the action, submit executes it, and submit only runs after an explicit confirmation state the host tracks. The model cannot "accidentally" write; the pathway doesn't exist. Identity is **injected**, never asserted: the verified `AccessContext` rides into every domain call from the host, so the model structurally cannot claim to be someone. A **gate** stops every domain tool until verification succeeds — enforced in the orchestrator, not requested in a prompt. Confirmations follow a **contract**: an explicit yes across a turn boundary, and repeated unclear replies tick a handoff counter until the conversation escalates to a human instead of looping forever. And the **pending limit** — one pending order per owner — is an atomic cache check at create time, immune to persuasion.

Look at what's left for the LLM: intent classification, argument extraction, narration. Every one of them recoverable. A misroute produces a clarifying question, not a bad write. That's not an accident — it's the design. The model gets exactly the jobs where being wrong is cheap.

Honesty demands the other list too, as of the time of writing: timeouts, retry policy, per-session budget ceilings, and durable state beyond Redis TTLs were all still thin. Writing the weakness list down felt bad and was half the roadmap.

## Two modes, one seam

Orchestration runs in two modes behind one interface. Mode B, production: a single merged LLM call plans and responds — fewer round-trips, faster replies. Mode A, debug: plan, execute, then narrate — slower, but every step inspectable.

The seam matters more than either mode. Because both live behind the same interface, switching is configuration. A bug needs step-by-step visibility? Flip to A, watch it think, flip back. Ship on B.

## Finding our place on the map

Hughes' series maps the coordination patterns with actual benchmarks, and reading it felt like finding our coordinates. His **V1 orchestrator** — the agentic loop — ran 17 to 34 LLM calls for the same request, high variance. His **V2 goal-aware** pattern: about 21 calls, low variance. His **V3 planner + DAG**: about 3 calls, zero variance — plan once, execute mechanically. His decision tree says conversational interfaces want V1's adaptability and predictable batch work wants V3's discipline.

A multi-turn healthcare support conversation needs both at once — conversational *and* predictable — which is why this system landed in the territory his Part 3 calls hybrid **adaptive planning**. Every turn runs a small plan: router plus planner, one merged call in mode B. Then the plan executes mechanically — validated, deduplicated, no agentic loop, no per-step "should I call this?" soul-searching. Per-turn cost is flat like V3. But re-planning happens *every turn*, which restores V1's adaptability: the person changes direction mid-conversation, and the next turn simply plans differently, with the session's accumulated state carried along in Redis.

Two of his findings, this build confirmed independently. Variance is architectural: he showed temperature 0 and a fixed seed still produced a ten-call spread in the agentic loop, because the model keeps making different *valid* strategic choices — the fix isn't sampling settings, it's moving decisions out of the runtime loop, which is precisely what plan-then-execute does. And his V3 observation that most workflow steps don't need LLM reasoning at all: lookups, cache checks, gates, and note-building here are mechanical; only intent understanding and narration genuinely need a model.

## The bug that lived in three layers

The multi-subtask collision from Part 2 deserves its full telling, because its fix taught me something about bugs in general. A user asks about two orders in one message. The plan produces two calls into the order domain. And the results — keyed by domain alone — overwrite each other. One answer to two questions.

The fix landed in three places: plan-level dedup by a `domain+tool+args` fingerprint, execution-level results keyed by `(domain, tool)`, and narration reading results per-call rather than per-domain. Three layers for one bug — and that's the lesson. When a fix needs the same change in three layers, the bug was never a line of code. It was an *assumption* — "one call per domain per turn" — quietly baked in everywhere, waiting.

## The subsystem I didn't build

Not every turn needs tools. "What was that order number again?" — the answer is sitting right there in conversation history. My first design for this was a whole separate answering subsystem, and it got pushed back on, correctly. The history was *already in the prompt*; the only gap was that the planner didn't know it was allowed to skip tools. The shipped fix is an `answer_from_context` flag, with narration-with-history as the safety net so a misfired flag degrades gracefully instead of failing.

I keep the memory of that review around as a warning: over-engineering usually announces itself as a new class.

## Notes written by code

Every conversation produces a structured note — for the human agents who pick up escalations, and for the audit trail, which in healthcare support matters clinically as well as operationally. The design choice: the note is built **deterministically from tool results**. Not generated. Not summarized by the model. Two rules govern it: facts come from tool outputs, so the note cannot hallucinate, because no generation is involved; and user vocabulary only — order IDs and names, never internal DB keys. If it can't be shown to the caller, it doesn't go in the note.

It took me a while to see what this actually was: context engineering's core discipline, applied in reverse. The same care about what enters the model's context applies to what leaves the system as a record. (Part 6 tells the story of what this note does for the model itself.)

## Watching it think

MLflow traces every turn — router decision, plan, tool calls with timing, narration. One fix from this chapter is worth its line: tracing initialization moved to lazy/background, because a synchronous init on the hot path is latency the user feels. Observability must never break the latency budget it exists to measure. There's a koan in there somewhere.

## The lessons file

The compressed version, for future-me. Guarantees in code, interpretation in the LLM — never trade across that line for convenience. Latency budgets shape architecture more than any pattern article. When a fix needs the same change in three layers, the bug was an assumption. Push back on your own subsystems; the flag was better than the class. Deterministic wherever possible — notes, gates, limits — generative only where language actually varies. And multi-file changes ship together, grep-verified, or they ship as mysteries.

## References

- Chris Hughes, *From Orchestrator to Goal-Aware* (Part 2 — orchestration variance): https://chris-hughes10.github.io/posts/multi-agent-part2/
- Chris Hughes, *Planning Multi-Agent Workflows* (Part 3 — planner + DAG, hybrid adaptive planning): https://chris-hughes10.github.io/posts/multi-agent-part3/#hybrid-approaches
- Anthropic, *Building effective agents*: https://www.anthropic.com/research/building-effective-agents
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- MLflow LLM tracing: https://mlflow.org/docs/latest/llms/tracing/index.html
- Model Context Protocol: https://modelcontextprotocol.io

*Next: Part 6 — the context window deserved its own post: a forgetting bug, a naive fix, a U-shaped curve, and the note that can't be evicted.*
