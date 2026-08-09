# Building an Agentic AI Customer Support System for a Healthcare Provider, Part 6: The Context Window Deserved Its Own Post

*The earlier parts treated the LLM's context as an implementation detail. This post is the story of how it became the main character — told in the order it actually happened: a forgetting bug, a naive fix, a stranger bug, a detour into attention research, and the boring little data structure that finally fixed everything.*

## Act 1: The model forgot what it just said

The bug report was almost funny. A user asks about their order; the assistant looks it up and replies with the order number. The very next message — "ok, cancel that one" — and the assistant asks, sweetly, *"Could you tell me which order you'd like to cancel?"*

It had the order number. It had *just said* the order number. One turn ago.

The cause was obvious in hindsight: each turn was effectively a fresh start. The pipeline passed the current message to the planner and the current tool results to narration, and that was it. The system had state — `SlotState`, session data in Redis — but the *model* never saw the conversation. Multi-turn on the outside, goldfish on the inside.

## Act 2: The naive fix — just feed it the history

The obvious fix is the one everybody tries first: take the chat history, paste it into the prompt, done. I did exactly that — a "conversation so far:" block, every turn, growing without bound.

It helped. "Cancel that one" started resolving. And then three new problems arrived to replace the one that left:

**It got slow and expensive.** A long support conversation meant every turn carried every previous turn. In a latency-bound system, prompt length is response time; the fix for forgetting was making the whole product feel sluggish.

**It answered the wrong message.** My prompt included both the pasted history *and* a line like "the user just said: …". With two copies of the latest message in view, the model would sometimes respond to the quoted line in isolation, ignoring the conversation it sat in. Duplicates are not emphasis; they're noise that competes with the signal.

**And strangest of all — it still forgot things.** Facts established early in the conversation (identity verified, which clinic, the order under discussion) would drop out of replies even though they were *right there in the prompt*. That one sent me to the literature.

## Act 3: The U-shape

The explanation is well documented: Liu et al.'s "Lost in the Middle" showed that language models retrieve information best from the **beginning and end** of the context, with accuracy sagging badly in the middle — a U-shaped curve. And Anthropic's context-engineering guidance frames the mechanism honestly: attention is a finite budget, and every token competes with every other token. A context window is not RAM. Presence does not imply recall.

Now the "still forgets" bug made sense. The facts that mattered most — verified identity, the active order, what's pending confirmation — were established in the *early-middle* of the conversation. As history grew, they migrated into exactly the region the model reads worst. Meanwhile the window I eventually added to cap latency made it worse in a second way: window the history, and those early turns aren't in the middle anymore — they're *gone*.

So the naive approach loses your most important facts twice: first to the attention valley, then to eviction. What a human agent does with such facts is neither: they write them down.

## Act 4: The system writes them down

The fix was almost anticlimactic — a structured **conversation note**, maintained in code:

- **Built deterministically from tool results.** No generation, no summarization call, no chance to hallucinate. When `verify_identity` succeeds, the note records it. When `lookup_order` returns, the note records the order in play. Facts enter from tool outputs only, in user vocabulary only — never internal DB keys.
- **Injected every turn as the *last* history entry** via a small `_with_note()` wrapper. Last position is the point: it sits at the favorable end of the U-curve, where recall is strongest. And because it's re-injected fresh each turn, the sliding window structurally *cannot* evict it. It is working memory that survives windowing by never depending on it.
- **One artifact, two jobs.** The same note becomes the human-agent handoff record (Part 5). The model's working memory and the audit trail are the same deterministic structure — which means they can't disagree.

Around the note, the rest of the context got disciplined too:

- **History became real chat messages, windowed.** Message-structured dialogue resolves references better than a pasted transcript blob; the window caps cost and latency; the note carries whatever the window drops.
- **The re-quote came out.** One copy of everything.
- **Each component got its own context diet.** The router sees the message and short domain descriptions — no tool schemas. The planner sees `SlotState` plus only the routed domain's tools. Narration sees labeled facts with declared events (`order_created`, `order_not_found`) — never raw rows. The guiding question for every prompt became: *what is the minimum context that makes this decision correctly?*
- **And a fast path fell out for free:** with history and the note reliably in view, "what was that order number again?" needs no tool call at all. An `answer_from_context` flag lets the planner skip tools; narration-with-history is the safety net if the flag misfires.

## Epilogue: it improved — and the moral

Reference resolution stopped failing. Early-conversation facts stopped vanishing. Turns got faster, not slower, because the prompt carried a compact note instead of an ever-growing transcript. And the improvement was measurable in the dullest possible way: fewer clarifying questions the user had already answered.

Looking back across the whole series, almost every "LLM bug" in this project was a context bug wearing a costume. The forgetting bug was missing history. The wrong-message bug was a duplicate. The still-forgetting bug was the U-curve plus eviction. The improvisation and false-promise bugs (Part 5) were undeclared events and over-licensed prompts. The fix was never a better model, and rarely better prompt *wording* — it was restructuring what entered the context, in what shape, at what position.

If I compress the series' thesis one more level: **DDD decides where logic lives; MCP enforces it; context engineering decides what the model gets to know. All three are boundary disciplines.**

## References

- Nelson F. Liu et al., *Lost in the Middle: How Language Models Use Long Contexts* (2023): https://arxiv.org/abs/2307.03172
- Anthropic, *Effective context engineering for AI agents*: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- Anthropic, *Building effective agents*: https://www.anthropic.com/research/building-effective-agents
- Chris Hughes, *Architecting Multi-Agent Systems*: https://chris-hughes10.github.io/posts/multi-agent-part1/
- Model Context Protocol: https://modelcontextprotocol.io
