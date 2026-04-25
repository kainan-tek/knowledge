---
name: knowledge
description: Query and manage the project PDF knowledge base with indexed access. Triggers on /knowledge commands or when users ask about project documents.
---

# Knowledge Skill — Claude Code Adapter

Execute the following steps before doing anything else:

1. Read `SKILL.md` — this is the universal skill definition
2. Read `CLAUDE.md` — this is the Claude Code platform adapter
3. Follow the instructions in SKILL.md, using the tool mapping and path resolution defined in CLAUDE.md

## Input Parsing

If the user typed `/knowledge X`:
- If X is `init`: execute the init subcommand (generate config)
- If X is `status`: execute the status subcommand
- If X is `index`: execute the index subcommand (full scan)
- If X starts with `index ` and has a filename: extract filename, execute index for that single file
- If X starts with `reindex ` and has a filename: extract filename, execute reindex
- Otherwise: treat X as a natural language question, execute the query flow
