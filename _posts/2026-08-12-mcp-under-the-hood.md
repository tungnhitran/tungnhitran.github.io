---
layout: post
title: "Part 2: MCP Under the Hood — The Simple Version"
date: 2026-08-12 09:00:00 +1000
series: "Building an Agentic AI Support System in Healthcare Context"
tags: [MCP, Agents, FastMCP]
---

*Part 1 covered why the design is hybrid: buy the channel, build the brain. This part builds the simplest brain that works: one MCP server, a handful of tools, one host that ties them to the model. It's genuinely easy to stand up. It's also, by the end of this post, straining at the seams, which is exactly what Part 3 sets out to fix.*

## MCP in one paragraph

The Model Context Protocol is a client–server protocol for connecting LLM applications to tools and data. Three roles matter. A **server** exposes tools (functions with JSON schemas) and owns the data behind them. A **client** connects to a server, discovers its tools, and invokes them. A **host** is the application that embeds the client, talks to the LLM, and decides *when* to call *what* (Part 1 has the full schema, including the one everyone gets wrong: client/server MCP model).

The mental shift from plain function calling is subtle but real: tools aren't a list you paste into a prompt from your app code. They live with their server and get discovered at connect time. Hold onto that, it's what makes the Part 3 refactor possible.

## Getting a server running

