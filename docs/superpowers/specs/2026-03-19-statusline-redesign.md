# Statusline Redesign

**Date:** 2026-03-19
**Status:** Approved

## Goal

Redesign the Claude Code statusline bash script to be more beautiful and informative, inspired by [ccstatusline](https://github.com/sirmalloc/ccstatusline), while staying minimal and readable. No overengineering — a single bash script.

## Layout

```
aikdeirel ❯  nanoclaw ⎇  main +5 -2 │ Sonnet 4.6 ◐ 52% $0.04 │ 🦀 slay bestie!
```

Three visual clusters separated by `│`:

1. **Identity + navigation** — user `❯` dir `⎇` branch +ins -del
2. **Session info** — model thinking% cost
3. **Mascot** — 🦀 rotating cringe phrase

## Segments

### Identity block
- `aikdeirel` — bold bright green (`\033[01;92m`)
- `❯` — dim (`\033[2m`)
- ` nanoclaw` — Nerd Font folder glyph + dirname, bold bright cyan (`\033[01;96m`)

### Git block (inline, no extra separator from identity)
- `⎇ ` — branch symbol, bright magenta (`\033[95m`)
- `main` — branch name, bright magenta
- `+5` — insertions, bright green (`\033[92m`), hidden if 0
- `-2` — deletions, bright red (`\033[91m`), hidden if 0
- Entire block hidden if not in a git repo

### Separator
- ` │ ` — dim, between git block and info block

### Info block
- `Sonnet 4.6` — short model name (strip "Claude " prefix), orange (`\033[38;5;208m`)
- Thinking effort glyph immediately after model name (no separator):
  - `○` low
  - `◐` medium
  - `●` high
  - `◉` max
  - hidden if not available
- `52%` — context usage percentage, color by threshold:
  - `< 50%` → bright cyan (`\033[96m`)
  - `50–79%` → bright yellow (`\033[93m`)
  - `≥ 80%` → bright red (`\033[91m`)
- `$0.04` — session cost, dim orange (`\033[38;5;172m`), hidden if not in JSON

### Separator
- ` │ ` — dim

### Mascot block
- `🦀` + rotating cringe phrase — dim (`\033[2m`)
- 15 phrases, cycle every ~4 seconds via `$(date +%S) % 15`

## Implementation Notes

- Single bash script at `~/.claude/statusline-command.sh`
- Reads JSON from stdin via `jq`
- Git diff stats: `git diff --shortstat HEAD` parsed for insertions/deletions
- Model short name: strip `^Claude ` prefix with bash parameter expansion
- Thinking effort: read from `.thinking_effort` or `.model.thinking_effort` in JSON
- Session cost: read from `.session_cost` or `.cost.session_total` in JSON
- All segments gracefully hidden when data unavailable
- Use `printf "%b"` for ANSI escape interpretation
- Nerd Font glyph `` for folder (U+F07C), `⎇` for branch (plain Unicode, widely supported)

## Colors Reference

| Element | ANSI |
|---|---|
| Username | `\033[01;92m` bold bright green |
| Dirname | `\033[01;96m` bold bright cyan |
| Branch + ⎇ | `\033[95m` bright magenta |
| Insertions | `\033[92m` bright green |
| Deletions | `\033[91m` bright red |
| Model + effort | `\033[38;5;208m` orange |
| ctx < 50% | `\033[96m` bright cyan |
| ctx 50–79% | `\033[93m` bright yellow |
| ctx ≥ 80% | `\033[91m` bright red |
| Cost | `\033[38;5;172m` dim orange |
| Separators | `\033[2m` dim |
| Mascot phrase | `\033[2m` dim |
