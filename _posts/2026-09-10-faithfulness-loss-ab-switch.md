---
layout: post
title: "Part 7: How to Detect LLM Hallucination Without an LLM Judge"
date: 2026-09-10 09:00:00 +1000
series: "Building an Agentic AI Support System in Healthcare Context"
tags: [LLM, Evaluation, Faithfulness, Orchestration]
---

*In part 5 we talked about 2 orchestration modes: the fast merged Mode B, the careful step-by-step Mode A and a real reason to switch between them: as the domains and tools multiply, Mode B's single merged prompt swells until the context window is bloated enough to hallucinate. I said the system should fall back to the leaner Mode A when that happens, and that **how it detects the moment to switch** was a story. So here it is. The detector turned out to be a small loss function that asks, every turn, one question: do we actually believe what we're about to say?*

## The switch we actually need

Falling back from fast-but-risky to slow-but-safe is only useful if something decides *when*. A human toggling a config flag doesn't count, this happens in the middle of live calls, thousands of times a day. And "the prompt got long" is too crude a trigger on its own: a long prompt doesn't always hallucinate, and a short one sometimes does. What I actually want to detect isn't prompt length, it's the *symptom* prompt length causes. **Drift**: the moment the fast answer stops being faithful to what the tools returned and what the caller asked. Catch the drift and I catch the hallucination, whatever its cause: bloated context, an unlucky sample, a genuinely ambiguous request. So the real question becomes: how do you measure, cheaply and automatically, whether a reply is faithful?

## The motivation: latency forces the question

Start with the constraint that shapes everything on a phone line: **latency**. A caller notices silence after about a second. So the default path has to be fast which leads to one merged LLM call that plans and answers in a single shot (Mode B). The careful path (Mode A: plan, execute, then narrate as separate, inspectable steps) is slower and safer, but you can't pay its cost on every turn or the conversation feels broken.

Fast-by-default, careful-when-needed is a good idea but only if the "when" is answered automatically, in production, without a human or a second model in the loop. That's the constraint that shaped the whole detector.

The obvious first tool to reach for is **LLM-as-a-judge**: hand the request, the tools, and the narration to another model and ask "is this faithful?" It's the standard move in 2026, it's flexible, and it would genuinely work. However it can't live here, and the reason is: **latency.** An LLM judge is another model call, hundreds of milliseconds to seconds, bolted onto a turn that's already racing a one-second silence budget. Worse, it's a call I'd have to make on *every* turn to know whether to trust the fast path (Mode B). The judge that's supposed to protect the latency budget becomes the thing that blows it. Paying for a second model to check the first defeats the point of having a fast path at all.

So the LLM judge is out, and that exclusion pins down what the metric actually has to be. Three brutal requirements, and between them they rule out almost everything on the shelf:

1. **Cheap and synchronous.** It runs on *every* turn, in production, inside the latency budget it's meant to protect, so no extra model calls. This is the requirement that kills LLM-as-a-judge.
2. **Label-free.** There's no ground-truth "correct answer" at inference time to compare against. That kills reference-based eval too: the RAGAS/DeepEval family, anything that needs a gold response.
3. **Paraphrase-proof.** "Where's my heart monitor?" and "device order O123 — dispatched" share no words and mean the same thing. Lexical overlap is synonym-blind; it would false-alarm on every natural reply.

Cheap, label-free, meaning-aware and crucially *not another LLM call*. That's a narrow target.

## The inspiration: autoencoders

The shape of the answer came from an old idea from my ML days: the **autoencoder**. An autoencoder learns by trying to *reconstruct its own input*: squeeze the data through a bottleneck, expand it back out, and measure how far the reconstruction landed from the original. The reconstruction error *is* the loss. No labels required, the input is its own ground truth.

That reframing was the unlock. I don't have a gold answer, but I have something autoencoder-shaped: the caller's **request** goes into the system, gets squeezed through tools and LLM models, and comes back out as a **narration**. If the narration is faithful, it should "reconstruct" the request, you should be able to look at what we're about to say and recognise the question it answers. The gap between them is a reconstruction error I can measure with no labels at all.

Faithfulness, in other words, is a three-way relationship:

- the **request**: what the user asked, as the planner understood it,
- the **tool_result**: what the domain servers actually returned,
- the **narration**: what we're about to reply to user.

A faithful turn reconstructs cleanly across those three. Break a link and the error rises.

![The autoencoder analogy: input/latent/output maps onto request/tool_result/narration]({{ '/assets/img/autoencoder_analogy.png' | relative_url }})

![Request to tool result to narration — the same three-part shape as an autoencoder]({{ '/assets/img/pipeline-shape.png' | relative_url }})

## First attempt: the term that was always zero

My first version followed the autoencoder instinct literally: reconstruct the narration from both sides: `loss = diff(request, narration) + diff(tool_result, narration)`. It looked clean, but when I ran it over real turns the second term was basically zero every single time, and in hindsight that's obvious: narration is generated *from* the tool result, so of course they're near-identical, the term was measuring "did the model paraphrase its input," which is always yes.  Worse, it was blind to the exact failure I most needed to catch, what I started calling **selection rot**: the planner fetches the *wrong* data, narration faithfully describes that wrong data, the two agree perfectly, and the loss cheerfully says "all good" while the caller hears a confident answer about the wrong order. The reply was faithful to the data; the data was unfaithful to the request and my loss never compared those two, so it couldn't see the gap.

## The one-line fix

The change, once the dead term made it obvious, was almost embarrassingly small, re-anchor the second reconstruction. Stop asking "does the narration match the tools" (always yes) and start asking "do the tools actually answer the request":

```
loss = diff(request, narration) + diff(tool_result, request)
```

