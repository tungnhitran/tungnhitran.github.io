---
layout: post
title: "Part 4: Scaling Up — State and Infrastructure"
date: 2026-08-20 09:00:00 +1000
series: "Building an Agentic AI Support System for a Healthcare Provider"
tags: [Redis, Docker, State, Scaling]
---

*Parts 1–3 were architecture. From here, the story is about making the thing production-shaped, starting with the least glamorous question in the system: where does state live?*

## The dictionaries had to die

Early versions kept session state in Python dictionaries, and for one process on my laptop that was fine. Then I started thinking about what happens when a container restarts mid-conversation. Or when there's more than one host process. Or when a deploy lands while someone is halfway through verifying their identity. The answer to all three was the same: the conversation evaporates, and the user starts over with a system that never met them.

So the dictionaries died, replaced by a `RedisSessionStore` whose design philosophy is best described as *aggressively boring*. One Redis hash per session, holding the serialized `AccessContext` (who's verified, what role) and `SlotState` (what the conversation has collected so far). A TTL on every session, because conversations end and their state should too — and in a PHI system, state that expires by default is a compliance feature, not just hygiene. A pipeline making `HSET` + `EXPIRE` atomic, so a crash between the two can't mint immortal keys.

The serialization choice tells you what I'd learned to value by this point. Pickle is smaller and easier. JSON means that at 2am, mid-incident, `redis-cli` shows you human-readable state. Debuggability beat elegance, and it wasn't close. (Pickle also carries deserialization risk, a bad trade in a system holding PHI.)

The access pattern is the part I'd defend hardest: **load once, save once, per turn.** A `_turn` wrapper in the host loads state at turn start and persists at turn end, and nothing in between touches the store. One seam. One place for bugs to hide, instead of dozens of scattered reads and writes each with their own opinion about freshness.

One Docker lesson from this chapter cost me an afternoon, so it goes in the record: inside a compose network, the Redis URL is `redis://redis:6379`, the *service name*. Not `localhost`. Localhost inside a container is the container, staring back at you.

## The cache grows an index

Domain servers cache entities with a two-part scheme that emerged from watching real queries. Entity keys (`order:<id>`) answer "give me this order" in one hop. Index keys (`order_by_patient:<owner>`, `order_by_clinic:<clinic>`) answer "which orders does this patient have" without a scan. And the business invariants ride the same rails: the one-pending-order-per-owner rule is a `pending:<owner>` key, checked atomically on create, cleared by the approval webhook. Cheap, deterministic, and enforced in code — not by asking the model nicely and hoping.

## The memory decision

Here's the decision from this phase I'd defend in any review. The agent-memory literature pushes two tiers: session memory (what's happened in this conversation) and long-term memory (facts extracted across sessions, "this user prefers X"). Tier 1, obviously yes. Tier 2, I rejected outright, and the reasoning is worth writing down because the pressure to add it never quite goes away.

In a healthcare system, the database is *already* the authoritative long-term memory. An LLM-extracted shadow memory of PHI is a second, unaudited store of the most sensitive data you hold — with extraction errors baked in at write time, and a brand-new attack surface thrown in for free. The right long-term memory for "does this patient have a pending order" is the order domain, queried fresh, every time. Some scaling advice is written for someone else's constraints. Knowing which advice that is turned out to be the actual skill.

## Containers, one network, one hard-won discipline

The runtime is Docker Compose: the host, the domain servers, and Redis on one bridge network. The details that earned their keep were small. A bind-mounted `./src` so editing locally shows up in containers immediately. A shared `./logs` volume so tailing one conversation across five services is a single terminal. An `x-common` env block so model keys, the Redis URL, and log levels are defined once and inherited everywhere, config drift between containers is an entire bug class, deleted. Pinned Python images, so whatever my laptop runs is irrelevant, which is the whole point of containers.

But the recurring failure that taught me the most had nothing to do with Docker and everything to do with me. **Partial merges.** A change touching the host, narration, and provider interfaces would get half-synced ; two files updated, one stale ; containers restarted, and the system would fail in ways that looked exactly like deep logic bugs. Hours went into "debugging" code that was simply two different versions of itself, running simultaneously.

The fix wasn't a tool. It was a discipline: multi-file changes sync together, verified with `grep -c` for the new symbols in *every* touched file, and only then restart. Boring. It saved more hours than any clever thing in this series.

## References

- Redis docs — hashes, TTL, pipelines: https://redis.io/docs/
- Docker Compose networking: https://docs.docker.com/compose/how-tos/networking/
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents

*Next: Part 5 — scaling the brain: what the system must guarantee that the LLM never can.*
