# Design: Per-Group LLM Model Configuration

**Date:** 2026-03-20
**Status:** Approved

## Summary

Add per-group LLM model configuration to NanoClaw. Each group can have its own model override stored in the database. A global default falls back to `claude-sonnet-4-6`. Agents can change their own model at runtime via a new `set_model` MCP tool. Agent Teams (swarms) inherit the leader's model automatically.

---

## Architecture

Four layers, each with a single responsibility:

```
.env / process.env              → DEFAULT_MODEL (global fallback, claude-sonnet-4-6)
    ↓
SQLite registered_groups.model  → per-group override (NULL = use default)
    ↓
ContainerInput.model            → transports model into container
    ↓
agent-runner query(options.model) → subagents inherit via CLI process
```

---

## Data Flow

### Startup / Config

- `src/config.ts`: `export const DEFAULT_MODEL = process.env.DEFAULT_MODEL || 'claude-sonnet-4-6'`
  - Read directly from `process.env`, same pattern as `CONTAINER_IMAGE`, `CONTAINER_TIMEOUT` etc. — **not** added to the `readEnvFile` allowlist (which is only for vars needed before dotenv loads)
- `.env.example`: add `DEFAULT_MODEL=claude-sonnet-4-6` (documents the option)
- `.env` (optional): user override, e.g. `DEFAULT_MODEL=claude-opus-4-6`

### Spawning a container

- `src/index.ts` builds `ContainerInput` → reads `group.model ?? DEFAULT_MODEL` → writes `model` into input
- `src/container-runner.ts`: `ContainerInput` interface gets `model?: string` field

### Inside the agent-runner

- `container/agent-runner/src/index.ts` has its **own local copy** of `ContainerInput` (not imported from the host) — `model?: string` must be added there separately
- Reads `containerInput.model`, passes to `query({ options: { model: containerInput.model } })`
- Agent Teams subagents run inside the same Claude Code CLI subprocess as the leader; the `--model` flag is passed at CLI startup and applies to all team members. No additional code needed.

### Self-set via MCP tool

- Agent calls `set_model("claude-haiku-4-5")` on the nanoclaw MCP server
- MCP writes IPC file: `{ type: "set_model", model: "...", groupFolder: "..." }` to `/workspace/ipc/tasks/`
- Host IPC watcher processes it → updates `model` column in DB for this group
- Takes effect from the next container session onward
- Agent can pass `"default"` to reset to the system default

---

## Database Schema

New column on `registered_groups`:

```sql
ALTER TABLE registered_groups ADD COLUMN model TEXT;
-- NULL = no override, DEFAULT_MODEL is used
```

Migration follows the established try/catch pattern (same as `is_main`, `context_mode`):

```typescript
try {
  database.exec(`ALTER TABLE registered_groups ADD COLUMN model TEXT`);
} catch { /* column already exists */ }
```

### Impact on existing DB functions

`setRegisteredGroup` uses `INSERT OR REPLACE` with an explicit column list — it **must** include `model` or the column will be silently reset to NULL on every write. Both read functions (`getRegisteredGroup`, `getAllRegisteredGroups`) must also read and map the `model` column from the row.

---

## Type Changes

`src/types.ts` — `RegisteredGroup`:

```typescript
export interface RegisteredGroup {
  name: string;
  folder: string;
  trigger: string;
  added_at: string;
  containerConfig?: ContainerConfig;
  requiresTrigger?: boolean;
  isMain?: boolean;
  model?: string; // Override DEFAULT_MODEL for this group. undefined = use default
}
```

---

## DB Functions

`src/db.ts` — changes required:

1. **Migration** (try/catch, see above)
2. **`setRegisteredGroup`**: add `model` to the `INSERT OR REPLACE` column list and values
3. **`getRegisteredGroup`** and **`getAllRegisteredGroups`**: read `model` from row, map to `RegisteredGroup.model` (treat `null` as `undefined`)
4. **New function** `setGroupModel`:

```typescript
export function setGroupModel(folder: string, model: string | null): void {
  db.prepare(`UPDATE registered_groups SET model = ? WHERE folder = ?`).run(model, folder);
}
```

---

## MCP Tool: `set_model`

Added to `container/agent-runner/src/ipc-mcp-stdio.ts`:

```typescript
server.tool(
  'set_model',
  'Set the Claude model for this group. Takes effect on the next session.',
  {
    model: z.string().describe(
      'Model ID, e.g. "claude-opus-4-6", "claude-sonnet-4-6", "claude-haiku-4-5-20251001". ' +
      'Pass "default" to reset to the system default.'
    ),
  },
  async (args) => {
    writeIpcFile(TASKS_DIR, {
      type: 'set_model',
      model: args.model === 'default' ? null : args.model,
      groupFolder,
      timestamp: new Date().toISOString(),
    });
    return {
      content: [{ type: 'text', text: `Model set to ${args.model}. Takes effect next session.` }],
    };
  },
);
```

### IPC Handler

New `case 'set_model'` in `processTaskIpc` in `src/ipc.ts`:

```typescript
case 'set_model': {
  // Authorization: a group can only set its own model.
  // (The IPC directory identity is `sourceGroup`, so this is already enforced by
  // the directory-based IPC security model — no extra check needed.)
  deps.setRegisteredGroupModel(sourceGroup, data.model ?? null);
  break;
}
```

**Authorization policy:** A group can only set its own model. The main group cannot set another group's model via this tool — if that's needed in the future, a `targetFolder` field can be added. This is intentionally more restrictive than `register_group` (main-only) and consistent with task operations (each group manages its own).

### `processTaskIpc` type

The `data` parameter type must include `model`:

```typescript
// existing fields...
model?: string | null;
```

### `IpcDeps` interface

```typescript
setRegisteredGroupModel(folder: string, model: string | null): void;
```

---

## Swarm / Subagents

| Type | Behaviour |
|------|-----------|
| **Agent Teams** (`TeamCreate` / `SendMessage`) | Inherit leader's model — the `--model` flag is passed to the Claude Code CLI at startup and applies to the entire process including spawned team members |
| **Programmatic subagents** (`agents` option in `query()`) | Can use `model: 'inherit'` (SDK default) or override explicitly — already supported by SDK `AgentDefinition` |

---

## Files Changed

| File | Change |
|------|--------|
| `src/config.ts` | Add `DEFAULT_MODEL` export (via `process.env`, not `readEnvFile`) |
| `src/types.ts` | Add `model?: string` to `RegisteredGroup` |
| `src/db.ts` | Migration; update `setRegisteredGroup`, `getRegisteredGroup`, `getAllRegisteredGroups` for `model` column; add `setGroupModel()` |
| `src/ipc.ts` | Add `model?: string \| null` to `processTaskIpc` data type; add `set_model` case; add `setRegisteredGroupModel` to `IpcDeps` |
| `src/index.ts` | Pass `model` in `ContainerInput` construction (`group.model ?? DEFAULT_MODEL`) |
| `src/container-runner.ts` | Add `model?: string` to `ContainerInput` interface |
| `container/agent-runner/src/index.ts` | Add `model?: string` to local `ContainerInput` interface; read `model` from input; pass to `query()` |
| `container/agent-runner/src/ipc-mcp-stdio.ts` | Add `set_model` MCP tool |
| `.env.example` | Add `DEFAULT_MODEL=claude-sonnet-4-6` |
