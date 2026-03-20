# Live Status Bubble Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Show a live-updating Telegram message ("status bubble") that reflects what the container agent is doing in real-time — tool calls, thinking text, and parallel sub-agent activity.

**Architecture:** Extend the existing stdout marker protocol with a new `type:'status'` event emitted by the agent-runner on each SDK `assistant` event (tool_use / text blocks). The host (`src/index.ts`) creates a `StatusBubble` per agent run and forwards status events to three new `TelegramChannel` methods that send/edit/delete a single Telegram message in-place.

**Tech Stack:** Node.js / TypeScript, grammY (Telegram SDK already used), Claude Agent SDK (already used), SQLite (unchanged), existing stdout marker protocol.

---

## File Map

| File | Change |
|---|---|
| `container/agent-runner/src/index.ts` | Extend `ContainerOutput` type; intercept SDK `assistant` events; emit `type:'status'` markers; track agentName via Task/TeamCreate tool_use mapping |
| `src/container-runner.ts` | Extend exported `ContainerOutput` union type with `type` discriminant |
| `src/channels/telegram.ts` | Add `sendStatusMessage`, `editStatusMessage`, `deleteStatusMessage` methods |
| `src/index.ts` | Add `StatusBubble` class; handle `type:'status'` in `onOutput` callback; guard `wrappedOnOutput` against status events |
| `src/task-scheduler.ts` | Guard `onOutput` callback against `type:'status'` events |

---

## Task 1: Extend `ContainerOutput` type in `src/container-runner.ts`

The exported `ContainerOutput` interface must become a discriminated union so callers can guard on `type` before accessing fields.

**Files:**
- Modify: `src/container-runner.ts:47-52`

- [ ] **Step 1: Replace the `ContainerOutput` interface with a union type**

Find this block (lines 47–52):
```typescript
export interface ContainerOutput {
  status: 'success' | 'error';
  result: string | null;
  newSessionId?: string;
  error?: string;
}
```

Replace with:
```typescript
export type ContainerOutput =
  | { type: 'result'; status: 'success'; result: string | null; newSessionId?: string }
  | { type: 'result'; status: 'error'; error: string; newSessionId?: string }
  | { type: 'status'; agentName: string; text: string };
```

- [ ] **Step 2: Fix the two internal parse sites in `src/container-runner.ts` that construct a `ContainerOutput` object**

In the streaming parse path (around line 362):
```typescript
const parsed: ContainerOutput = JSON.parse(jsonStr);
```
No change needed here — JSON.parse produces the right shape.

In the legacy fallback path (around line 599):
```typescript
const output: ContainerOutput = JSON.parse(jsonLine);
```
No change needed here either — same reasoning.

- [ ] **Step 3: Fix ALL `resolve({...})` call sites in `src/container-runner.ts` that construct literal objects**

Each object literal must add `type: 'result'`. Find all sites first:
```bash
grep -n 'resolve({' src/container-runner.ts
```

There are 6 resolve() sites — update all of them:

```typescript
// Timeout with output (around line 465):
resolve({ type: 'result', status: 'success', result: null, newSessionId });

// Timeout without output (around line 479):
resolve({ type: 'result', status: 'error', result: null, error: `Container timed out after ${configTimeout}ms` });

// Non-zero exit (around line 558):
resolve({ type: 'result', status: 'error', result: null, error: `Container exited with code ${code}: ${stderr.slice(-200)}` });

// Streaming mode completion (around line 573):
resolve({ type: 'result', status: 'success', result: null, newSessionId });

// Legacy mode parse error (around line 623):
resolve({ type: 'result', status: 'error', result: null, error: `Failed to parse container output: ${err instanceof Error ? err.message : String(err)}` });

// Spawn error (around line 637):
resolve({ type: 'result', status: 'error', result: null, error: `Container spawn error: ${err.message}` });
```

**Note:** Legacy mode parsed output (around line 599, `const output: ContainerOutput = JSON.parse(jsonLine)`) is already typed from JSON — no change needed there.

**Note:** `ContainerOutput` in the type has no `result` field on the error variant per spec — but keep `result: null` at call sites that already have it, just don't require it in the type. To avoid widespread breakage, keep `result?: string | null` on the error variant:

