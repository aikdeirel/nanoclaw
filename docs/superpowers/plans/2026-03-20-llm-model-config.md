# LLM Model Configuration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add per-group LLM model configuration so each group can run a different Claude model, with a global default fallback and a `set_model` MCP tool for runtime changes.

**Architecture:** `DEFAULT_MODEL` is exported from `src/config.ts` and used as a fallback when no group-level override is set. The model is stored as a new `model TEXT` column in `registered_groups`, threaded through `ContainerInput` into the agent-runner, and passed to `query(options.model)`. A new `set_model` MCP tool writes an IPC file that the host processes to update the DB.

**Tech Stack:** TypeScript, SQLite (better-sqlite3), Vitest, `@anthropic-ai/claude-agent-sdk` query() options, `@modelcontextprotocol/sdk` for the MCP tool.

---

## File Map

| File | Change |
|------|--------|
| `src/config.ts` | Add `DEFAULT_MODEL` export |
| `src/types.ts` | Add `model?: string` to `RegisteredGroup` |
| `src/db.ts` | Migration + update `setRegisteredGroup`, `getRegisteredGroup`, `getAllRegisteredGroups` + add `setGroupModel()` |
| `src/ipc.ts` | Add `model` + `targetFolder` to data type; add `set_model` case; add `setRegisteredGroupModel` to `IpcDeps` |
| `src/index.ts` | Pass `model` in `ContainerInput` construction |
| `src/container-runner.ts` | Add `model?: string` to `ContainerInput` interface |
| `container/agent-runner/src/index.ts` | Add `model?: string` to local `ContainerInput`; pass to `query()` |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | Add `set_model` MCP tool |
| `.env.example` | Document `DEFAULT_MODEL=claude-sonnet-4-6` |
| `src/db.test.ts` | Tests for `model` round-trip and `setGroupModel` |
| `src/ipc.ts` tests (`src/ipc-auth.test.ts`) | Tests for `set_model` IPC dispatch |

---

## Task 1: Add `DEFAULT_MODEL` to config and `model` to types

**Files:**
- Modify: `src/config.ts`
- Modify: `src/types.ts`
- Modify: `.env.example`

- [ ] **Step 1: Add `DEFAULT_MODEL` to `src/config.ts`**

Add after the `CONTAINER_TIMEOUT` export (around line 43):

```typescript
export const DEFAULT_MODEL =
  process.env.DEFAULT_MODEL || 'claude-sonnet-4-6';
```

Do NOT add it to the `readEnvFile` call on line 9 — it is read directly from `process.env`, consistent with `CONTAINER_IMAGE`, `CONTAINER_TIMEOUT`, etc.

- [ ] **Step 2: Add `model` to `RegisteredGroup` in `src/types.ts`**

In the `RegisteredGroup` interface (line 35), add after `isMain?`:

```typescript
model?: string; // Claude model override. undefined = use DEFAULT_MODEL
```

- [ ] **Step 3: Document in `.env.example`**

Add this line to `.env.example` near the other container settings:

```bash
DEFAULT_MODEL=claude-sonnet-4-6
```

- [ ] **Step 4: Typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 5: Commit**

```bash
git add src/config.ts src/types.ts .env.example
git commit -m "feat: add DEFAULT_MODEL config and model field to RegisteredGroup"
```

---

## Task 2: DB migration and model round-trip

**Files:**
- Modify: `src/db.ts`
- Modify: `src/db.test.ts`

- [ ] **Step 1: Write failing tests in `src/db.test.ts`**

Add a new describe block at the end of the file:

