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

As with other things in silicon, it's something we have to simulate. We have to provide mechanisms the model can use to simulate some sort of memory persistence. In `*.md` files is a standard convention that has been popularized. It is native to what LLM's is trained on (large amounts of text data, often just sourced right from the Internet), and provides low overhead with read tools.

This is not a limitation that can be patched around, although it is something that we've been trying to do ever since the first time we interacted with LLM's and started wondering "How do I get this thing to remember what we just talked about?"

For those who have used LLM's ever since they were only chat bots in a website remember you would type a prompt to the LLM and it would spit out some code. You would then have to copy the code, create the file, and paste the code into the file and save. The problems would arise when you would hit that small 32K context window and the chat would need to be restarted. Now, if you asked the LLM to extend on that file you had to copy the file, and paste it back to the LLM as context. Eventually, the codebase would get so large you could not fit the code and expect a full response back out. Messages would be fragmented, because you'd hit the model output token limit as well. It was a constant back and forth.

Then we finally got actual tools where the model could read the filesystem, and that solved the dilemma of copy and pasta back and forth, but there was another problem that we're still yet to successfully crack today, even though we are very close. 

Claude has one of the more robust, dynamic memory systems and that's exactly what I'll talk about here in this document, but before I do I must preface this by stating the following:

> It is a direct consequence of how Transformer-based models work, and understanding *why* makes it easier to see where things go wrong. It is by design, and later you'll find out why, and why it's so important to keep prompts to a minimum. Your LLM doesn't need to know every thing you talked about last session.

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

The lesson here isn't "don't use memory systems." Auto-memory is genuinely powerful and necessary for LLMs to be useful across sessions. The lesson is that **auto-memory left unchecked will degrade your agent's performance over time**, and most people won't notice because the degradation is silent.

From firsthand experience on an active project over a weekend: Claude created well over a dozen memory files. That's not unusual — it's the expected behavior on any project with sustained activity. Each correction, each preference stated, each pattern observed gets its own memory or appended to an existing one. The accumulation is fast.

### The full attack surface

Memory files are the most obvious vector for context pollution, but they're not the only one. Every piece of persistent context that gets injected into the system prompt is a potential source of stale, contradictory, or diluting information:

- **`~/.claude/projects/<project>/memory/`** — Auto-memory files. These accumulate fastest and have no built-in cleanup. On active projects, you can hit dozens of files within days. Each one competes for attention with everything else in the context.
- **`.claude/rules/`** — Project rules files. These are typically written intentionally, but they can go stale too. A rule written for an early architecture decision that's since been reversed is just as poisonous as a bad memory — maybe more so, because rules feel "official" and rarely get revisited.
- **`CLAUDE.md` files** — Both project-level and user-level (`~/.claude/CLAUDE.md`). The global user CLAUDE.md is particularly sneaky because it applies to *every* project. A preference you set six months ago for a different codebase is now injected into every session across all your work. If it contradicts a project-level rule, the model resolves the conflict probabilistically — and you'll never see it happen.
- **Nested and inherited CLAUDE.md files** — Claude Code walks up the directory tree and loads CLAUDE.md files from parent directories. In monorepos, you can inherit instructions from other teams' directories that have nothing to do with your work.

All of these sources contribute tokens to the context window. All of them compete for attention. All of them can contain stale or contradictory information. And none of them have built-in expiry, reconciliation, or conflict detection.

### What to do about it

- **Treat your memory and rules files like production config.** They are injected into every session and shape every response. They deserve the same review discipline as any other configuration that affects system behavior.
- **Less is more.** Given how attention dilution works, a small number of high-quality, accurate memories will always outperform a large collection of accumulated micro-memories. Prune aggressively.
- **Audit regularly.** Sweep your memory directory, your rules, and your CLAUDE.md files periodically. Look for contradictions, stale information, duplicates, and over-specific rules that were really one-time corrections. This is especially important after periods of high activity where auto-memory accumulates fastest.
- **Watch your global CLAUDE.md.** `~/.claude/CLAUDE.md` applies everywhere. Keep it minimal and genuinely universal. Anything project-specific should live in the project's own CLAUDE.md or rules.
- **Understand the architecture.** The more you understand about how self-attention and context windows work, the better you'll be at curating what goes into them. This isn't about "prompt engineering" — it's about understanding that every token you inject is competing for attention with every other token, and that the model will do its best to reconcile all of them whether they're reconcilable or not.

### An idea: memory reconciliation as a skill

There's a gap in the current system: Claude can *create* memories but has no built-in mechanism to *reconcile* them. No periodic review, no deduplication pass, no conflict detection, no expiry. The result is predictable — memories accumulate monotonically until a human manually intervenes.

A potential solution: a Claude Code skill (or slash command) that prompts Claude to perform a memory reconciliation. Something like `/reconcile-memory` that would:

- Read all memory files in the project's memory directory
- Identify duplicates and near-duplicates (memories serving the same context)
- Flag contradictions (memory A says "use semicolons," memory B says "no semicolons")
- Identify stale memories (references to files, functions, or patterns that no longer exist in the codebase)
- Merge overlapping memories into consolidated entries
- Remove or archive memories that are no longer relevant
- Optionally do the same sweep across `.claude/rules/` and CLAUDE.md files

This would give users a way to periodically "defragment" their context — keeping auto-memory's learning benefits while preventing the slow accumulation of toxic context that degrades every future session. The skill could even be hooked into a schedule or run automatically after a certain number of sessions or memory writes.

## Open questions: primacy vs recency at scale

