---
layout: post
title: "Part 3: MCP Meets DDD"
date: 2026-08-15 09:00:00 +1000
series: "Building an Agentic AI Support System in Healthcare Context"
tags: [MCP, Domain-Driven Design, Bounded Contexts]
---

*Part 2 left us with one server that worked but had started to hurt since every domain in one file, no wall between them, "easy to add a tool" curdling into "scared to add a tool." This part is the fix, and a claim I'll try to earn along the way: MCP servers are the most literal implementation of DDD bounded contexts I've ever worked with.*

## The refactor

Take the crowded server from Part 2 and split it along its domains — one server per bounded context:

```
                         ┌──────────────┐
              ┌────────▶ │ verification │  lookup_by_phone, verify_identity
              │          └──────────────┘
   ┌──────┐   │          ┌──────────────┐
   │ HOST │ ──┼────────▶ │   patient    │  get_patient_info
   └──────┘   │          └──────────────┘
              │          ┌──────────────┐
              └────────▶ │      kb      │  search_knowledge_base
                         └──────────────┘
```

Same tools, same behaviour as Part 2. But now the wall between patient logic and verification logic isn't discipline — it's a process boundary. The patient server *cannot* read an order, because the order server is a different process it can only reach through a declared tool. The thing we wanted in Part 2 — physics instead of good intentions — is now just how the system is shaped. And adding a new domain? A new folder, a new server, zero risk to the ones already running. Below is why that mapping onto DDD is so exact.

## DDD, pointed at real code

Abstract DDD talk loses people — it certainly used to lose me. Every concept below is pointed at the actual system, because that's the only way any of it ever stuck.

Start with the **bounded context**: a boundary inside which a domain model is consistent and its language unambiguous. Here, that's each folder under `servers/` — verification, patient, order, clinic, kb. "Order" means one precise thing (a device order) inside the order server. The verification server doesn't even have the word.

The **ubiquitous language** follows: design docs, code, and user-facing replies all use the same terms. When the business says "pending order," the tool event is `pending`, the cache key is `pending:<owner>`, and the reply says "pending." No translation layers. No synonyms quietly drifting apart over months until nobody's sure whether "held" and "pending" are the same thing.

An **aggregate** is the consistency unit you load and save as one thing. An order with its status, owner, and approval state is one aggregate — which is why the one-pending-order rule is enforced at the aggregate level, as an atomic check on a `pending:<owner>` key at creation time, rather than scattered across every caller that might create an order.

**Domain events** are facts the domain emits: `order_created`, `order_not_found`, `identity_verified`. In this system they're not an abstraction bolted on — they are literally what tools return to the host, and literally what narration speaks from. The event vocabulary *is* the contract between domain and conversation.

The **anti-corruption layer** shows up twice, which surprised me. First, in Hughes' framing — which shaped much of my thinking here — the tool layer itself is an ACL: it translates between "strings and simple parameters an LLM can reason about" and "rich domain objects the services work with." Every MCP tool is a thin adapter over a real service. Second, the host injects an `AccessContext` — who the verified caller is, and whether they're a patient, clinic, or doctor — into every domain call. Domains never parse raw user input for identity; they trust the injected context. Identity concerns can't leak into business domains, because they never enter them.

And **dependency injection** ties it together: the LLM provider, session store, and orchestration mode all sit behind interfaces wired at startup. Swapping the fast merged path for the slow debuggable one is config, not surgery.

## The layered architecture

Hughes' Part 1 maps classic DDD layers onto agentic systems — presentation, application, tools-as-ACL, services-as-domain, models, infrastructure. When I laid my system against his table, it matched without being forced, which I take as evidence the mapping is natural:

| Layer | Hughes' agentic mapping | Where it lives here |
| --- | --- | --- |
| Presentation | User interaction | Channel event in / reply text out |
| Application | Orchestrates workflows | The host: router, planner, orchestrator, narration |
| Anti-corruption | Tools translating LLM ⇄ domain | MCP tool functions in each server |
| Domain | Business logic | Domain services behind the tools |
| Domain model | Entities, events | `AccessContext`, `SlotState`, order aggregates, domain events |
| Infrastructure | External systems, persistence | Redis, DB clients, channel platform API |