```typescript
// --- RegisteredGroup model round-trip ---

describe('registered group model', () => {
  it('persists model through set/get round-trip', () => {
    setRegisteredGroup('tg:-1001@g.us', {
      name: 'Telegram Main',
      folder: 'telegram_main',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
      model: 'claude-opus-4-6',
    });

    const groups = getAllRegisteredGroups();
    const group = groups['tg:-1001@g.us'];
    expect(group).toBeDefined();
    expect(group.model).toBe('claude-opus-4-6');
  });

  it('returns undefined model when not set', () => {
    setRegisteredGroup('tg:-1002@g.us', {
      name: 'Other Group',
      folder: 'telegram_other',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
    });

    const groups = getAllRegisteredGroups();
    expect(groups['tg:-1002@g.us'].model).toBeUndefined();
  });

  it('setGroupModel updates model for existing group', () => {
    setRegisteredGroup('tg:-1003@g.us', {
      name: 'Group3',
      folder: 'telegram_group3',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
    });

    setGroupModel('telegram_group3', 'claude-haiku-4-5');

    const groups = getAllRegisteredGroups();
    expect(groups['tg:-1003@g.us'].model).toBe('claude-haiku-4-5');
  });

  it('setGroupModel with null resets to undefined (use default)', () => {
    setRegisteredGroup('tg:-1004@g.us', {
      name: 'Group4',
      folder: 'telegram_group4',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
      model: 'claude-opus-4-6',
    });

    setGroupModel('telegram_group4', null);

    const groups = getAllRegisteredGroups();
    expect(groups['tg:-1004@g.us'].model).toBeUndefined();
  });

  it('model survives a setRegisteredGroup overwrite', () => {
    setRegisteredGroup('tg:-1005@g.us', {
      name: 'Group5',
      folder: 'telegram_group5',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
      model: 'claude-opus-4-6',
    });

    // Overwrite with same model field present
    setRegisteredGroup('tg:-1005@g.us', {
      name: 'Group5 Updated',
      folder: 'telegram_group5',
      trigger: '@Andy',
      added_at: '2024-01-01T00:00:00.000Z',
      model: 'claude-opus-4-6',
    });

    const groups = getAllRegisteredGroups();
    expect(groups['tg:-1005@g.us'].model).toBe('claude-opus-4-6');
    expect(groups['tg:-1005@g.us'].name).toBe('Group5 Updated');
  });
});
```

Also add `setGroupModel` to the imports at the top of `src/db.test.ts`.

- [ ] **Step 2: Run tests to verify they fail**

```bash
npm test -- src/db.test.ts
```

Expected: test failures — `setGroupModel` not found, model not persisted.

- [ ] **Step 3: Add DB migration in `src/db.ts`**

In `createSchema()`, after the existing `is_main` migration block (around line 120), add:

```typescript
// Add model column if it doesn't exist (migration for existing DBs)
try {
  database.exec(`ALTER TABLE registered_groups ADD COLUMN model TEXT`);
} catch {
  /* column already exists */
}
```

- [ ] **Step 4: Update `setRegisteredGroup` to include `model`**

In `setRegisteredGroup` (around line 586), update the INSERT OR REPLACE:

```typescript
db.prepare(
  `INSERT OR REPLACE INTO registered_groups (jid, name, folder, trigger_pattern, added_at, container_config, requires_trigger, is_main, model)
   VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?)`,
).run(
  jid,
  group.name,
  group.folder,
  group.trigger,
  group.added_at,
  group.containerConfig ? JSON.stringify(group.containerConfig) : null,
  group.requiresTrigger === undefined ? 1 : group.requiresTrigger ? 1 : 0,
  group.isMain ? 1 : 0,
  group.model ?? null,
);
```

- [ ] **Step 5: Update `getRegisteredGroup` to read `model`**

The raw row type (around line 548) needs `model: string | null`:

```typescript
| {
    jid: string;
    name: string;
    folder: string;
    trigger_pattern: string;
    added_at: string;
    container_config: string | null;
    requires_trigger: number | null;
    is_main: number | null;
    model: string | null;
  }
```

And in the return object (around line 567), add:

```typescript
model: row.model ?? undefined,
```

- [ ] **Step 6: Update `getAllRegisteredGroups` to read `model`**

The row type in `getAllRegisteredGroups` (around line 602) needs the same `model: string | null` field. In the result assignment (around line 621), add:

```typescript
model: row.model ?? undefined,
```

- [ ] **Step 7: Add `setGroupModel` function to `src/db.ts`**

Add after `setRegisteredGroup`:

```typescript
export function setGroupModel(folder: string, model: string | null): void {
  db.prepare(
    `UPDATE registered_groups SET model = ? WHERE folder = ?`,
  ).run(model, folder);
}
```

- [ ] **Step 8: Run tests to verify they pass**

