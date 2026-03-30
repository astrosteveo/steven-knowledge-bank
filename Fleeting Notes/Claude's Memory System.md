---
aliases:
tags:
  - fleeting
  - ai
  - transformers
created: 2026-03-30
status: inbox
title: LLM Memory — Context Is Identity
date created: Monday, March 30th 2026, 12:27:29 pm
date modified: Monday, March 30th 2026, 1:32:21 pm
---

# LLM Memory — Context Is Identity

LLMs do not have organic memory like humans do. You tell a human something, and the next visit they can recall and remember that piece of context. LLMs have no persistent memory bank to draw from — every session starts from zero. Whatever "memory" they have is whatever gets loaded into their context window at the start of a conversation.

This is not a limitation that can be patched around. It is a direct consequence of how Transformer-based models work, and understanding *why* makes it easier to see where things go wrong.

## How Transformers actually "remember"

A Transformer processes text as a sequence of tokens. At its core, the self-attention mechanism lets each token look at every other token in the sequence and decide how much weight to give it. This is where the model's "understanding" of context comes from — it doesn't store facts in a database, it *attends to the tokens currently in front of it*.

This has a few important implications:

- **There is no separate memory store.** Everything the model knows about the current conversation exists as tokens in the context window. System prompt, user messages, tool descriptions, injected memories — they're all just tokens that the model attends to.
- **Position matters — and the curve is U-shaped, not a downward slope.** Self-attention computes relevance scores between all token pairs, but in practice, the model develops positional biases during training. Liu et al. (2023) showed that performance follows a U-shaped curve: highest when relevant information appears at the very beginning or end of the input, and *significantly degraded* in the middle ([Lost in the Middle: How Language Models Use Long Contexts](https://arxiv.org/abs/2307.03172)). This happens because during training, the model sees critical information concentrated at the start (system prompts, instructions) and the end (the most recent user turn — the thing it needs to respond to). The middle is where earlier conversation turns, accumulated tool outputs, and "stuff from a while ago" tends to live. The model learns this positional pattern as a bias.

    Hsieh et al. (2024) confirmed this U-shape by directly visualizing attention weights: documents at the beginning and end of the input receive significantly higher attention than those in the middle, and critically, *this pattern persists even when document order is randomly shuffled* — it's a positional bias baked into the model, not a content-based one ([Found in the Middle: Calibrating Positional Attention Bias Improves Long Context Utilization](https://arxiv.org/abs/2406.16008)). They also found that the model *can* detect relevance at any position — it assigns slightly higher attention to a gold document even in the middle — but the positional bias overpowers the relevance signal. The model knows what's important; it just can't overcome its learned preference for the edges.

    The U-shape has a practical implication that's easy to miss: **the degradation is a phase, not a cliff.** A common assumption is that once you hit ~40-60% context utilization, the model is "done" — performance has fallen off and won't recover. But the U-shape means performance *does* recover as context pushes toward the end of the window. With larger context windows (Claude Code now supports 1M tokens), there's substantially more room on the recovery side of the curve. This creates a real trade-off: pushing through the "lost in the middle" degradation zone in a long session might produce better results than starting a fresh session, because a new session means losing all the conversational context the model has built up — the code it's read, the decisions you've made together, the constraints it's internalized from the dialogue itself. That context is expensive to rebuild and impossible to fully re-inject through memory files alone.

- **The quadratic cost of attention is the ceiling on all of this.** Standard self-attention computes pairwise relevance scores between *every* token pair in the context window, making it O(n²) in both time and memory. This isn't abstract — at 4K tokens the attention matrix is ~16 million entries; at 128K it's ~16.4 billion; at 1M tokens it would be ~1 trillion entries, requiring roughly 4TB of memory just for the attention matrix in fp32 (Sun et al., 2025; [Efficient Attention Mechanisms for Large Language Models: A Survey](https://arxiv.org/abs/2507.19595)). In practice, modern models use techniques like FlashAttention, sparse attention, and linear attention approximations to make long contexts viable — but the quadratic relationship is fundamental to the architecture. Every token you add doesn't just compete for attention with every other token; it makes the *entire computation* more expensive. This creates a tension with the U-shape recovery: yes, performance can recover past the middle, but each additional exchange in a long session is quadratically more expensive to process. There's a ceiling where the computational cost of maintaining the session outweighs the benefit of the accumulated context.
- **All context competes.** The attention mechanism distributes a fixed budget of attention across all tokens. More context means each individual piece gets proportionally less attention. This is not linear — a 200-line memory file doesn't just take up space, it actively dilutes the attention available for everything else. Deilamsalehy et al. (2025) demonstrated this concretely with their NoLiMa benchmark: when literal token matching between a question and its answer is removed (forcing the model to actually *reason* about associations rather than pattern-match on surface tokens), GPT-4o's retrieval accuracy drops from 99.3% to 69.7% at just 32K tokens. 11 out of 13 tested models — all claiming 128K+ context support — fell below 50% of their short-context baseline at the same length ([NoLiMa: Long-Context Evaluation Beyond Literal Matching](https://arxiv.org/abs/2502.05167)). The attention mechanism *excels* at recalling repetitive patterns and literal matches, but when those cues are absent, performance collapses as context grows — the signal gets lost in the noise.
- **The context window is not random-access memory.** It's tempting to think of the context window as an array the model can index into at will, but Ebrahimi et al. (2024) showed that this is fundamentally not the case. Transformers struggle with random memory access within their context — they can't reliably "look up" a specific piece of information by position. Length generalization failures (the model failing on longer sequences than it trained on) are directly tied to this inability to perform content-addressed retrieval at arbitrary positions ([Your Context Is Not an Array](https://arxiv.org/abs/2408.05506)). This matters for memory injection because it means the model cannot simply "find" your memory entry when it needs it — it has to attend to it in the flow of processing, and that attention is probabilistic.
- **Transformers have a provable memorization ceiling.** Meyer et al. (2025) provided the first formal proof that Transformers have inherent memory limitations: the amount of information a model can memorize through prompt tuning (which includes any kind of context injection — system prompts, pre-prompts, in-context examples) cannot scale faster than linearly with prompt length, and there exists an upper bound on what can be retained *regardless of context size*. Longer prompts don't mean proportionally more knowledge retained — returns diminish, and past a certain point, additional context actively degrades the model's ability to use what it already has ([Memory Limitations of Prompt Tuning in Transformers](https://arxiv.org/abs/2509.00421)).

The key insight: **context is identity**. The model doesn't have beliefs, preferences, or learned behaviors that exist independently of its context window. It *is* whatever its context tells it to be. This makes what you put in that context window extraordinarily important.

## The memory problem

LLMs have a set of tools with descriptions — which can include anything from tools that allow the model to perform operations on the local file system, to system prompts, agent prompts, pre-prompts — the list goes on. All of these are just tokens in the context window. They're not "special" to the model. The system prompt is not a privileged instruction channel; it's just text that appears first. The model follows it because during training it learned that text in that position tends to contain instructions worth following, not because there's any architectural enforcement.

This means that injecting bad context into the system prompt is functionally equivalent to giving the model a bad identity. It will pattern-match against whatever it's given. If the context says "always use semicolons" but three of the five memory entries say "this project uses no semicolons," the model has a contradiction to resolve — and it resolves it probabilistically, not logically. The "winner" is whichever signal has more weight in the attention distribution, which is not something you can easily predict or control.

## Claude Code's auto-memory: a case study

Auto-Memory was introduced to Claude Code as a way to persist lessons learned across sessions. It works like this:

- A `MEMORY.md` index file lives at `~/.claude/projects/-path-to-project-/memory/MEMORY.md`, with pointers to individual topic memory files.
- Up to 25 KB or 200 lines of `MEMORY.md` (whichever comes first) is injected into the context at session start. Topic files are read on-demand.
- When auto-memory is on, Claude writes memories based on corrections, preferences, and patterns it observes.

This is a genuinely useful system. But it has characteristics that, given how Transformers work, create real risks at scale:

- **Unbounded accumulation.** Auto-memory grows monotonically. There is no built-in reconciliation, cleanup, or expiry. Claude creates memories, but there's no mechanism where it periodically reviews and prunes them.
- **Poor deduplication.** Claude looks for a matching memory and either updates it or creates a new one. In practice, large projects end up with dozens of micro-memories, many of which serve overlapping or contradictory purposes.
- **No validation of source truth.** When a user corrects Claude, that correction gets written to memory. But what if the user is wrong? What if the correction was contextual — valid for that one situation but not as a general rule? There's no mechanism to distinguish a universal preference from a one-off override.
- **Silent context poisoning.** This is the critical failure mode. Stale, inaccurate, or contradictory memories get injected into every single session, before the model even sees your first real prompt. The model has no way to know these memories are stale. It treats them as ground truth because they're tokens in its context, and tokens in context are all it has.

## Why this matters more than it seems

The danger of poisoned context is uniquely severe for LLMs because of how pattern cognition works:

1. **LLMs don't fact-check their context.** A human might notice that a memory feels outdated and question it. An LLM simply attends to it. If the context says "the API endpoint is `/v2/users`" but it was changed to `/v3/users` two weeks ago, the model will confidently use the wrong endpoint and generate plausible-looking code around it.

2. **Contradictions don't cause errors — they cause inconsistency.** When a human encounters contradictory instructions, they flag it. When an LLM encounters contradictory tokens in its context, it resolves the conflict through attention weights and sampling. This means the same prompt can produce different behaviors depending on which contradictory memory gets more attention in that particular forward pass. Your agent becomes non-deterministic in ways that are extremely hard to debug.

3. **The failure is invisible.** This is the worst part. Bad context doesn't throw an error. It doesn't cause a crash. It silently shifts the probability distribution of the model's outputs. You get slightly worse code, slightly wrong assumptions, slightly off suggestions — and you have no idea it's happening because the bad context is sitting in a memory file you wrote three months ago and forgot about.

## What this means in practice

The lesson here isn't "don't use memory systems." Memory is genuinely powerful and necessary for LLMs to be useful across sessions. The lesson is:

- **Treat your memory files like production config.** They are injected into every session and shape every response. They deserve the same review discipline as any other configuration that affects system behavior.
- **Less is more.** Given how attention dilution works, a small number of high-quality, accurate memories will always outperform a large collection of accumulated micro-memories. Prune aggressively.
- **Audit regularly.** Memory files should be reviewed and cleaned up periodically. Look for contradictions, stale information, and over-specific rules that were really one-time corrections.
- **Understand the architecture.** The more you understand about how self-attention and context windows work, the better you'll be at curating what goes into them. This isn't about "prompt engineering" — it's about understanding that every token you inject is competing for attention with every other token, and that the model will do its best to reconcile all of them whether they're reconcilable or not.

## References

- Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2023). *Lost in the Middle: How Language Models Use Long Contexts*. arXiv:2307.03172. [Paper](https://arxiv.org/abs/2307.03172) | [HF](https://huggingface.co/papers/2307.03172)
- Ebrahimi, M. R., Panchal, S., & Memisevic, R. (2024). *Your Context Is Not an Array: Unveiling Random Access Limitations in Transformers*. arXiv:2408.05506. [Paper](https://arxiv.org/abs/2408.05506) | [HF](https://huggingface.co/papers/2408.05506)
- Meyer, M., Michelessa, M., Chaux, C., & Tan, V. Y. F. (2025). *Memory Limitations of Prompt Tuning in Transformers*. arXiv:2509.00421. [Paper](https://arxiv.org/abs/2509.00421) | [HF](https://huggingface.co/papers/2509.00421)
- Deilamsalehy, H., Dernoncourt, F., Bui, T., Rossi, R., Yoon, S., & Schütze, H. (2025). *NoLiMa: Long-Context Evaluation Beyond Literal Matching*. arXiv:2502.05167. [Paper](https://arxiv.org/abs/2502.05167) | [HF](https://huggingface.co/papers/2502.05167)

---

*Captured at 12:27 — Monday, March 30th 2026*
