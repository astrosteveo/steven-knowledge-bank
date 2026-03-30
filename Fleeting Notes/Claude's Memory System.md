---
tags:
  - fleeting
  - general
created: 2026-03-30
status: inbox
---

# Memory



LLM's do not have an organic memory like humans do. You tell a human something, and the next visit they can recall and remember that piece of context. LLM's do not have an organic memory bank to call on so they are 
## The problem

Fundamentally LLM's have a memory problem. They have a set of tools with descriptions -- which can include any kind of tool from tools that allow the LLM to perform operations on the local file system, or just straight up system prompts, agent prompts, pre-prompts -- the list goes on. Context is their identity. It's just fundamentally how Transformers-based models work.



Auto-Memory was recently introduced to CC where the agent will save lessons learned to a file. Starting with a `MEMORY.md` file inside the `~/.claude/projects/-path-to-project-/memory/MEMORY.md`, which serves as an index file with pointers to individual memory files. With Auto-Memory, up to 25 KB or 200 lines, whichever comes first, is injected into the system prompt.

- Auto-memory auto accumulates over time. While on, when you correct Claude, it writes a memory to remember it.
- Claude just looks in memories, attempts to find a matching memory, if it does, it reads and edits it, if it doesn't, then it creates one.
- There is no mechanism where Claude remembers to reconcile, clean up, and maintain this memory.
- Large running projects end up with dozens (yes, that's right) of micro memories that Claude creates.
- It is also not very good at de-duplication by itself. You will end up with multiple memories that are serving the same context, or at least attempting to do so.

This leads to large complex projects reaching the point where toxic, stale, and inaccurate context is injected into Claude Code's system prompt. Or even just really active projects. The more active you are, the more Claude gets corrected, or the more you specify your preferences, the more times Claude creates or updates it.

This is really powerful, but it leads to poisoned context. There's no mechanism that says "The user corrected me on X so I wrote it to memory", but what if the user is wrong? Now your context is poisoned, every single session, before it receives your first real prompt.



---
*Captured at 12:27 — Monday, March 30th 2026*