```bash
npm test -- src/db.test.ts
```

Expected: all tests pass.

- [ ] **Step 9: Run full test suite**

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 10: Commit**

```bash
git add src/db.ts src/db.test.ts
git commit -m "feat: add model column to registered_groups with round-trip tests"
```

---

## Task 3: Thread model through ContainerInput into agent-runner

**Files:**
- Modify: `src/container-runner.ts`
- Modify: `src/index.ts`
- Modify: `container/agent-runner/src/index.ts`

- [ ] **Step 1: Add `model` to `ContainerInput` in `src/container-runner.ts`**

In the `ContainerInput` interface (around line 36), add:

```typescript
model?: string;
```

- [ ] **Step 2: Pass `model` when building ContainerInput in `src/index.ts`**

In the `runContainerAgent` call (around line 521), add `model` to the input object:

```typescript
{
  prompt,
  sessionId,
  groupFolder: group.folder,
  chatJid,
  isMain,
  assistantName: ASSISTANT_NAME,
  model: group.model ?? DEFAULT_MODEL,
  ...(imageAttachments.length > 0 && { imageAttachments }),
},
```

Also import `DEFAULT_MODEL` at the top of `src/index.ts`:

```typescript
import { ..., DEFAULT_MODEL } from './config.js';
```

- [ ] **Step 3: Add `model` to local `ContainerInput` in `container/agent-runner/src/index.ts`**

The agent-runner has its own local copy of `ContainerInput` (line 22). Add `model?`:

```typescript
interface ContainerInput {
  prompt: string;
  sessionId?: string;
  groupFolder: string;
  chatJid: string;
  isMain: boolean;
  isScheduledTask?: boolean;
  assistantName?: string;
  model?: string;
  imageAttachments?: Array<{ relativePath: string; mediaType: string }>;
}
```

- [ ] **Step 4: Pass `model` to `query()` in the agent-runner**

In the `query()` call (around line 431), add `model` to the options object:

```typescript
for await (const message of query({
  prompt: stream,
  options: {
    model: containerInput.model,
    cwd: '/workspace/group',
    // ... rest of existing options unchanged
  }
})) {
```

