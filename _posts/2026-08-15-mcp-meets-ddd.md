# Building an Agentic AI Customer Support System for a Healthcare Provider, Part 3: MCP Meets DDD

*Part 2 opened up the protocol. This part is about the design philosophy layered on top — and a claim I'll try to earn: MCP servers are the most literal implementation of DDD bounded contexts I've ever worked with.*

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