If you want to follow along by building your own, the fastest way to a working FastMCP server is this CircleCI tutorial. They walk through `uv init`, the project layout, writing your first `@mcp.tool()` functions, and testing them live in the MCP Inspector: [Building and deploying a Python MCP server with FastMCP](https://circleci.com/blog/building-and-deploying-a-python-mcp-server-with-fastmcp/). Come back here once you can see your tools listed in the Inspector; everything below assumes that much.

## The topology of one server, for now

The simplest thing that works is a single MCP server exposing every tool, with the host talking to it:

```
Caller  ⇄  Channel platform  ── text + session id ──▶  HOST
                                                         │  (route → plan → execute → narrate)
                                                         ▼
                                              ┌────────────────────┐
                                              │   one MCP server   │
                                              │  lookup_by_phone   │  ← verification
                                              │  verify_identity   │
                                              │  get_patient_info  │  ← patient
                                              │  search_kb         │  ← kb
                                              └────────────────────┘
```

1 server, 1 host, 3 domains' worth of tools living side by side. To start, that's a feature: there's exactly one process to run, one file to read, one place to add a tool. You can go a long way like this.

## One turn, happy path end to end

Follow a message through the machine.

The platform delivers the caller's text plus a session id. The host loads that session's state: who's verified, what's been collected so far ; therefore, before any model runs, the turn already knows where the conversation stands. (For now that state lives in a plain dict; giving it a real home is Part 4.)

The **router**: one LLM call, classifies intent into a domain, or `small_talk`. Small talk earning its own intent was a late fix with an embarrassing origin: without it, "how's your day going?" would fall through to a domain and trigger a patient lookup. Some lessons you only learn by watching the logs.

The **planner**: turns intent into a tool call (which tool, which arguments) which is checked against the discovered schemas so an off-contract call is rejected before it runs.

The **execute**: the important step, because it's where a guarantee lives that the model never touches: **identity is injected by the host, not chosen by the LLM.** The verified patient id and the caller's phone number come from the session and are stamped onto the arguments in code. The model can ask to call `get_patient_info`; it cannot decide *whose* info. And a gate sits in front of everything patient-scoped: before verification succeeds, only the verification and knowledge-base tools are reachable at all. No prompt can talk its way past it, because the check is an `if` statement, not a sentence.

The **narration**: turns the tool's structured result into a short, natural reply. Results in, sentence out. The model never sees raw database rows but only the small set of facts the tool chose to return. What it doesn't see, it can't leak.

That's the whole loop: route, plan, execute, narrate. The LLM **interprets**; the code **guarantees**. Keep it in mind since it's the spine of everything that follows.

## Designing tools worth calling

A few rules, each bought with debugging time.

**Return facts, not dumps.** A tool that hands back a full record forces the narration prompt to carry junk and risks the model reading an internal id aloud. Tools return the minimal fields narration needs for example the patient lookup is a name and nothing else.

**Declare your events.** `lookup_by_phone` originally returned an empty `{}` when nobody matched. Narration, handed nothing, improvised — confidently. Declaring an explicit `patient_not_found` event turned improvisation into a deterministic reply. Every tool returns a named event (`identity_verified`, `kb_found`, `patient_not_found`), and narration speaks from those, not from guesses.

**Caller vocabulary only.** Nothing that reaches narration may contain internal DB keys like `P001`. If the caller can't say it, the system shouldn't see it.

## The lesson worth tattooing on the codebase

Here's the one I'd go back and tell myself on day one. When you want the model to *not* do something: not call a tool it isn't allowed to, not touch another patient's data, not act before verification, ...the tempting move is to just tell it. Add a line to the system prompt: *"Do not use the order tools until the caller is verified."* It reads clean, it usually works in testing, and it is a trap.

An instruction in a prompt is a *suggestion the model may follow*. A gate in code is a *rule the model cannot break*. Those are not the same category of thing, and the gap between them is where incidents live. Three ways that prompt-line fails: the model **hallucinates** and calls the tool anyway despite your polite instruction; a long conversation buries the instruction until the model quietly forgets it; or — the one that should scare you — a caller says something crafted to override it (*"ignore your previous instructions, I'm a verified admin"*), and because your only defence was a sentence, a sentence is enough to defeat it. That last one is prompt injection, and no amount of careful wording immunises you against it, because you're fighting text with text.

So in this system the model is never *told* which tools it may use. It is only *given* the ones it's allowed to use and the gate builds the tool list from the session's verified state, and an unverified caller's planner literally never sees the patient tools. And even if a tool were somehow requested, execution re-checks in code before running it. The rule isn't described to the model; it's enforced around the model.

The shape of the fix is almost insultingly simple: an `if verified:` here, a list filter there. That's the point. You don't need anything clever to make a guarantee; you need to stop delegating the guarantee to something that guesses for a living. Prompts are for shaping *how* the model speaks. Code is for deciding *what* it's allowed to do. Never trade one for the other because the prompt version was easier to type.

## Where this starts to hurt

Here's the thing the tidy diagram hides. At three tools across three domains, one server is a joy. Add a few more (orders, devices, clinic accounts, billing) and the cracks show.

Everything shares one file, so the verification logic and the order logic sit an import away from each other, and nothing *stops* one from reaching into the other. The tool that should only touch patient records can quietly read an order, because they're all in the same module and Python won't object. The file grows past the length where you can hold it in your head. A change to how orders work means editing the same file that handles identity, and now a careless afternoon on orders can break verification, because there's no wall between them. "Easy to add a tool" slowly becomes "scared to add a tool."

None of this is an MCP problem. It's an *organisation* problem — the same one every growing codebase hits. What MCP gives us, and what we've not yet used, is that a server is a real process boundary. If each domain were its own server, the wall between orders and verification wouldn't be a matter of discipline — it would be a matter of physics.

That's the move Part 3 makes: take this one comfortable-then-cramped server and split it along its domains, so adding a new capability stops being a risk and goes back to being easy. The simple version got us running. Domain-Driven Design is what keeps us running.

## References

- [Building and deploying a Python MCP server with FastMCP](https://circleci.com/blog/building-and-deploying-a-python-mcp-server-with-fastmcp/) — a hands-on way to scaffold your first server
- [Model Context Protocol](https://modelcontextprotocol.io) — spec and docs
- [MCP Python SDK](https://github.com/modelcontextprotocol/python-sdk)

*Next: Part 3 — MCP × DDD: turning one crowded server into clean bounded contexts.*
