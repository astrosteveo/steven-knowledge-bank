---
tags:
  - fleeting
  - general
created: 2026-03-30
status: inbox
---

# Auto-Memory

Auto-Memory was recently introduced to CC where the agent will save lessons learned to a file. Starting with a `MEMORY.md` file inside the `~/.claude/projects/-path-to-project-/memory/MEMORY.md`, which serves as an index file with pointers to individual memory files. With Auto-Memory, up to 25 KB or 200 lines, whichever comes first, is injected into the system prompt.

- Auto-memory auto accumulates over time. While on, when you correct Claude, it writes a memory to remember it.
- Claude just looks in memories, attempts to find a matching memory, if it does, it reads and edits it, if it doesn't, then it creates one.
- There is no mechanism where Claude remembers to reconcile, clean up, and maintain this memory.
- Large running projects end up with dozens (yes,)

It is also not very good at deduplication by itself. You will end up with multiple memories that are serving the same context, or at least attempting to do so.

This leads to large complex projects reaching the point where toxic, stale, and inaccurate context is injected into Claude Code's system prompt.


---
*Captured at 12:27 — Monday, March 30th 2026*