- [ ] **Step 5: Typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add src/container-runner.ts src/index.ts container/agent-runner/src/index.ts
git commit -m "feat: thread model through ContainerInput to agent-runner query()"
```

---

## Task 4: IPC handler for `set_model`

**Files:**
- Modify: `src/ipc.ts`
- Modify: `src/ipc-auth.test.ts` (or add to existing IPC test file)

- [ ] **Step 1: Write failing test**

Find the IPC test file. Check what exists:

```bash
cat src/ipc-auth.test.ts | head -50
```

Add a new describe block for `set_model`. The test should verify that the IPC handler calls `setRegisteredGroupModel` with the right arguments:

```typescript
describe('processTaskIpc set_model', () => {
  it('updates model for own group', async () => {
    let calledWith: { folder: string; model: string | null } | undefined;

    const deps = {
      sendMessage: async () => {},
      registeredGroups: () => ({}),
      registerGroup: () => {},
      syncGroups: async () => {},
      getAvailableGroups: () => [],
      writeGroupsSnapshot: () => {},
      onTasksChanged: () => {},
      setRegisteredGroupModel: (folder: string, model: string | null) => {
        calledWith = { folder, model };
      },
    };

    await processTaskIpc(
      { type: 'set_model', model: 'claude-opus-4-6', groupFolder: 'telegram_main' },
      'telegram_main',
      false,
      deps,
    );

    expect(calledWith).toEqual({ folder: 'telegram_main', model: 'claude-opus-4-6' });
  });

  it('main can set model for another group via targetFolder', async () => {
    let calledWith: { folder: string; model: string | null } | undefined;

    const deps = {
      sendMessage: async () => {},
      registeredGroups: () => ({}),
      registerGroup: () => {},
      syncGroups: async () => {},
      getAvailableGroups: () => [],
      writeGroupsSnapshot: () => {},
      onTasksChanged: () => {},
      setRegisteredGroupModel: (folder: string, model: string | null) => {
        calledWith = { folder, model };
      },
    };

    await processTaskIpc(
      { type: 'set_model', model: 'claude-haiku-4-5', targetFolder: 'telegram_other', groupFolder: 'main' },
      'main',
      true,
      deps,
    );

    expect(calledWith).toEqual({ folder: 'telegram_other', model: 'claude-haiku-4-5' });
  });

  it('non-main cannot target another group', async () => {
    let calledWith: { folder: string; model: string | null } | undefined;

    const deps = {
      sendMessage: async () => {},
      registeredGroups: () => ({}),
      registerGroup: () => {},
      syncGroups: async () => {},
      getAvailableGroups: () => [],
      writeGroupsSnapshot: () => {},
      onTasksChanged: () => {},
      setRegisteredGroupModel: (folder: string, model: string | null) => {
        calledWith = { folder, model };
      },
    };

    await processTaskIpc(
      { type: 'set_model', model: 'claude-opus-4-6', targetFolder: 'telegram_other', groupFolder: 'telegram_main' },
      'telegram_main',
      false,
      deps,
    );

    // targetFolder ignored for non-main; updates own group
    expect(calledWith).toEqual({ folder: 'telegram_main', model: 'claude-opus-4-6' });
  });

  it('resets to null when model is null (MCP tool converts "default" → null before writing IPC)', async () => {
    let calledWith: { folder: string; model: string | null } | undefined;

    const deps = {
      sendMessage: async () => {},
      registeredGroups: () => ({}),
      registerGroup: () => {},
      syncGroups: async () => {},
      getAvailableGroups: () => [],
      writeGroupsSnapshot: () => {},
      onTasksChanged: () => {},
      setRegisteredGroupModel: (folder: string, model: string | null) => {
        calledWith = { folder, model };
      },
    };

    // The MCP tool translates "default" → null before writing the IPC file,
    // so by the time the IPC handler sees it, null already means "reset to default".
    await processTaskIpc(
      { type: 'set_model', model: null, groupFolder: 'telegram_main' },
      'telegram_main',
      false,
      deps,
    );

    expect(calledWith).toEqual({ folder: 'telegram_main', model: null });
  });
});
```

Also ensure `processTaskIpc` is imported in the test file.

- [ ] **Step 2: Run test to verify it fails**

```bash
npm test -- src/ipc-auth.test.ts
```

Expected: FAIL — `setRegisteredGroupModel` not in `IpcDeps`, `set_model` case missing.

- [ ] **Step 3: Add `model` and `targetFolder` to `processTaskIpc` data type in `src/ipc.ts`**

In the `processTaskIpc` function signature (around line 167), the `data` parameter type needs two new optional fields:

```typescript
model?: string | null;
targetFolder?: string;
```

- [ ] **Step 4: Add `setRegisteredGroupModel` to `IpcDeps` interface**

In `IpcDeps` (around line 14), add:

```typescript
setRegisteredGroupModel(folder: string, model: string | null): void;
```

- [ ] **Step 5: Add `set_model` case to `processTaskIpc`**

In the `switch (data.type)` block, add:

```typescript
case 'set_model': {
  // Main can target any group; others can only update their own
  const targetFolder =
    isMain && data.targetFolder ? data.targetFolder : sourceGroup;
  deps.setRegisteredGroupModel(targetFolder, data.model ?? null);
  break;
}
```

- [ ] **Step 6: Wire up `setRegisteredGroupModel` in `src/index.ts`**

Find where `startIpcWatcher` is called with the `deps` object. Add `setRegisteredGroupModel` to the deps:

```typescript
setRegisteredGroupModel: (folder: string, model: string | null) => {
  setGroupModel(folder, model);
  // Refresh in-memory groups map
  groups = getAllRegisteredGroups();
},
```

Also import `setGroupModel` from `./db.js` at the top of `src/index.ts`.

(Check how `registeredGroups` is managed in index.ts — use the same pattern used for `registerGroup` to keep the in-memory map fresh.)

- [ ] **Step 7: Run IPC tests**

```bash
npm test -- src/ipc-auth.test.ts
```

Expected: all new tests pass.

- [ ] **Step 8: Run full test suite**

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 9: Typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 10: Commit**

```bash
git add src/ipc.ts src/ipc-auth.test.ts src/index.ts
git commit -m "feat: add set_model IPC handler with main group cross-group support"
```

---

## Task 5: `set_model` MCP tool in the agent-runner

**Files:**
- Modify: `container/agent-runner/src/ipc-mcp-stdio.ts`

No automated tests for the MCP server (it's a thin IPC wrapper — the logic is tested via IPC tests in Task 4). Manual verification in Task 6.

- [ ] **Step 1: Add `set_model` tool to `container/agent-runner/src/ipc-mcp-stdio.ts`**

Add after the `register_group` tool (before the `StdioServerTransport` lines at the bottom):

```typescript
server.tool(
  'set_model',
  `Set the Claude model for this group. Takes effect on the next session.

Use this when the user wants to switch the AI model for this conversation group.
Pass "default" to reset to the system default (claude-sonnet-4-6).`,
  {
    model: z.string().describe(
      'Model ID to use, e.g. "claude-opus-4-6", "claude-sonnet-4-6", "claude-haiku-4-5-20251001". ' +
      'Pass "default" to reset to the system default.',
    ),
    target_folder: z.string().optional().describe(
      '(Main group only) Folder name of the group to update, e.g. "telegram_main". ' +
      'Defaults to the current group.',
    ),
  },
  async (args) => {
    writeIpcFile(TASKS_DIR, {
      type: 'set_model',
      model: args.model === 'default' ? null : args.model,
      targetFolder: args.target_folder,
      groupFolder,
      timestamp: new Date().toISOString(),
    });

    const target = args.target_folder ?? 'this group';
    const modelName = args.model === 'default' ? 'system default' : args.model;
    return {
      content: [
        {
          type: 'text' as const,
          text: `Model for ${target} set to ${modelName}. Takes effect on the next session.`,
        },
      ],
    };
  },
);
```

- [ ] **Step 2: Typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 3: Commit**

```bash
git add container/agent-runner/src/ipc-mcp-stdio.ts
git commit -m "feat: add set_model MCP tool to agent-runner"
```

---

## Task 6: Manual smoke test

The agent-runner runs inside a container — there are no unit tests for the container-level code. Verify the full flow manually.

- [ ] **Step 1: Rebuild the agent container**

```bash
./container/build.sh
```

- [ ] **Step 2: Start the service**

```bash
# Linux (systemd)
systemctl --user restart nanoclaw

