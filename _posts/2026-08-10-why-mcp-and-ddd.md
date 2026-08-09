---
layout: post
title: "Part 1: Why MCP + DDD"
date: 2026-08-10 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [mcp, ddd, agents, architecture]
---

*This series is the story of building an agentic AI customer-support system for a healthcare provider — a conversational, multi-turn assistant that verifies who's talking to it, carries context across a whole dialogue, acts on accounts, and knows when to hand off to a human. It's partly a portfolio piece and partly a letter to future-me, for the day I forget how any of this works. Everything here is a generalised reference design; no company-specific or proprietary details appear.*

## Where it started

A healthcare provider's support line is a strange place. Patients call about device orders. Clinics call about their patients. Doctors call about both. Everyone arrives mid-thought, nobody states their whole request in one message, and before you can discuss *anything* of substance, you have to be sure who you're talking to — because everything behind the curtain is protected health information, and the rules around PHI are not suggestions.

That last part shaped everything. So did another realisation that took me longer to accept: the conversations are genuinely *conversations*. Identity gets established early and relied on for the rest of the dialogue. Details accumulate over turns. People correct themselves mid-stream — "actually, the other order." Confirmations happen across message boundaries. I went in thinking multi-turn support was a feature I'd add; it turned out to be the substrate everything else sits on.

The brief I set myself: build something that greets a person, verifies them, follows the thread of what they need across a whole conversation, does it, and escalates gracefully when it can't. It should feel like talking to a competent agent — not like filling in a form one field at a time.

## Build vs buy, and the answer that was neither

The first weeks were spent evaluating platforms — managed support suites, contact-centre products, chat infrastructure. The more demos I sat through, the clearer a line became. On one side of it: channels, queuing, handoff UIs, message plumbing. Platforms do this well, and building it yourself in 2026 would be a hobby, not a decision. On the other side: *our* logic. Looking up a patient. Verifying identity against our database. Order workflows with their own rules. And above all, the reasoning layer that figures out what a person actually wants.

No platform does that side. It can't — it doesn't know our domain.

So the architecture became a hybrid. The platform owns everything up to "here is what the user said, as text." From that moment, it's ours: our system does the thinking and hands back the words to say. The platform sends an event with a session ID; we send back a sentence.

There was a security bonus I didn't see coming. Because the platform handles the raw channel data, none of it ever touches our servers — our backend only ever sees clean, structured values. If we're ever breached, the attacker gets a name and a date, not the raw exchange. Smaller PHI surface, one less pipeline to secure. I'll take luck when it's shaped like architecture.

## Why MCP

The "our logic" side needed the LLM to call tools — look up a patient, verify identity, fetch an order, create one. The default move is ad-hoc function calling: a list of functions pasted into a prompt somewhere in application code. I went with the Model Context Protocol instead, and the reasons compounded over time.

Tools live in servers, not in prompt-assembly code — each MCP server owns its tools, their schemas, and the data behind them, and the host discovers them rather than hardcoding them. The server boundary is a real protocol boundary, so servers can be built, tested, and deployed independently; when one domain's workflow started changing weekly (it did), only that server changed. And — the actual point of this post — MCP maps almost suspiciously well onto Domain-Driven Design.

## Why DDD

DDD shaped the system from day one, and Chris Hughes' *Architecting Multi-Agent Systems* series convinced me the mapping was real rather than fashionable. His core argument: agentic systems have the same concerns as any complex application, so the same layered discipline applies — and he proves it with a working build, not a diagram.

But honestly, the codebase became its own best argument. The system is organised as bounded contexts, and each bounded context is an MCP server:

```
src/
├── client/            ← the host (router, planner, orchestrator, narration)
└── servers/
    ├── verification/  ← identity: lookup, verify
    ├── patient/       ← patient account domain
    ├── order/         ← device orders: lookup, create, cancel
    ├── clinic/        ← clinic/provider domain
    └── kb/            ← knowledge base / product questions
```

Each server speaks only its domain's language and touches only its domain's data. The verification server doesn't know orders exist. The order server doesn't know how identity was established — it receives an already-verified context, injected by the host, and trusts it.

Here's what I love about this arrangement: the bounded context isn't a folder convention someone can quietly violate at 6pm before a demo. It's a *process*. You physically cannot reach across it except through declared tools. DDD's promise, made load-bearing.

## The road not taken

The pattern literature kept offering me a tempting alternative: one big MCP endpoint, free-form requests, a mega-server that figures it out. The "simple" option. I read those articles carefully and walked away, because that pattern erases exactly what DDD had bought me. Explicit tools-per-domain means the action space is enumerable — I can audit what the system *can* do. Small tool sets keep every routing decision cheap, which matters when response time is a product feature. And domain isolation contains blast radius: a bug in order creation cannot corrupt verification, because it cannot reach it.

What I did keep from that literature was the shape of the whole system — the **Router + Tool Groups** stack. An LLM router classifies intent into a domain (or out-of-scope, or small talk); a planner turns intent into tool calls; domain servers execute. That's the machine, and Part 2 opens it up.

One benchmark from Hughes' series haunted the early design in a useful way: a free-running agentic orchestrator produced somewhere between 17 and 34 LLM calls for the *same request*, run to run — variance that survives temperature 0, because it's architectural, not statistical. For a latency-bound conversation, that unpredictability is disqualifying. It's why this system commits, from day one, to plan-then-execute rather than letting a model in a loop decide the workflow as it goes. That trade-off gets its full story in Part 5.

## What I'd tell past-me

Don't build the channel layer; build the reasoning layer — all the differentiating value lives there. Decide early what the vendor platform is allowed to see and log, because PHI in someone's flow-variable debug logs is a real incident waiting to happen — turn it off, and get the BAA sorted before the architecture debate, not after. And DDD is not ceremony when your bounded contexts are literal processes. It's the cheapest insurance you'll ever buy against a codebase turning into soup over months of iteration. I know, because months of iteration is exactly what came next.

## References

- Chris Hughes, *Architecting Multi-Agent Systems*: https://chris-hughes10.github.io/posts/multi-agent-part1/
- Model Context Protocol: https://modelcontextprotocol.io

*Next: Part 2 — MCP under the hood: one conversational turn end-to-end, and what multi-turn actually demands.*
