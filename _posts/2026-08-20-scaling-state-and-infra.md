---
layout: post
title: "Part 4: Scaling Up: State and Infrastructure"
date: 2026-08-20 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [Redis, Docker, State, Scaling]
---

*Parts 1–3 were architecture. From here, the story is about making the thing production-shaped, starting with the least glamorous question in the system: where does state live?*

## The dictionaries had to die

Early versions kept session state in Python dictionaries, and for one process on my laptop that was fine. Then I started thinking about what happens when a container restarts mid-conversation. Or when there's more than one host process. Or when a deploy lands while someone is halfway through verifying their identity. The answer to all three was the same: the conversation evaporates, and the user starts over with a system that never met them.

So the dictionaries died, replaced by a `RedisSessionStore` whose design philosophy is best described as *aggressively boring*. One Redis hash per session, holding the serialized `AccessContext` (who's verified, what role) and `SlotState` (what the conversation has collected so far). A TTL on every session, because conversations end and their state should too, and in a PHI system, state that expires by default is a compliance feature, not just hygiene. 

The access pattern is the part I'd defend hardest: **load once, save once, per turn.** A `_turn` wrapper in the host loads state at turn start and persists at turn end, and nothing in between touches the store. One seam. One place for bugs to hide, instead of dozens of scattered reads and writes each with their own opinion about freshness.

One Docker lesson from this chapter cost me an afternoon, so it goes in the record: inside a compose network, the Redis URL is `redis://redis:6379`, the *service name*. Not `localhost`. Localhost inside a container is the container, staring back at you.

## The cache grows an index

Domain servers cache entities with a two-part scheme that emerged from watching real queries. Entity keys (`order:<id>`) answer "give me this order" in one hop. Index keys (`order_by_patient:<owner>`, `order_by_clinic:<clinic>`) answer "which orders does this patient have" without a scan. And the business invariants ride the same rails: the one-pending-order-per-owner rule is a `pending:<owner>` key, checked atomically on create, cleared by the approval webhook. Cheap, deterministic, and enforced in code — not by asking the model nicely and hoping.

## The memory decision

Here's the decision from this phase I'd defend in any review. The agent-memory literature pushes two tiers: session memory (what's happened in this conversation) and long-term memory (facts extracted across sessions, "this user prefers X"). Tier 1, obviously yes. Tier 2, I rejected outright, and the reasoning is worth writing down because the pressure to add it never quite goes away.

In a healthcare system, the database is *already* the authoritative long-term memory. An LLM-extracted shadow memory of PHI is a second, unaudited store of the most sensitive data you hold, with extraction errors baked in at write time, and a brand-new attack surface thrown in for free. The right long-term memory for "does this patient have a pending order" is the order domain, queried fresh, every time. Some scaling advice is written for someone else's constraints. Knowing which advice that is turned out to be the actual skill.

## Containers, one network, one hard-won discipline

The runtime is Docker Compose: host, domain servers, and Redis on one bridge network. Moving from local to containers surfaced a logging gotcha I didn't expect. My app configured its own logger, but once it ran under Docker, Docker's own log capture was in play too, and the two ended up fighting over the same output: every line got written twice, and the duplicates didn't line up cleanly, timestamps and the order of events drifted, so reading the log to reconstruct what actually happened during a call (the timing, which turn hit which tool) became unreliable. When you're debugging a conversation flow, a log you can't trust is worse than no log at all.

The fix was small once I understood it: stop the log record from propagating up to the root handler and attach a single explicit handler instead, so each line is written exactly once, by one owner. But it cost me a couple of days first, because a doubled, misordered log looks like a concurrency bug in your system long before you suspect it's just the logging setup.

## References

- Redis docs — hashes, TTL, pipelines: https://redis.io/docs/
- Docker Compose networking: https://docs.docker.com/compose/how-tos/networking/
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

*Next: Part 5: scaling the brain: what the system must guarantee that the LLM never can.*