# macOS (launchd)
launchctl kickstart -k gui/$(id -u)/com.nanoclaw
```

Or run in foreground for easier debugging:

```bash
npm run dev
```

- [ ] **Step 3: Verify default model is passed**

Check a container log after a message is processed. The `model` field should appear in the container input log:

```bash
cat groups/telegram_main/logs/container-*.log | grep -A5 "=== Input ==="
```

Expected: `"model": "claude-sonnet-4-6"` in the input JSON (or whatever DEFAULT_MODEL is set to).

- [ ] **Step 4: Test `set_model` from the agent**

Send a message to the main group:
```
@Andy set your model to claude-opus-4-6
```

Expected: agent calls `set_model`, responds with confirmation.

- [ ] **Step 5: Verify DB update**

```bash
sqlite3 store/messages.db "SELECT folder, model FROM registered_groups;"
```

Expected: the group's `model` column shows `claude-opus-4-6`.

- [ ] **Step 6: Verify new session uses new model**

Send another message and check the container log. The `model` field in the input should now be `claude-opus-4-6`.

- [ ] **Step 7: Test reset to default**

Send:
```
@Andy reset your model to default
```

Expected: agent calls `set_model("default")`, DB resets to NULL, next session uses `DEFAULT_MODEL`.

- [ ] **Step 8: Commit if any fixes were needed**

```bash
git add -p
git commit -m "fix: <describe what needed fixing from smoke test>"
```

---

## Task 7: Final cleanup

- [ ] **Step 1: Run full test suite one last time**

```bash
npm test
```

Expected: all tests pass.

- [ ] **Step 2: Typecheck**

```bash
npm run typecheck
```

Expected: no errors.

- [ ] **Step 3: Final commit if anything changed**

```bash
git add .
git commit -m "chore: final cleanup for model config feature"
```
