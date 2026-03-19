---
name: handover
description: Compress the current session into a dense, lossless context snapshot for seamless continuation in a fresh session. Trigger on /handover or /handover save, or when the user mentions "handover", wants to compress/save context, or end a session cleanly.
---

# Handover — Context Compression Skill

## Purpose

Conversations accumulate noise. This skill extracts the signal and outputs it as a single copyable block.

Think hard. You are compressing a session. The output is *fuel for a future AI instance* — optimize for that reader, not for human readability.

**Guiding principles:**
- Ruthlessly cut anything a fresh instance could reconstruct or doesn't need
- Keep: decisions made, their reasoning, open questions, blockers, exact state of work, what comes next
- Compress language — telegraphic style over full sentences
- Domain-agnostic: let the conversation content dictate the structure, not a template
- If there is code, data, or a query that matters — include the exact text, not a description of it
- If there are file paths, URLs, IDs, or names that matter — include the exact values

## Output format — critical

The entire response MUST be a single fenced code block and nothing else. No greeting, no explanation before or after, no "here is your handover", no follow-up questions. Just:

```
[compressed session content]
```

Use a plain triple-backtick fence. The user copies the block and pastes it into a new session to resume.
