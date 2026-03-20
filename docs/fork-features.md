# Fork Features vs Upstream

This document captures features present in this fork (`aikdeirel/nanoclaw`) but not in upstream (`qwibitai/nanoclaw`). Useful for a quick "oh right, I added that" overview.

---

## 1. WhatsApp Channel

A full WhatsApp channel using `@whiskeysockets/baileys`. Handles QR-code auth, message ingestion/outbound, and image attachments via `sharp`.

**Relevant files:** `src/channels/whatsapp.ts`, `src/whatsapp-auth.ts`
**Type:** New channel — self-registers with the channel registry at startup.

---

## 2. Telegram Channel

Telegram integration using `grammy`. Supports text, photos, voice, video, documents, and stickers. `@mentions` are translated to the configured trigger pattern. Includes typing indicators.

**Relevant files:** `src/channels/telegram.ts`
**Type:** New channel — extends the channel registry system identically to WhatsApp.

---

## 3. Telegram Bot Pool (Agent Teams)

Multiple Telegram bot tokens can be registered as a pool. When an agent sub-agent sends a `send_message` IPC call with a `sender` field, the message is dispatched through a pool bot renamed to that sender's role. Bots are assigned round-robin per `{group}:{senderName}` for stable identity across a conversation.

**Relevant files:** `src/channels/telegram.ts` (pool logic), `src/config.ts` (`TELEGRAM_BOT_POOL`), `src/ipc.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts` (optional `sender` field on `send_message`)
**Type:** Extends the Telegram channel and the IPC `send_message` tool.

---

## 4. Per-Group LLM Model Config (`set_model`)

Each group can have an independently configured LLM model. The main group can set any group's model; non-main groups can only set their own. Stored in the `registered_groups` SQLite table. The selected model flows through `ContainerInput` to the agent-runner's `query()` call.

**Relevant files:** `src/db.ts`, `src/types.ts`, `src/ipc.ts`, `src/container-runner.ts`, `container/agent-runner/src/ipc-mcp-stdio.ts` (`set_model` MCP tool), `container/agent-runner/src/index.ts`
**Type:** New feature — extends IPC, DB schema, and agent-runner tooling.

---

## 5. Streaming Status Bubble

While the agent is running, a live "thinking…" message is sent to the chat and updated in real-time with streaming text/thinking deltas from the Claude SDK. It's deleted once the agent finishes. Implemented as a `StatusBubble` class wired into the orchestrator.

**Relevant files:** `src/index.ts` (`StatusBubble`), `src/channels/telegram.ts` (send/edit/delete status message methods), `src/types.ts` (`ContainerOutput` discriminated union), `container/agent-runner/src/index.ts` (emits `type:status` events)
**Type:** New feature — extends Telegram channel, container output protocol, and the main orchestrator.

---

## 6. Brave Search MCP Server

The agent-runner is configured with a Brave Search MCP server, giving all agents web search capability via the `BRAVE_API_KEY` env var.

**Relevant files:** `container/agent-runner/src/index.ts`
**Type:** Extends agent capabilities — drop-in addition to the MCP server list.

---

## 7. Agent Skills: handover, news-summary, software-research

Three new skills available to all agents inside containers:

- **`handover`** — Compresses the current conversation into a dense snapshot for resuming in a fresh session. Output is a single fenced code block.
- **`news-summary`** — Summarizes news articles in German (~30s read), with paywall detection, bias check, context, and open questions.
- **`software-research`** — Evaluates software/tools: fetches GitHub READMEs, pricing pages, docs; produces a structured analysis covering data flows, costs, maturity, and alternatives.

**Relevant files:** `container/skills/handover/SKILL.md`, `container/skills/news-summary/SKILL.md`, `container/skills/software-research/SKILL.md`
**Type:** New skills — additive, no changes to existing skill loading logic.

---

## 8. Image Vision for Telegram Photos

When a Telegram photo is received, the channel downloads the full-resolution image, processes it with the shared `processImage()` / `sharp` utility, and attaches it as multimodal content for the agent. Mirrors the same capability already present in the WhatsApp channel.

**Relevant files:** `src/channels/telegram.ts` (photo message handler), `src/image.ts`
**Type:** Extends the Telegram channel — uses the existing image processing utility.