```typescript
export type ContainerOutput =
  | { type: 'result'; status: 'success'; result: string | null; newSessionId?: string }
  | { type: 'result'; status: 'error'; error: string; result?: string | null; newSessionId?: string }
  | { type: 'status'; agentName: string; text: string };
```

- [ ] **Step 4: Build TypeScript to verify no compile errors**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

Expected: build succeeds (or only shows pre-existing errors unrelated to this change).

- [ ] **Step 5: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add src/container-runner.ts
git commit -m "feat(status-bubble): extend ContainerOutput to discriminated union with type field"
```

---

## Task 2: Guard `onOutput` consumers against `type:'status'` events

Before implementing StatusBubble, protect all existing `onOutput` callbacks so they don't crash when they receive the new event type.

**Files:**
- Modify: `src/index.ts:259-293` (processGroupMessages onOutput)
- Modify: `src/index.ts:358-366` (wrappedOnOutput in runAgent)
- Modify: `src/task-scheduler.ts:185-199` (streamedOutput callback)

- [ ] **Step 1: Guard `wrappedOnOutput` in `src/index.ts`**

Find the `wrappedOnOutput` in `runAgent` (around line 358):
```typescript
const wrappedOnOutput = onOutput
  ? async (output: ContainerOutput) => {
      if (output.newSessionId) {
        sessions[group.folder] = output.newSessionId;
        setSession(group.folder, output.newSessionId);
      }
      await onOutput(output);
    }
  : undefined;
```

Replace with:
```typescript
const wrappedOnOutput = onOutput
  ? async (output: ContainerOutput) => {
      if (output.type === 'status') {
        await onOutput(output);
        return;
      }
      if (output.newSessionId) {
        sessions[group.folder] = output.newSessionId;
        setSession(group.folder, output.newSessionId);
      }
      await onOutput(output);
    }
  : undefined;