His litmus test for boundaries is the one I now use everywhere: *if I replaced this external system, what would change?* Replace the channel vendor — only the event adapter. Replace the LLM — only the provider behind the interface. Replace Redis — only the session store. Each boundary is a replacement point, and the LLM one wasn't hypothetical: we exercised it for real when switching orchestration modes.

One deliberate divergence is worth naming. Hughes organises agents by *job* (SearchAgent, SummarizeAgent) and services by *external system* (youtube.py). This system decomposes by **business domain** on both sides — because in healthcare support, the domains (orders, verification, patients, clinics) *are* the business, not wrappers around someone else's API. Same principle, cohesion around what changes together; different axis, chosen by asking what the litmus test says would actually be replaced.

## Where MCP makes DDD physical

Here's the thing about bounded contexts in a monolith: they're a discipline. A folder structure, a linting rule, code-review vigilance. And someone can always `import` across the line at 6pm before a demo, and someone eventually will.

With each context as an MCP server, the discipline becomes physics. The boundary is a process — crossing it requires a declared tool with a schema, and there is no sneaky import. The published language is machine-checked, because tool schemas are the context's public contract and `_validate_plan` rejects anything off-contract on every single call. Context maps are visible: the host is the only place domains compose, so reading `src/client/` tells you the entire inter-domain choreography — there's nowhere else to look. And teams scale along contexts: one domain's workflow can change weekly without touching verification. Conway's Law, working *for* you, for once.

The inverse deserves saying just as plainly: **DDD is what makes a multi-server MCP system sane.** Without domain thinking, "more MCP servers" just means more places for logic to hide. The domain decomposition tells you where the boundaries belong. MCP merely — usefully, physically — enforces them.

## The real payoff: add a domain, inherit the machinery

Here's where the split stops being tidy architecture and starts paying rent, and it's the part I didn't fully appreciate until the fourth domain went in.

But first, the mechanism that makes it possible — and it's the quiet beauty of MCP. You never *register* a tool with the host imperatively. You **declare** it. Each MCP server publishes a manifest: for every tool, a name, a description, and a JSON schema of its arguments (which are required, which are optional, their types). The host doesn't hardcode any of this — it calls `list_tools` at connect time and *discovers* what each server offers.

That single fact is what makes everything below automatic. The host doesn't need to be taught that the order server has a `lookup_order` tool taking a required `order_id`. It reads that from the manifest. Here's what that actually looks like — this is (lightly trimmed) what the order server hands the host when it calls `list_tools`:

```json
{
  "name": "lookup_order",
  "description": "Look up one of the caller's orders by its ID.",
  "inputSchema": {
    "type": "object",
    "properties": {
      "order_id": { "type": "string", "description": "The order number, e.g. 1002" }
    },
    "required": ["order_id"]
  }
}
```

That's the whole contract, and no host-side code mentions `lookup_order` by name anywhere. Look how much the host gets from just those few lines:

- The **router** learns the order domain has a way to look orders up, so it can send "where's my order?" here.
- The **planner** learns the tool takes one argument, `order_id`, a string — so it knows exactly what to pass.
- **Slot-filling** reads `"required": ["order_id"]` and, when the caller hasn't given an order number, generates the question to ask for it. That behaviour comes *entirely* from that one line in the schema — I wrote no "ask for the order number" logic anywhere.
- **Narration** has the human-readable `description` to ground its phrasing.

Change the tool — add an optional `include_history` argument, say — and every one of those adjusts on the next connect, because they all read from the manifest rather than from anything hand-wired.

You describe *what a tool is* in the manifest; the host figures out *how to drive it* from that description. Declare, don't wire. A new capability is a schema entry, and the machinery meets it there.

Now the payoff that mechanism buys.

Some logic isn't *about* any one domain — it's about how the whole conversation behaves. The clearest example in this system is the escalation rule: after three unproductive turns — three failed verifications, three tool calls that got nowhere, three "I didn't catch that" — the caller goes to a human instead of looping forever. That "three strikes" counter has nothing to do with orders or patients or the knowledge base specifically. It's a property of *the host*, not of any domain.

So it lives in the host, once. The host runs every turn for every domain, counts unproductive outcomes, and trips the escalation when the count hits the threshold. And here's the payoff: **when I add a new domain, it gets three-strikes escalation for free.** I don't wire it up. I don't remember to add it. A new server shows up exposing its tools, the host routes to it like any other, and the moment that domain produces three dead-end turns, the same escalation fires — because the counter was never the domain's job in the first place.

