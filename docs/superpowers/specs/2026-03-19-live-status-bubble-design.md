# Live Status Bubble — Design Spec

**Date:** 2026-03-19
**Status:** Approved

## Overview

Show a live-updating Telegram message ("status bubble") that reflects what the agent is doing in real-time — tool calls, thinking text, and parallel sub-agent activity. The bubble is edited in-place as status changes, and deleted when the agent finishes.

## Problem

The current `setTyping` / `sendChatAction('typing')` indicator disappears as soon as the bot executes tools or coordinates agent swarms. Users have no visibility into what is happening during long-running operations.

## Goals

- Show real-time per-agent status during execution
- Support parallel agents (swarms) — all active agents shown simultaneously
- No hardcoded mappings — status text comes directly from SDK events
- Minimal architectural footprint — extend existing stdout marker protocol

## Non-Goals

- Persistent history of steps (bubble is deleted on completion)
- Debouncing/rate-limiting (edit on every event, ignore 429s)
- Support for channels other than Telegram (initially)

---

## Architecture

### 1. Protocol Extension (`container/agent-runner/src/index.ts`)

Extend the existing `---NANOCLAW_OUTPUT_START---` / `---NANOCLAW_OUTPUT_END---` marker protocol with a new event type:

```typescript
// Existing types (extended with explicit type discriminant)
{ type: 'result'; status: 'success'; result: string | null; newSessionId?: string }
{ type: 'result'; status: 'error'; error: string }

// New type
{ type: 'status'; agentName: string; text: string }
```

All three variants now use `type` as the top-level discriminant. The existing `status` field on result variants is preserved but no longer the primary discriminant.

**Source of status text — no hardcoded mappings:**

| SDK Event | `agentName` | `text` |
|---|---|---|
| `assistant` → `tool_use` block (any tool) | `currentAgentName` (see below) | `"{tool_name}: {primaryStringFromInput}"` |
| `assistant` → `text` block | `currentAgentName` | First 80 chars + `"..."` (if longer) |

**Agent name tracking for swarms:**

When the SDK emits a `tool_use` block with `name === 'Task'` (or `'TeamCreate'`), the `input` object contains the sub-agent task definition. Parse the agent name from `input.description` first, then `input.task` (whichever is a string — first ~30 chars). Store two mappings:
- `toolUseId → agentName` (for attributing subsequent assistant messages via `parent_tool_use_id`)
- `taskId → agentName` (for removal on `task_notification`, which carries `task_id`, not `tool_use` id)

The `task_id` in `task_notification` is distinct from the `tool_use` block `id`. When the Task tool_use fires, also store the expected task_id if available in the response, or fall back to `tool_use_id` matching if the SDK doesn't provide it separately (to be verified during implementation).

When subsequent `assistant` messages arrive with a matching `parent_tool_use_id`, use the `toolUseId → agentName` map.

For the main agent: `agentName = "main"`.

For sub-agents without a parseable name: `agentName = "agent-{toolUseId.slice(0,6)}"`.

`task_notification` events (which fire on sub-agent *completion*) look up `agentName` via the `taskId → agentName` map and call `onAgentDone(agentName)` to remove from the active bubble.

**Input serialization for `tool_use`:**

Extract the first string value from the `input` object (e.g. `query`, `command`, `description`, `prompt`, `task` — whatever key has a string value first). Slice to 80 chars. If no string values found, use `JSON.stringify(input).slice(0, 80)`.

### 2. Type Extension (`src/container-runner.ts` + `container/agent-runner/src/index.ts`)

Both files define `ContainerOutput`. Both must be updated:

```typescript
type ContainerOutput =
  | { type: 'result'; status: 'success'; result: string | null; newSessionId?: string }
  | { type: 'result'; status: 'error'; error: string }
  | { type: 'status'; agentName: string; text: string }
```

**All consumers must guard on `type` first:**

- `src/index.ts` — `onOutput` callback in `processGroupMessages`
- `src/index.ts` — `wrappedOnOutput` in `runAgent` (reads `newSessionId`)
- `src/task-scheduler.ts` — `onOutput` in scheduled task runner

Guard pattern:
```typescript
if (result.type === 'status') { /* handle status */ return; }
// result is now narrowed to type: 'result'
```

### 3. Status Bubble Lifecycle (`src/index.ts`)