The U-shaped attention curve raises a natural question: if you don't care about costs or response time — say you're using Claude Code with a 1M token context window at flat-rate pricing — and you just want the best raw output quality, how does performance at the beginning of the context compare to performance at the end?

The existing research suggests the U-shape is **not symmetric** — primacy and recency are qualitatively different:

- **Primacy tends to be more stable and reliable.** Cobbina & Zhou (2025) found that placing important context at the start of the system prompt yielded the most stable and accurate outputs, with accuracy gains of up to +6 points. The beginning of context is where the model looks for identity and rules — "who am I and what should I do" ([Where to show Demos in Your Prompt](https://arxiv.org/abs/2507.22887)).
- **Recency gets high attention but is more volatile.** Placing critical information at the end of the user message flipped over 30% of predictions *without improving correctness* in QA tasks. The end of context is where the model looks for "what do I need to do right now" — it gets strong attention, but that attention is reactive rather than grounding.
- **The optimal position is task-specific and model-size-specific.** For larger models (72B parameters), end-of-message placement sometimes overtook start-of-system-prompt. For smaller models, the beginning dominated consistently. This means any general rule about "beginning vs end" will have exceptions.
- **Transformers develop the same primacy/recency effects as human episodic memory.** Mistry et al. (2025) showed that these aren't just metaphors — transformers trained on language develop measurably the same temporal biases (primacy, recency, and contiguity) that cognitive scientists observe in human memory experiments. Recency effects are directly shaped by the magnitude of positional encoding and emerge gradually during training ([Emergence of Episodic Memory in Transformers](https://arxiv.org/abs/2502.06902)).

**The honest gap in the research: nobody has tested this at 1M token scale in agentic workflows.** The foundational studies tested at 4K-32K tokens. NoLiMa pushed to 128K. While 1M token contexts have existed since Gemini 1.5 Pro shipped in early 2024, having that context length available in sustained, tool-heavy, multi-turn coding sessions (like Claude Code's recent 1M support) is a different beast — and not something many people are running at that scale, let alone studying rigorously.

At 1M tokens, the distance between the primacy zone and the recency zone is enormous — potentially 800K+ tokens of middle. Open questions that the literature hasn't answered yet:

- Does the primacy signal survive across 800K tokens of middle, or does it decay in ways not captured by shorter-context studies?
- Does recency dominate at extreme scale simply because the beginning is so far away?
- How do the sparse attention and approximation techniques that *must* be used at 1M scale (you can't do naive O(n²) at that length) change the shape of the U-curve?
- For agentic coding workflows specifically — where 10-15% of the context is consumed by system prompts and tool descriptions before the first user message — does that initial overhead push your actual working context further into the degradation zone, or does it benefit from primacy positioning?

These are empirically testable questions, but as of early 2026, the academic literature hasn't caught up to the context lengths that production models now support. What we know about the U-shape at 32K may not extrapolate cleanly to 1M.

## References

- Liu, N. F., Lin, K., Hewitt, J., Paranjape, A., Bevilacqua, M., Petroni, F., & Liang, P. (2023). *Lost in the Middle: How Language Models Use Long Contexts*. arXiv:2307.03172. [Paper](https://arxiv.org/abs/2307.03172) | [HF](https://huggingface.co/papers/2307.03172)
- Ebrahimi, M. R., Panchal, S., & Memisevic, R. (2024). *Your Context Is Not an Array: Unveiling Random Access Limitations in Transformers*. arXiv:2408.05506. [Paper](https://arxiv.org/abs/2408.05506) | [HF](https://huggingface.co/papers/2408.05506)
- Meyer, M., Michelessa, M., Chaux, C., & Tan, V. Y. F. (2025). *Memory Limitations of Prompt Tuning in Transformers*. arXiv:2509.00421. [Paper](https://arxiv.org/abs/2509.00421) | [HF](https://huggingface.co/papers/2509.00421)
- Hsieh, C.-Y., Chuang, Y.-S., Li, C.-L., Wang, Z., Le, L. T., Kumar, A., Glass, J., Ratner, A., Lee, C.-Y., Krishna, R., & Pfister, T. (2024). *Found in the Middle: Calibrating Positional Attention Bias Improves Long Context Utilization*. arXiv:2406.16008. [Paper](https://arxiv.org/abs/2406.16008) | [HF](https://huggingface.co/papers/2406.16008)
- Sun, Y., Li, Z., Zhang, Y., Pan, T., Dong, B., Guo, Y., & Wang, J. (2025). *Efficient Attention Mechanisms for Large Language Models: A Survey*. arXiv:2507.19595. [Paper](https://arxiv.org/abs/2507.19595) | [HF](https://huggingface.co/papers/2507.19595)
- Deilamsalehy, H., Dernoncourt, F., Bui, T., Rossi, R., Yoon, S., & Schütze, H. (2025). *NoLiMa: Long-Context Evaluation Beyond Literal Matching*. arXiv:2502.05167. [Paper](https://arxiv.org/abs/2502.05167) | [HF](https://huggingface.co/papers/2502.05167)
- Cobbina, K. & Zhou, T. (2025). *Where to show Demos in Your Prompt: A Positional Bias of In-Context Learning*. arXiv:2507.22887. [Paper](https://arxiv.org/abs/2507.22887) | [HF](https://huggingface.co/papers/2507.22887)
- Mistry, D. M., Bajaj, A., Aggarwal, Y., Maini, S. S., & Tiganj, Z. (2025). *Emergence of Episodic Memory in Transformers*. arXiv:2502.06902. [Paper](https://arxiv.org/abs/2502.06902) | [HF](https://huggingface.co/papers/2502.06902)

---

*Captured at 12:27 — Monday, March 30th 2026*
