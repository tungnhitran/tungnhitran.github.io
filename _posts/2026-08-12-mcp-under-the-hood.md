---
layout: post
title: "Part 2: MCP Under the Hood"
date: 2026-08-12 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [mcp, agents, multi-turn]
---

*Part 1 covered why the design is hybrid — buy the channel, build the brain. This part opens the brain up: what MCP actually is, and what happens in the half-second between a user's message and the system's reply.*

## MCP in one paragraph

The Model Context Protocol is a client–server protocol for connecting LLM applications to tools and data. Three roles matter. A **server** exposes tools — functions with JSON schemas — and owns the data behind them. A **client** connects to a server, discovers its tools, and invokes them. A **host** is the application that embeds clients, talks to the LLM, and decides *when* to call *what*.

The mental shift from plain function calling is subtle but real: tools aren't a list you paste into a prompt from your app code. They live with their server, get discovered at connect time, and the server is a genuine process boundary. That shift is what lets the DDD story of Part 3 happen at all.

## The topology

```
User message ⇄ Channel platform
                     │ text + session metadata (event)
                     ▼
              ┌─────────────┐
              │    HOST     │  router → planner → orchestrator → narration
              │ (src/client)│
              └──────┬──────┘
      MCP     ┌──────┼──────────┬──────────┬─────────┐
              ▼      ▼          ▼          ▼         ▼
        verification patient   order     clinic     kb
          server     server   server     server   server
```

Five servers, one host. Each server registers a handful of tools — the order server, for instance, exposes `lookup_order`, `create_order` (split into preview and submit, a story for Part 5), and `cancel_order`.

## One turn, end to end

Follow a message through the machine.

The platform delivers the user's text plus session metadata. The host loads the session's `AccessContext` and `SlotState` from Redis — so before any model runs, the turn already knows who's verified and what the conversation has collected so far. This is the difference between multi-turn and a series of one-shots wearing a trenchcoat.

The **router** — one LLM call — classifies intent into a domain, or `out_of_scope`, or `small_talk`. Small talk earning its own intent was a late fix with an embarrassing origin: without it, "how's your day going?" would fall through to a domain and trigger a patient lookup. Some lessons you only learn by watching the logs.

The **planner** turns intent into tool calls — domain, tool, arguments. Every plan passes through `_validate_plan`, which checks calls against the discovered schemas and deduplicates them by a `domain+tool+args` fingerprint. That fingerprint exists because of a real bug: a user asking about "my order and my other order" once produced colliding calls whose results silently overwrote each other. The user got one answer to two questions and no indication anything was wrong.

The **orchestrator** executes the plan through the MCP clients, collecting results keyed by `(domain, tool)` — not just by domain, for the same reason. One domain can legitimately be called twice in a turn; the result store has to be able to say so.

Finally, **narration** — the second LLM call — turns structured results into a short, natural reply. Results in, sentence out. The model never sees raw database rows, only curated facts. What it doesn't see, it can't leak.

Two LLM calls per turn, in the merged fast path I call mode B — because users notice slow replies, and every extra round-trip is silence. A two-stage debuggable path (mode A) lives behind the same interface, swappable by config when I need to watch the machinery think.

There's a quieter benefit to this structure than speed: **predictability**. Hughes benchmarked a classic agentic-loop orchestrator at 17–34 LLM calls for the same request — the model deciding the workflow at runtime is high-variance, and temperature 0 doesn't help because the variance is architectural. Here, the LLM plans and code executes. Tool calls happen because the validated plan says so — never because a model in a loop decided to try one more thing. Every turn costs the same two calls, every time.

## What multi-turn actually demands

Single-shot Q&A hides most of the hard problems. Multi-turn drags them into the light.

**Slot filling across turns.** "I'd like to order a monitor." — "Which clinic is this for?" — "The one on my file." Arguments accumulate in `SlotState` turn by turn until a plan is executable; the planner sees the new message *and* everything already collected.

**Reference resolution.** "Cancel *that* one" only means something because the session remembers which order was last discussed. History rides along in the narration prompt, and an `answer_from_context` path lets the planner skip tools entirely when the answer is already on the table.

**Mid-conversation correction.** People change their minds, constantly. Because every turn re-plans from current state, a correction isn't an error path — it's just the next plan.

**Confirmation as a conversation state.** Writes need an explicit yes across a turn boundary. The host tracks pending confirmations in session state, and repeated unclear replies escalate to a human rather than looping forever.

## Designing tools for conversation

A few rules, each purchased with debugging time.

**Return facts, not dumps.** A tool that returns a full record forces the narration prompt to carry junk and risks the model reading an internal ID back to the user. Tools return the minimal fields narration needs.

**Declare your events.** `lookup_order` originally returned an empty `{}` on a miss. Narration, handed nothing, improvised — confidently. Declaring an explicit `order_not_found` event turned improvisation into a deterministic, helpful reply: "I couldn't find that order — is there anything else on your account I can help with?"

**Keep tools fast.** Indexed lookups, entity caching (`order:<id>` plus index keys like `order_by_patient:<owner>`), no chained slow calls. Sub-second replies are a product requirement, not a nice-to-have.

**User vocabulary only.** Nothing that reaches narration may contain internal DB keys. If the user can't say it, the system shouldn't either.

And one rule taken straight from Hughes: **tools are the LLM interface; services are the business logic.** A tool function accepts simple typed arguments, calls the domain service, formats a compact result — that's all. Configuration, error handling, caching, and DB access live in the service layer beneath, testable without any LLM in the loop. When a workflow changed, the service changed; the tool signatures mostly didn't.

## Why MCP earned its keep

The honest test was change frequency. The order workflow changed constantly — pending limits, approval webhooks, the preview/submit split — and every one of those changes stayed inside `servers/order/`. The host never moved. The other four servers never noticed. That containment is the protocol boundary paying rent, month after month.

It's also the perfect on-ramp for the next post, because somewhere along the way I realised what these servers actually were: the most literal implementation of DDD bounded contexts I'd ever worked with.

## References

- Model Context Protocol — spec and docs: https://modelcontextprotocol.io
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- MCP Python SDK: https://github.com/modelcontextprotocol/python-sdk
- Chris Hughes, *Architecting Multi-Agent Systems* (Part 1 — tools vs services): https://chris-hughes10.github.io/posts/multi-agent-part1/

*Next: Part 3 — MCP × DDD: bounded contexts as processes.*