That's the entire edit, one term's reference swapped from `narration` to `request`. Now the two terms measure genuinely different things. The first: does what we're about to say match the topic of the question. The second: did we fetch something that actually answers the question. Selection rot has nowhere left to hide, fetch the wrong order and `diff(tool_result, request)` spikes no matter how faithfully the narration then describes it.

Both terms are cosine distances between embeddings, a `rel_diff` with a single relevance floor (`_REL_FLOOR = 0.50`) so trivially-related text doesn't register as a match. Cosine, deliberately, for the paraphrase-proofing from the motivation: it reads meaning, not words. And embeddings are the trick that satisfies the whole wishlist at once, an embedding call is an order of magnitude cheaper and faster than a judge model, cheap enough to run on every turn without anyone noticing, and it needs no gold label. Meaning-aware like an LLM judge, but without being another LLM call in the hot path.
 
```python
_REL_FLOOR = 0.50          # cosine at/above which a pair counts as "aligned"
 
def rel_diff(a: str, b: str, floor: float) -> float | None:
    """cosine(a,b) >= floor -> 0 (aligned); below scales up to 1 at cosine 0."""
    sim = _cos(a, b)                       # sentence-transformer cosine, [0,1]
    if sim is None:                        # no embedding model available
        return None
    if sim >= floor:
        return 0.0
    return round((floor - sim) / floor, 3)
 
def faithfulness_loss(request, narration, tool_results, **_):
    tool_text = _tool_text(tool_results)   # flatten dict/list/str -> one string
    d_narr = rel_diff(request, narration, _REL_FLOOR)   # did we address the request?
    d_tool = rel_diff(tool_text, request, _REL_FLOOR)   # did we fetch the right data?
    loss = (d_narr + d_tool) / 2           # equal weight; both must be low
    return {"loss": round(loss, 3), "diff_narr": d_narr, "diff_tool": d_tool}
```

## The blind spot I *did* close: wrong values
 
Embedding similarity is blind to single-value swaps: "shipped Aug 4" and "shipped Aug 8" are nearly identical vectors, but one is fabricated. A wrong date, an invented order number, these are the *dangerous* hallucinations, and they sail straight past a purely semantic check. So the real reconstruction term doesn't trust cosine alone. It pairs the semantic score with a **deterministic value check**: pull every number, date, and ID out of the narration, and confirm each one actually appears in the tool results.
 
```python
_NUM_RE  = re.compile(r"\b\d[\d,]*(?:\.\d+)?\b")
_DATE_RE = re.compile(r"\b(?:\d{4}-\d{2}-\d{2}|\d{1,2}[/-]\d{1,2}...)\b")
_ID_RE   = re.compile(r"\b[A-Z]{1,4}[-_]?\d{3,}\b")     # O1001, PO-20260802-0001
 
def _values(text):
    """Checkable, fabrication-prone tokens: numbers, dates, IDs."""
    dates = {m.group() for m in _DATE_RE.finditer(text)}
    ids   = {m.group() for m in _ID_RE.finditer(text)}
    # ... numbers, minus digits already claimed by a date or ID ...
    return dates | ids | nums
```
 
Semantic similarity catches *topic* drift; the value check catches *value* fabrication. Any number the narration says that the tools never returned is a fabrication, full stop. It's the most deterministic part of the whole system and, for a clinical context, the part I trust most.

## The bug that wasn't a faithfulness bug

One case nearly cost me the whole design. A caller says "thank you," and the system responds by dumping their entire order history. Obviously broken, surely the loss should flag it?

It shouldn't, and forcing it to would have wrecked the metric. The narration was a truthful, grounded description of real order data; it scored clean, *correctly*. The failure was upstream: the **planner** decided a "thank you" warranted an order lookup. That's a planning failure, caught by a different guard, not a faithfulness one.

This is the Part 5 responsibility framework doing its job. Faithfulness answers exactly one question: *is this narration grounded in data that answers the request?* and must not be seduced into answering others. A grounded narration of a badly-chosen tool call is *supposed* to score clean. The moment one metric owns every kind of failure, it owns none of them.

## Where it lives, and the blind spot I kept

So the loop closes, and it closes exactly where Part 5 left off. Every turn, Mode B produces its fast answer; the loss function scores it with two near-free cosine terms. Under threshold, the caller hears it immediately. Over threshold, the system distrusts its own reflex and re-runs the turn in Mode A, the leaner, step-by-step path whose smaller prompts are less prone to the very hallucination the score just caught or escalates. That's the detector Part 5 promised: not "the prompt got long," but "the answer drifted," which is the symptom bloated context produces and also catches the drifts it doesn't. The A/B switch wasn't a human toggling config, it's the system watching its own reconstruction error in real time and reaching for the careful brain the moment the fast one wanders.

I'll end on the blind spot I chose to keep. There's a failure class this loss can't catch: **same entity, different attribute.** Ask about a warranty, get back the order record for the right order, same order, wrong field. Cosine sees a strong match (it *is* the right entity) and stays quiet. Every fix I tried that caught it also lit up false alarms on good turns, because the distinction lives below cosine's resolution. So I documented it as a known, bounded gap rather than bury it under a coefficient that would quietly punish good calls.

That felt like the most grown-up decision in the project. Not every gap is a bug to fix; some are limits to name, document, and watch, the difference between a system you've tuned and one you actually understand.

## References

- [Autoencoders](https://en.wikipedia.org/wiki/Autoencoder): reconstruction loss, the seed of the whole idea
- Anthropic, [*Building effective agents*](https://www.anthropic.com/research/building-effective-agents)

*That's the series for now. Seven parts, one system: a call arrives, five bounded contexts and a careful pipeline do their work, and a two-term loss sits at the end quietly asking, every single turn, "do we actually believe what we're about to say?"*
