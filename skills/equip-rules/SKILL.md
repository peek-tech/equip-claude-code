---
name: equip-rules
description: Install baseline project rules to CLAUDE.md
allowed-tools: Read, Write, Edit
---

# Install Project Rules

Append baseline rules to `CLAUDE.md` in the project root, filtered by the detected tech stack.

## Rules Table

| Rule | Match When |
|------|------------|
| Follow existing file structure and naming conventions. | Always |
| Use Typescript strict mode. | `typescript` in languages |
| Use functional components and hooks. | `react`, `preact`, or `next` in deps |
| Write unit tests for new features. | Always |
| Always output full code blocks. | Always |

## Installation Procedure

1. **Read** `CLAUDE.md` (or start empty)
2. **Collect** matching rules based on detected tech stack
3. **Dedup** — skip any rule whose text already appears in `CLAUDE.md`
4. **Append** new rules under a `# Project Rules` section (create the section if it doesn't exist)

## Policies

- **Append-only** — rules are never removed on re-run, since the user may have customized them
- Only add rules that match the detected stack
- Preserve all existing content in `CLAUDE.md`