A `StatusBubble` object is created per `processGroupMessages` call — isolated per group, no shared state:

```typescript
class StatusBubble {
  private statuses = new Map<string, string>(); // agentName → current status text
  private messageId: number | null = null;

  constructor(
    private chatJid: string,
    private telegramChannel: TelegramChannel  // typed directly, not Channel
  ) {}

  async onStatus(agentName: string, text: string): Promise<void>
  // Updates statuses map, sends or edits bubble.
  // If editStatusMessage returns 'not_found', clears this.messageId (no further edits until next onStatus).

  async onAgentDone(agentName: string): Promise<void>
  // Removes agentName from statuses map.
  // If statuses is now empty → calls delete() instead of editing.
  // If statuses still has entries → edits bubble with remaining entries.

  async delete(): Promise<void>
  // Calls deleteStatusMessage and clears messageId.

  private renderText(): string {
    return [...this.statuses.entries()]
      .map(([name, text]) => `⟳ ${name}: ${text}`)
      .join('\n');
  }
}
```

`StatusBubble` is only instantiated when the active channel is `TelegramChannel`. For other channels, status events are silently ignored.

**Lifecycle:**

1. First `type: 'status'` event → `sendStatusMessage` → store `messageId`
2. Subsequent `type: 'status'` events → update map → `editStatusMessage`
3. `task_notification` (sub-agent done) → `onAgentDone(agentName)` → remove from map → edit bubble (or delete if empty)
4. Final `result` with `result.result !== null` → `delete()` bubble → send final answer

**Note:** `result` events with `result.result === null` (session-update markers between turns) do NOT trigger bubble deletion.

### 4. Telegram Channel (`src/channels/telegram.ts`)

Add three methods to `TelegramChannel`:

```typescript
async sendStatusMessage(chatJid: string, text: string): Promise<number | null>
// Returns message_id, or null on failure (silently — no bubble is fine)

async editStatusMessage(chatJid: string, messageId: number, text: string): Promise<'ok' | 'not_found'>
// Returns 'not_found' when Telegram says "message to edit not found"
// Silently ignores: 429 (returns 'ok'), "message is not modified" (returns 'ok')
// StatusBubble.onStatus must null out this.messageId when 'not_found' is returned

async deleteStatusMessage(chatJid: string, messageId: number): Promise<void>
// Silently ignores all errors
```

These are distinct from `sendMessage` — different error-handling semantics. Not added to the `Channel` interface since this is Telegram-only initially.

---

## Data Flow

```
SDK Event (tool_use / text / task_notification)
  ↓
agent-runner: extract agentName + text
  ↓
writeOutput({ type: 'status', agentName, text })
  ↓  (stdout marker)
container-runner: parse → call onOutput({ type: 'status', ... })
  ↓
index.ts onOutput: StatusBubble.onStatus(agentName, text)
  ↓
TelegramChannel.editStatusMessage (or sendStatusMessage on first call)
  ↓
Telegram API: editMessageText
  ↓
User sees updated bubble
```

---

## Files Changed

| File | Change |
|---|---|
| `container/agent-runner/src/index.ts` | Intercept SDK assistant events, emit `type:'status'` markers; track agentName via Task tool_use mapping |
| `src/container-runner.ts` | Extend `ContainerOutput` union type with `type` discriminant |
| `src/index.ts` | `StatusBubble` class, handle `type:'status'` in `onOutput`; guard `wrappedOnOutput` |
| `src/task-scheduler.ts` | Guard `onOutput` against `type:'status'` events |
| `src/channels/telegram.ts` | Add `sendStatusMessage`, `editStatusMessage`, `deleteStatusMessage` |

## Edge Cases

| Scenario | Handling |
|---|---|
| 429 on edit | Silently ignore — next status event overwrites |
| "message to edit not found" (user deleted bubble) | Clear `messageId`, no further edits |
| Agent completes before any status event | No bubble created, nothing to delete |
| `result.result === null` (session-update marker) | Do NOT delete bubble — wait for non-null result |
| Container error (`status: 'error'`) | Delete bubble, send error message as normal |
| Channel is not Telegram | `StatusBubble` not instantiated, status events dropped silently |
| Sub-agent name unparseable from Task input | Fall back to `"agent-{toolUseId.slice(0,6)}"` |
