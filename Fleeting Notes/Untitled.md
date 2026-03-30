---
tags:
  - fleeting
  - general
created: 2026-03-30
status: inbox
---

# Auto-Memory

Auto-Memory was recently introduced to CC where the agent will save lessons learned to a file. Starting with a `MEMORY.md` file inside the `~/.claude/projects/-path-to-project-/memory/MEMORY.md`, and then serving as an index file with pointers to individual memory files to keep it under 200 lines.

It uses the syntax of `- [filename](filename.md) - Description of memory`, and inside the file itself is the rule with frontmatter.


- Auto-memory auto accumulates over time
- Claude does not appear to tidy up memory as it adds and changes memories
	- It checks to see if the memory exists
		- If it does, it appends to or updates that memory
		- If it doesn't, it creates that memory



---
*Captured at 12:27 — Monday, March 30th 2026*