```

- [ ] **Step 2: Guard `onOutput` in `processGroupMessages` in `src/index.ts`**

Find the `onOutput` callback passed to `runAgent` (around line 264):
```typescript
async (result) => {
  // Streaming output callback — called for each agent result
  if (result.result) {
```

Replace with:
```typescript
async (result) => {
  // Status events are handled by StatusBubble (added in next task)
  if (result.type === 'status') return;
  // Streaming output callback — called for each agent result
  if (result.result) {
```

- [ ] **Step 3: Guard `onOutput` in `src/task-scheduler.ts`**

Find the streaming callback in `runTask` (around line 185):
```typescript
async (streamedOutput: ContainerOutput) => {
  if (streamedOutput.result) {
```

Replace with:
```typescript
async (streamedOutput: ContainerOutput) => {
  if (streamedOutput.type === 'status') return;
  if (streamedOutput.result) {
```

- [ ] **Step 4: Build to verify**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

Expected: no new errors.

- [ ] **Step 5: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add src/index.ts src/task-scheduler.ts
git commit -m "feat(status-bubble): guard onOutput consumers against type:status events"
```

---

## Task 3: Add `sendStatusMessage`, `editStatusMessage`, `deleteStatusMessage` to `TelegramChannel`

**Files:**
- Modify: `src/channels/telegram.ts`

- [ ] **Step 1: Add the three methods to `TelegramChannel` class**

After the `setTyping` method (around line 430), add:

```typescript
async sendStatusMessage(chatJid: string, text: string): Promise<number | null> {
  if (!this.bot) return null;
  try {
    const numericId = chatJid.replace(/^tg:/, '');
    const msg = await this.bot.api.sendMessage(numericId, text);
    return msg.message_id;
  } catch (err) {
    logger.debug({ chatJid, err }, 'Failed to send status message');
    return null;
  }
}

async editStatusMessage(chatJid: string, messageId: number, text: string): Promise<'ok' | 'not_found'> {
  if (!this.bot) return 'ok';
  try {
    const numericId = chatJid.replace(/^tg:/, '');
    await this.bot.api.editMessageText(numericId, messageId, text);
    return 'ok';
  } catch (err: any) {
    const description: string = err?.description || err?.message || '';
    if (description.includes('message to edit not found')) return 'not_found';
    // 429 rate limit, "message is not modified", and other transient errors — ignore
    return 'ok';
  }
}

async deleteStatusMessage(chatJid: string, messageId: number): Promise<void> {
  if (!this.bot) return;
  try {
    const numericId = chatJid.replace(/^tg:/, '');
    await this.bot.api.deleteMessage(numericId, messageId);
  } catch {
    // Silently ignore all errors (message already deleted, etc.)
  }
}
```

- [ ] **Step 2: Build to verify**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add src/channels/telegram.ts
git commit -m "feat(status-bubble): add sendStatusMessage, editStatusMessage, deleteStatusMessage to TelegramChannel"
```

---

## Task 4: Implement `StatusBubble` class in `src/index.ts`

**Files:**
- Modify: `src/index.ts`

- [ ] **Step 1: Add the `StatusBubble` class**

After the imports (after line 69) and before the module-level state variables, add:

```typescript
class StatusBubble {
  private statuses = new Map<string, string>(); // agentName → current status text
  private messageId: number | null = null;

  constructor(
    private chatJid: string,
    private telegramChannel: TelegramChannel,
  ) {}

  async onStatus(agentName: string, text: string): Promise<void> {
    this.statuses.set(agentName, text);
    const rendered = this.renderText();
    if (this.messageId === null) {
      this.messageId = await this.telegramChannel.sendStatusMessage(this.chatJid, rendered);
    } else {
      const result = await this.telegramChannel.editStatusMessage(this.chatJid, this.messageId, rendered);
      if (result === 'not_found') {
        this.messageId = null;
      }
    }
  }

  async onAgentDone(agentName: string): Promise<void> {
    this.statuses.delete(agentName);
    if (this.statuses.size === 0) {
      await this.delete();
    } else if (this.messageId !== null) {
      await this.telegramChannel.editStatusMessage(this.chatJid, this.messageId, this.renderText());
    }
  }

  async delete(): Promise<void> {
    if (this.messageId !== null) {
      await this.telegramChannel.deleteStatusMessage(this.chatJid, this.messageId);
      this.messageId = null;
    }
  }

  private renderText(): string {
    return [...this.statuses.entries()]
      .map(([name, text]) => `⟳ ${name}: ${text}`)
      .join('\n');
  }
}
```

- [ ] **Step 2: Wire `StatusBubble` into `processGroupMessages`**

In `processGroupMessages`, after the channel lookup (after line 162), add bubble instantiation when channel is Telegram.

First update the import at line 13 of `src/index.ts`. The current import is:
```typescript
import { initBotPool } from './channels/telegram.js';
```
Change it to:
```typescript
import { TelegramChannel, initBotPool } from './channels/telegram.js';
```

Then inside `processGroupMessages`, after `const channel = findChannel(channels, chatJid);`:

```typescript
const bubble = channel instanceof TelegramChannel
  ? new StatusBubble(chatJid, channel)
  : null;
```

- [ ] **Step 3: Forward `type:'status'` events to the bubble in the `onOutput` callback**

Replace the guard added in Task 2 Step 2:
```typescript
if (result.type === 'status') return;
```

With:
```typescript
if (result.type === 'status') {
  await bubble?.onStatus(result.agentName, result.text);
  return;
}
```

- [ ] **Step 4: Delete bubble on final result**

In the same `onOutput` callback, after the `if (result.result)` block, add before `if (result.status === 'success')`:

```typescript
// Delete status bubble when a real result arrives
if (result.result !== null) {
  await bubble?.delete();
}
```

Full updated `onOutput` callback should look like:
```typescript
async (result) => {
  if (result.type === 'status') {
    await bubble?.onStatus(result.agentName, result.text);
    return;
  }
  if (result.result !== null) {
    await bubble?.delete();
  }
  if (result.result) {
    const raw = typeof result.result === 'string'
      ? result.result
      : JSON.stringify(result.result);
    const text = raw.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
    logger.info({ group: group.name }, `Agent output: ${raw.slice(0, 200)}`);
    if (text) {
      await channel.sendMessage(chatJid, text);
      outputSentToUser = true;
    }
    resetIdleTimer();
  }

  if (result.status === 'success') {
    queue.notifyIdle(chatJid);
  }

  if (result.status === 'error') {
    hadError = true;
  }
},
```

- [ ] **Step 5: Also delete bubble on error**

After the `runAgent` call returns (around line 299), add bubble cleanup for the error path:

```typescript
if (output === 'error' || hadError) {
  await bubble?.delete();  // Clean up bubble on agent error
  // ... existing cursor rollback logic
```

- [ ] **Step 6: Build to verify**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

Expected: no errors.

- [ ] **Step 7: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add src/index.ts
git commit -m "feat(status-bubble): add StatusBubble class and wire into processGroupMessages"
```

---

## Task 5: Extend `ContainerOutput` and emit `type:'status'` in `container/agent-runner/src/index.ts`

This is the largest task — intercept SDK events and write status markers to stdout.

**Important — container agent-runner isolation:** Each group gets its own copy of agent-runner source in `data/sessions/{group}/agent-runner-src/`. Changes to `container/agent-runner/src/index.ts` are the "template" only — they don't affect existing groups until the per-group copies are updated. For development/testing, use a new group, or manually copy the updated file over an existing group's copy, or run `./container/build.sh` (which rebuilds the container image but doesn't update per-group copies).

**Files:**
- Modify: `container/agent-runner/src/index.ts`

- [ ] **Step 1: Update `ContainerOutput` interface to match the host type**

Find (lines 44–49):
```typescript
interface ContainerOutput {
  status: 'success' | 'error';
  result: string | null;
  newSessionId?: string;
  error?: string;
}
```

Replace with:
```typescript
type ContainerOutput =
  | { type: 'result'; status: 'success'; result: string | null; newSessionId?: string }
  | { type: 'result'; status: 'error'; error: string; result?: string | null; newSessionId?: string }
  | { type: 'status'; agentName: string; text: string };
```

- [ ] **Step 2: Update all `writeOutput(...)` call sites to add `type: 'result'`**

There are several `writeOutput({status: ..., result: ..., ...})` calls. Add `type: 'result'` to each:

In `main()` error handling for bad stdin (around line 523):
```typescript
writeOutput({ type: 'result', status: 'error', result: null, error: `Failed to parse input: ...` });
```

In the slash command handler, all `writeOutput` calls (around lines 608–649). Each needs `type: 'result'` added.

In `runQuery`, the `writeOutput` call (around line 501):
```typescript
writeOutput({ type: 'result', status: 'success', result: textResult || null, newSessionId });
```

In the main query loop, the session-update marker (around line 678):
```typescript
writeOutput({ type: 'result', status: 'success', result: null, newSessionId: sessionId });
```

In the error handler (around line 695):
```typescript
writeOutput({ type: 'result', status: 'error', result: null, newSessionId: sessionId, error: errorMessage });
```

- [ ] **Step 3: Add agent name tracking state before `runQuery`**

Add these module-level variables at the top of the file (after the constants, before `MessageStream`):

```typescript
// Agent name tracking for status bubble
// Maps tool_use_id → agentName (for Task/TeamCreate subagent attribution)
const toolUseIdToAgentName = new Map<string, string>();
// Maps task_id → agentName (for task_notification completion events)
const taskIdToAgentName = new Map<string, string>();

/**
 * Extract a display name from a Task/TeamCreate tool input.
 * Tries input.description, then input.task (first ~30 chars).
 * Falls back to agentId if neither is a string.
 */
function extractAgentName(input: Record<string, unknown>, fallbackId: string): string {
  for (const key of ['description', 'task']) {
    const val = input[key];
    if (typeof val === 'string' && val.trim()) {
      return val.trim().slice(0, 30);
    }
  }
  return `agent-${fallbackId.slice(0, 6)}`;
}

/**
 * Extract the primary string from a tool_use input for display.
 * Returns first string value found, sliced to 80 chars.
 */
function extractPrimaryString(input: Record<string, unknown>): string {
  for (const val of Object.values(input)) {
    if (typeof val === 'string' && val.trim()) {
      return val.trim().slice(0, 80);
    }
  }
  return JSON.stringify(input).slice(0, 80);
}
```

- [ ] **Step 4: Add status event emission inside the `for await` loop in `runQuery`**

The main SDK message loop is in `runQuery` (around line 431). After the existing `task_notification` handler and before the `result` handler, add:

```typescript
// Emit status events for assistant messages (tool use and text)
if (message.type === 'assistant') {
  const assistantMsg = message as { message?: { content?: unknown[] }; parent_tool_use_id?: string | null };
  const content = assistantMsg.message?.content;
  if (Array.isArray(content)) {
    // Determine agentName from parent_tool_use_id (for sub-agents) or default to 'main'
    const parentId = assistantMsg.parent_tool_use_id;
    const currentAgentName = (parentId && toolUseIdToAgentName.get(parentId)) || 'main';

    for (const block of content) {
      const b = block as { type: string; name?: string; id?: string; input?: Record<string, unknown>; text?: string };
      if (b.type === 'tool_use') {
        // Track sub-agent name for Task and TeamCreate tool calls
        if ((b.name === 'Task' || b.name === 'TeamCreate') && b.id && b.input) {
          const agentName = extractAgentName(b.input, b.id);
          toolUseIdToAgentName.set(b.id, agentName);
          // Best-effort: also register task_id mapping if available in input.
          // If task_id is not in input, the fallback is to store using the tool_use_id
          // as the task_id key — this won't match task_notification (which uses a different
          // task_id), but prevents losing the mapping entirely. task_notification events
          // will still attempt the lookup; if it fails, no agentDone signal is emitted
          // (the sub-agent simply stays in the bubble until the final delete).
          const taskId = b.input.task_id;
          if (typeof taskId === 'string') {
            taskIdToAgentName.set(taskId, agentName);
          }
          // Also store by tool_use_id as a fallback key
          taskIdToAgentName.set(b.id, agentName);
        }
        // Emit status for all tool calls
        const primaryStr = b.input ? extractPrimaryString(b.input) : '';
        const statusText = primaryStr ? `${b.name}: ${primaryStr}` : (b.name || 'tool');
        writeOutput({ type: 'status', agentName: currentAgentName, text: statusText });
      } else if (b.type === 'text' && b.text) {
        const truncated = b.text.length > 80 ? b.text.slice(0, 80) + '...' : b.text;
        writeOutput({ type: 'status', agentName: currentAgentName, text: truncated });
      }
    }
  }
}
```

- [ ] **Step 5: Handle `task_notification` for sub-agent completion**

Find the existing `task_notification` handler (around line 492):
```typescript
if (message.type === 'system' && (message as { subtype?: string }).subtype === 'task_notification') {
  const tn = message as { task_id: string; status: string; summary: string };
  log(`Task notification: task=${tn.task_id} status=${tn.status} summary=${tn.summary}`);
}
```

Extend it to also emit an `agentDone` status event. Since the `StatusBubble` doesn't know about `agentDone` events directly from the container, we'll emit a `status` event with an empty text and a special sentinel that the host can use — wait, the spec says `onAgentDone` is called by `StatusBubble` when it receives `task_notification`. But `task_notification` is an SDK event, not a marker we emit.

**Re-read spec section "Agent name tracking for swarms":** `task_notification` events call `onAgentDone(agentName)` via `StatusBubble`. This means we need a new marker type OR we emit a status with empty text to signal completion.

The spec says: `{ type: 'status'; agentName: string; text: string }` — use `text: ''` to signal done:

Actually, re-reading the spec: the host calls `StatusBubble.onAgentDone(agentName)` — but `task_notification` arrives inside the container. The simplest approach is to emit a `status` event with `text: ''` when task completes, and have the host treat `text: ''` as `onAgentDone`. This stays within the spec's single event type.

Update the `task_notification` handler:
```typescript
if (message.type === 'system' && (message as { subtype?: string }).subtype === 'task_notification') {
  const tn = message as { task_id: string; status: string; summary: string };
  log(`Task notification: task=${tn.task_id} status=${tn.status} summary=${tn.summary}`);
  // Signal sub-agent completion to the host's StatusBubble
  const agentName = taskIdToAgentName.get(tn.task_id);
  if (agentName) {
    writeOutput({ type: 'status', agentName, text: '' });
    taskIdToAgentName.delete(tn.task_id);
  }
}
```

- [ ] **Step 6: Build the agent-runner TypeScript**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw/container/agent-runner && npx tsc --noEmit 2>&1 | head -60
```

If tsc is not local: check `package.json` for the build command.

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw/container/agent-runner && npm run build 2>&1 | head -60
```

Expected: no errors.

- [ ] **Step 7: Rebuild the host as well**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

- [ ] **Step 8: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add container/agent-runner/src/index.ts
git commit -m "feat(status-bubble): emit type:status markers from agent-runner on SDK assistant events"
```

---

## Task 6: Handle `text:''` (agentDone) in `StatusBubble`

Update `StatusBubble.onStatus` to route empty-text events to `onAgentDone`.

**Files:**
- Modify: `src/index.ts` (StatusBubble class, onStatus method)

- [ ] **Step 1: Update `onStatus` to handle `text: ''` as agent-done signal**

Find `onStatus` in `StatusBubble`:
```typescript
async onStatus(agentName: string, text: string): Promise<void> {
  this.statuses.set(agentName, text);
  ...
}
```

Replace with:
```typescript
async onStatus(agentName: string, text: string): Promise<void> {
  if (text === '') {
    await this.onAgentDone(agentName);
    return;
  }
  this.statuses.set(agentName, text);
  const rendered = this.renderText();
  if (this.messageId === null) {
    this.messageId = await this.telegramChannel.sendStatusMessage(this.chatJid, rendered);
  } else {
    const result = await this.telegramChannel.editStatusMessage(this.chatJid, this.messageId, rendered);
    if (result === 'not_found') {
      this.messageId = null;
    }
  }
}
```

- [ ] **Step 2: Build to verify**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run build 2>&1 | head -60
```

- [ ] **Step 3: Commit**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add src/index.ts
git commit -m "feat(status-bubble): route empty-text status events to onAgentDone in StatusBubble"
```

---

## Task 7: Fix tests broken by ContainerOutput type change

The discriminated union change breaks test fixtures that construct `ContainerOutput` objects without `type`.

**Files:**
- Modify: any `*.test.ts` files in the project

- [ ] **Step 1: Find all test files that reference ContainerOutput**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && grep -rn 'status.*success\|status.*error\|emitOutputMarker\|ContainerOutput' --include='*.test.ts' .
```

- [ ] **Step 2: Update all test fixture objects and emitOutputMarker helpers**

Any place a test creates a literal `ContainerOutput`-shaped object must add `type: 'result'`. For example in `src/container-runner.test.ts`, the `emitOutputMarker()` helper function likely emits `{ status: 'success', result: ... }` — update to `{ type: 'result', status: 'success', result: ... }`.

Pattern to find and fix:
```typescript
// Before:
{ status: 'success', result: 'hello', newSessionId: 'abc' }
// After:
{ type: 'result', status: 'success', result: 'hello', newSessionId: 'abc' }
```

- [ ] **Step 3: Run existing test suite**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm test 2>&1 | tail -40
```

Expected: all previously-passing tests still pass.

- [ ] **Step 4: Fix any remaining TypeScript narrowing failures in tests**

If tests access `result.status` or `result.result` directly without a type guard, add:
```typescript
if (result.type !== 'result') throw new Error('Expected result type');
```

- [ ] **Step 5: Commit test fixes**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw
git add -p  # stage only test files
git commit -m "fix(status-bubble): update tests to use discriminated ContainerOutput type"
```

---

## Task 8: Manual verification checklist

- [ ] **Step 1: Start the service in dev mode**

```bash
cd /home/aikdeirel/code-workspaces/nanoclaw && npm run dev
```

- [ ] **Step 2: Send a message to the Telegram bot that triggers the agent**

Expected behavior:
- A status bubble appears shortly after the agent starts (first tool call)
- Bubble updates as more tool calls occur
- Bubble is deleted when the final response is sent
- The final response appears as a normal message after the bubble is gone

- [ ] **Step 3: Verify edge cases**

- Send a quick message that the agent answers without tool calls → no bubble should appear
- If bubble is manually deleted mid-run → subsequent edits should be silently ignored (no errors in logs)

---

## Notes

- **`result: null` session-update markers** — these have `type: 'result'` and `result: null`. The bubble is only deleted when `result !== null` (real content). The guard `if (result.result !== null) { await bubble?.delete(); }` handles this correctly.
- **Non-Telegram channels** — `bubble` is `null` when channel is not a `TelegramChannel`, so all `bubble?.` calls are no-ops.
- **Container agent-runner isolation** — each group gets its own copy of agent-runner source in `data/sessions/{group}/agent-runner-src/`. Changes to `container/agent-runner/src/index.ts` only take effect for **new groups** or after manually copying. To propagate to existing groups, run `./container/build.sh` and wipe the per-group copies. For development, test with a new group or clear the copy.
- **`toolUseIdToAgentName` / `taskIdToAgentName` are module-level** in agent-runner — but each container run is a fresh process, so there's no cross-run contamination.
