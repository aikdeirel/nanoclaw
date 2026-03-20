# Streaming Status Bubble Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable real-time status bubble updates during Claude's thinking phase by switching the agent-runner from batched `assistant` events to streaming `stream_event` events via `includePartialMessages: true`.

**Architecture:** Add `includePartialMessages: true` to the `query()` options so the SDK emits `stream_event` messages (`SDKPartialAssistantMessage`) for every token delta. Handle `content_block_start` and `content_block_delta` events to emit status updates immediately as Claude thinks or types. The existing `assistant` event handler remains unchanged — it still fires at message completion and handles tool-use tracking for swarms.

**Tech Stack:** TypeScript, `@anthropic-ai/claude-agent-sdk` `query()`, `BetaRawMessageStreamEvent` (field names verified against SDK `.d.ts`: `content_block_start.content_block`, `content_block_delta.delta` with subtypes `thinking_delta.thinking`, `text_delta.text`, `input_json_delta.partial_json`)

**Known limitation:** `stream_event` messages arrive before the `assistant` event that populates `toolUseIdToAgentName`. Sub-agent streaming deltas will always resolve to `'main'` as agent name. Main-agent thinking/text streaming is unaffected.

---

## File Map

| File | Change |
|------|--------|
| `container/agent-runner/src/index.ts` | Add `includePartialMessages: true` to query options; add `stream_event` handler before query loop |

No other files change — the status protocol, `StatusBubble`, and Telegram channel are untouched.

---

### Task 1: Implement stream_event → status emission

**Files:**
- Modify: `container/agent-runner/src/index.ts` (query options + new handler inside for-await loop)

- [ ] **Step 1: Add `includePartialMessages: true` to query options**

In `container/agent-runner/src/index.ts`, inside the `options: { ... }` block passed to `query()`, add after the `hooks` property (around line 496):

```typescript
includePartialMessages: true,
```

- [ ] **Step 2: Add throttle state variables before the query loop**

Directly above the `for await (const message of query(...))` line, add:

```typescript
// Streaming status throttle state (reset per content block via content_block_start)
let streamStatusLastText = '';
let streamStatusLastEmitLen = 0;
const STREAM_THROTTLE_CHARS = 50;
```

- [ ] **Step 3: Add the stream_event handler inside the for-await loop**

Add this block **before** the existing `// Emit status events for assistant messages` comment (i.e., before the `if (message.type === 'assistant')` block):

```typescript
// Handle streaming partial events for real-time status updates during thinking/generation
if (message.type === 'stream_event') {
  const partial = message as {
    type: 'stream_event';
    parent_tool_use_id: string | null;
    event: {
      type: string;
      content_block?: { type: string; name?: string };
      delta?: { type: string; thinking?: string; text?: string; partial_json?: string };
    };
  };
  const ev = partial.event;
  // Note: parent_tool_use_id is not yet in toolUseIdToAgentName at stream time
  // (the assistant event that populates it fires after streaming completes).
  // Sub-agent stream events therefore always resolve to 'main' — known limitation.
  const currentAgentName = (partial.parent_tool_use_id && toolUseIdToAgentName.get(partial.parent_tool_use_id)) || 'main';

  if (ev.type === 'content_block_start') {
    // Reset accumulator on each new content block
    streamStatusLastText = '';
    streamStatusLastEmitLen = 0;
    if (ev.content_block?.type === 'tool_use' && ev.content_block.name) {
      writeOutput({ type: 'status', agentName: currentAgentName, text: ev.content_block.name });
    } else if (ev.content_block?.type === 'thinking') {
      writeOutput({ type: 'status', agentName: currentAgentName, text: '💭 ...' });
    }
  }

  if (ev.type === 'content_block_delta' && ev.delta) {
    const delta = ev.delta;
    if (delta.type === 'thinking_delta' && delta.thinking) {
      streamStatusLastText += delta.thinking;
    } else if (delta.type === 'text_delta' && delta.text) {
      streamStatusLastText += delta.text;
    } else if (delta.type === 'input_json_delta' && delta.partial_json) {
      streamStatusLastText += delta.partial_json;
    }

    const grown = streamStatusLastText.length - streamStatusLastEmitLen;
    if (grown >= STREAM_THROTTLE_CHARS) {
      streamStatusLastEmitLen = streamStatusLastText.length;
      const snippet = streamStatusLastText.slice(-80); // last 80 chars = most recent context
      const label = delta.type === 'thinking_delta' ? '💭' : delta.type === 'input_json_delta' ? '⚙️' : '✍️';
      writeOutput({ type: 'status', agentName: currentAgentName, text: `${label} ${snippet}` });
    }
  }
}
```

- [ ] **Step 4: Rebuild the container image**

```bash
./container/build.sh
```

Expected: `Build complete! Image: nanoclaw-agent:latest`

- [ ] **Step 5: Restart and test with a thinking-heavy prompt**

```bash
systemctl --user restart nanoclaw
```

Send to agent: "Think step by step through 5 distinct pros and cons of TypeScript vs JavaScript."

Expected behavior:
- Status bubble appears immediately (from `init()` in `StatusBubble`)
- Bubble updates every ~50 chars as Claude thinks: `💭 TypeScript provides static...`
- When a tool is called: `WebSearch` appears
- Bubble disappears when final answer arrives

- [ ] **Step 6: Test that simple messages still work**

Send to agent: "Hallo"

Expected: Bubble appears (from `init()`), updates once or twice with `✍️ ...`, disappears when answer arrives. No flood of Telegram edits.

- [ ] **Step 7: Verify no debug code was introduced**

```bash
grep -n 'console\.log\|TODO\|FIXME\|debug' container/agent-runner/src/index.ts
```

Expected: no unexpected output.

- [ ] **Step 8: Commit**

```bash
git add container/agent-runner/src/index.ts
git commit -m "feat(streaming): emit real-time status updates from thinking/text stream deltas"
```

---

## Edge Cases & Notes

| Scenario | Handling |
|----------|----------|
| `content_block_start` resets accumulator | `streamStatusLastText = ''` + `streamStatusLastEmitLen = 0` — clean per block |
| 429 on Telegram edit (fast generation) | `editStatusMessage` silently ignores 429 — bubble lags but no crash |
| Sub-agent `parent_tool_use_id` | Map not populated yet at stream time → always `'main'` (known limitation) |
| `message_delta` / `message_stop` / `content_block_stop` events | Ignored — no action needed |
| Simple "Hallo" (no thinking, short text) | 1-2 status updates then bubble deleted — acceptable |
| Multiple `query()` turns (IPC follow-up) | Throttle state reset per `content_block_start` — safe across turns |

## Container Build Reminder

The container buildkit caches aggressively. Normal `./container/build.sh` is sufficient. To force truly clean rebuild: prune builder volume first.