It's not alone. A whole layer of behaviour lives in the host, defined once, applied to every turn of every domain — and inherited by each new bounded context the day it's born:

- **The verification gate** — no patient-scoped tool runs until identity is established. A new domain's sensitive tools are gated automatically, because the gate lives above the domains, not inside them.
- **Identity injection** — the verified patient id and phone are stamped onto tool arguments by the host, never chosen by the model. New tools that need them just receive them.
- **Three-strikes escalation** — three unproductive turns and the caller goes to a human. Domain-agnostic by construction.
- **Slot-filling** *(generates language)* — a tool needs an order number and the caller hasn't said one? The host notices the missing argument, holds the call, and **produces the question itself** — "Sure, what's the order number?" The domain never sees the half-formed request; it's only called once the slots are full. A new domain gets this the instant it declares a tool with a required argument, writing zero words of dialogue.
- **Confirmation before writes** *(generates language)* — before anything irreversible, the host **generates** "Just to confirm — you want to cancel order 1002, yes?" and waits for the yes across a turn boundary. Declare a write-shaped tool and it's protected on day one.
- **Narration** — turning a tool's structured result into a natural sentence is a host step, so a new domain returns facts and gets fluent replies for free.
- **Request logging & the audit trail** — every turn recorded the same way, no matter which domain served it.

Notice the two marked *generates language*. Most cross-cutting services just gate or count — they block, route, or tally. But slot-filling and confirmation go further: they **write the actual words** sent back to the caller, on the domain's behalf. That's the difference between a shared *rule* and a shared *voice*. The upshot is the same either way — a new domain server can be almost pure data access, *here are my tools, here's what they return*, and still behave like a polished conversational agent, because the parts that make it *feel* conversational (asking for what's missing, confirming before acting, escalating when stuck) were never its job to build.

This is the division that makes the system *expandable* rather than merely *organised*. DDD tells you where the domain boundaries go; putting the cross-cutting services in the host means crossing one of those boundaries — adding a whole new domain — costs almost nothing, because everything that should apply to "every conversation" already does. A monolith can do this too, in principle. But in a monolith "shared service every domain uses" and "thing any domain can accidentally reach into and break" are the same code with the same access. Here the host's services are above the domains and the domains can't touch each other — so shared behaviour is inherited, not entangled.

## The pattern we refused

All through the build, articles kept offering the single-endpoint pattern as the simple option: one mega-server, free-form requests, let the model sort it out. Viewed through DDD it's the anti-pattern in protocol form — one context, one blurred language, unbounded blast radius. We kept explicit tools-per-domain, and every incident that stayed small over the following months stayed small *because* of it.

## What the LLM changes about DDD — and what it doesn't

The genuinely novel part of putting an LLM in front of bounded contexts: the router is doing **context selection by natural language**. A user says "where's my thing," and the router's whole job is mapping that to the order context. Which makes the ubiquitous language load-bearing in a way Evans never anticipated — the router prompt describes each domain in the same terms the tools and events use, and any mismatch there becomes a misroute, directly, measurably.

What doesn't change: invariants stay in code. The LLM never enforces the one-pending-order rule, never verifies identity, never decides whether a write is allowed. Domains guarantee; the LLM interprets. Hold onto that sentence — it becomes the entire theme of Part 5.

## References

- Chris Hughes, *Architecting Multi-Agent Systems*: https://chris-hughes10.github.io/posts/multi-agent-part1/
- Chris Hughes, *From Orchestrator to Goal-Aware*: https://chris-hughes10.github.io/posts/multi-agent-part2/
- Eric Evans, *Domain-Driven Design: Tackling Complexity in the Heart of Software* (2003)
- Martin Fowler, *BoundedContext*: https://martinfowler.com/bliki/BoundedContext.html
- Martin Fowler, *UbiquitousLanguage*: https://martinfowler.com/bliki/UbiquitousLanguage.html
- Model Context Protocol: https://modelcontextprotocol.io
- Vaughn Vernon, *Implementing Domain-Driven Design* (2013)

*Next: Part 4 — making it production-shaped: where state lives, and the memory decision I'd defend in any review.*
