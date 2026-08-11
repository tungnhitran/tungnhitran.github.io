---
layout: post
title: "Part 1: Why MCP + DDD"
date: 2026-08-10 20:00:00 +1000
series: "Building an Agentic AI Support System in Healthcare Context"
tags: [MCP, Domain-Driven Design, Agents, Architecture]
---

![A cheerful engineer beside a friendly HOST robot whose tidy arms plug into labelled domain boxes: verification, patient, order, kb]({{ '/assets/img/hero-part1.png' | relative_url }})

*This series is the story of building an agentic AI customer-support phone system — a conversational, multi-turn assistant that verifies who's talking to it, carries context across a whole dialogue, acts on accounts, and knows when to hand off to a human.*

## Where it started

A contact centre in a healthcare context is a special place. Patients call about device orders or their records. Clinics call about their patients. Doctors call about both. Everyone arrives mid-thought, nobody states their whole request in one message, and before you can discuss *anything* of substance you have to be sure who you're talking to, because everything behind the curtain is protected health information, and the rules around PHI are not suggestions.

That last part shaped everything. So did another realisation that took me longer to accept: the conversations are genuinely *conversations*. Identity gets established early and relied on for the rest of the dialogue. Details accumulate over turns. People correct themselves mid-stream — "actually, the other order." Confirmations happen across message boundaries. I went in thinking multi-turn support was a feature I'd add; it turned out to be the substrate everything else sits on.

The brief was simple to state and less simple to build: greet a person, verify them, follow the thread of what they need across a whole conversation, do it, and escalate gracefully when it can't. It should feel like talking to a competent agent — not like filling in a form one field at a time.

## Build vs buy, and the answer that was neither

The first weeks went to evaluating platforms: managed support suites, contact-centre products, chat infrastructure. The more demos I sat through, the clearer a line became. On one side of it: channels, queuing, IVR, handoff UIs, and the message plumbing. There are plenty of platforms like that: Amazon Connect, Talkdesk, NICE CXone, Genesys. They'll all let you assemble an AI-flavoured contact centre with drag-and-drop flows, prebuilt integrations, and APIs. Crucially, they also own the parts you really shouldn't build yourself: the telephony, and the speech layer: speech-to-text on the way in, text-to-speech on the way out. Building your own STT in 2026 is a hobby, not a decision.

But there's a catch worth thinking about early: **replaceability**. The more of your actual *logic* lives inside the platform's flow builder, the more painful and expensive it becomes the day you want to switch providers. Vendor lock-in isn't a licensing line item; it's every business rule you quietly drew on someone else's canvas.

So the architecture became a hybrid, and the dividing line is clear. The platform owns everything up to "here is what the caller said, as text": the call arrives, the platform answers it, transcribes the speech, and fires a webhook to us with the transcript and a session ID. From that point it's ours: our system does the reasoning and hands back the words to say, which the platform speaks with TTS. The platform is the ears and the mouth; the brain lives on our side of the webhook, where we can test it, version it, and, if that day comes, carry it to a different provider without leaving our logic behind.

There was a security bonus I didn't see coming. Because the platform handles the raw channel data, none of it ever touches our servers — our backend only ever sees clean, structured values. If we're ever breached, the attacker gets a name and a date, not the raw exchange. Smaller PHI surface, one less pipeline to secure. I'll take luck when it's shaped like architecture.

## MCP — Model Context Protocol — our first brick

MCP, first proposed by Anthropic in late 2024, has become hard to avoid in agentic-system design, and its main pitch is plug-and-play: it lets the LLM (the *brain*) reach external resources (the *arms*) to get real work done. I'd heard about it a couple of years back but never sat down with it properly. And when I finally did, a friend and I both tripped over the same first-glance mistake. You see "MCP server" and "MCP client" everywhere and assume it's a two-part client/server setup with the LLM on one end. Actually there are *three* roles, and the one people get wrong is the Host:

1. **MCP Host** — the application you build: the user-facing program that *embeds* the LLM and owns the whole experience. It decides which servers to connect to, manages the client connections, and enforces policy. The LLM lives *inside* the host; it isn't the host.
2. **MCP Client** — a lightweight connector embedded in the host, holding a 1-to-1 connection to a single server: it discovers that server's tools and shuttles calls back and forth.
3. **MCP Server** — an independent process where the work happens: querying data, looking things up, calling out to other systems, each exposing its tools through the protocol.

That distinction turned out to matter for us. In this system the Host is our own `src/client/` code including the router, planner, orchestrator, and narration. The LLM is a provider it calls when it needs reasoning, not the thing in charge. Keep that picture; it's what makes the DDD story below click.

Our side of the line needed the LLM to call tools: look up a patient, verify identity, fetch an order, create one. The default move for that is ad-hoc function calling: a list of functions pasted into a prompt somewhere in application code. I went with MCP instead, and the reasons compounded over time.

Tools live in servers, each MCP server owns its tools, their schemas, and the data behind them, and the host discovers them rather than hardcoding them. The server boundary is a real protocol boundary, so servers can be built, tested, and deployed independently; when one domain's workflow started changing weekly (it did), only that server changed. And — the actual point of this post — MCP maps almost suspiciously well onto Domain-Driven Design.

## Why DDD

*(For this part, I really want a shout-out to my senior, Sri, who instructed and led me to the MCP-DDD approach.)*

Everything became clearer for me after reading Chris Hughes' *Building Scalable MCP Servers with Domain-Driven Design*. It convinced me the pairing wasn't just tidy but load-bearing. His core argument stuck with me: an MCP tool written the naive way mixes everything together — protocol decorator, HTTP call, JSON parsing, business rule, response formatting — all in one function, and the moment requirements change ("now support a second provider," "only show severe alerts") that soup becomes the problem. DDD's answer is to organise by business capability rather than technical concern, and he makes a point I hadn't considered: this matters *more* with LLMs than with ordinary software, because a model reasons over your domain language. A tool called `get_severe_alerts()` is self-explanatory to a model; `execute_query_with_severity_filter()` makes it parse jargon. Clean domains don't just help maintainers — they help the model pick the right tool.

The codebase became its own best argument, though. The system is organised as bounded contexts, and each bounded context is an MCP server:

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

There's a second reason I avoided handing the whole workflow to a free-running agent loop, one I'll only gesture at here and pay off in Part 5: an LLM left to decide the workflow as it goes is wildly non-deterministic — the same request can fan out into very different numbers of tool calls on different runs, and that variance survives even at temperature 0, because it's architectural, not statistical. For a latency-bound conversation, that unpredictability is disqualifying. So this system commits, from day one, to plan-then-execute rather than loop-and-hope.

## What I'd tell past-me

Don't build the channel layer; build the reasoning layer: all the differentiating value lives there. Decide early what the vendor platform is allowed to see and log, because PHI in someone's flow-variable debug logs is a real incident waiting to happen. And DDD is not ceremony when your bounded contexts are literal processes. It's the cheapest insurance you'll ever buy against a codebase turning into soup over months of iteration. I know, because months of iteration is exactly what came next.
 
## References
 
- Chris Hughes, [*Building Scalable MCP Servers with Domain-Driven Design*](https://medium.com/@chris.p.hughes10/building-scalable-mcp-servers-with-domain-driven-design-fb9454d4c726) ([code](https://github.com/Chris-hughes10/mcp-ddd))
- [Model Context Protocol](https://modelcontextprotocol.io)

*Next: Part 2 — MCP under the hood: one conversational turn end-to-end, and what multi-turn actually demands.*