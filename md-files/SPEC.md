# agy-swarm-bus: Specification

## Goal
Let several Google Antigravity chat tabs working on the same project coordinate through a shared local SQLite database, exposed as an MCP server. It provides: peer messaging, advisory file locks, and a shared task board.

## Key facts about how it runs
- Antigravity spawns **one stdio MCP server process per tab**. There is no single daemon.
- All processes share state through **one SQLite file in WAL mode**.
- Each process generates its own `session_id` (UUID) at startup and keeps it in memory. Agents never pass a session id.
- Locks are **advisory**: they only work if agents follow the rules in `AGENTS.md`.

## Tech stack
- Node 20+, TypeScript, compiled to `dist/index.js` (do not rely on running `.ts` directly)
- `@modelcontextprotocol/sdk` (stdio transport)
- `better-sqlite3` (synchronous, so atomic operations are simple)
- `zod` for tool input validation
- `vitest` for tests

## Project layout
```
agy-swarm-bus/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts        # MCP server, tool registration
│   ├── db.ts           # open DB, pragmas, schema, helpers
│   └── tools/          # one file per tool group (agents, locks, mail, tasks)
├── test/
│   ├── locks.test.ts
│   ├── mailbox.test.ts
│   └── tasks.test.ts
├── AGENTS.md           # rules for tabs (copy into the user's workspace)
└── mcp_config.example.json
```

## Configuration
- `SWARM_DB_PATH` env var: **must be an absolute path**. If it is relative, resolve it against `process.cwd()` and log a warning to stderr. Create the parent directory if missing.
- `SWARM_LOCK_TTL_SECONDS` (default `300`)
- `SWARM_AGENT_TTL_SECONDS` (default `120`): an agent with no heartbeat for this long is considered offline
- **Never write to stdout** except MCP protocol messages. All logs go to stderr.

## Database
On open, run:
```sql
PRAGMA journal_mode=WAL;
PRAGMA busy_timeout=5000;
PRAGMA foreign_keys=ON;
```
Schema:
```sql
CREATE TABLE IF NOT EXISTS agents (
  session_id TEXT PRIMARY KEY,
  role TEXT NOT NULL,
  current_task TEXT,
  last_heartbeat INTEGER NOT NULL      -- unix seconds
);

CREATE TABLE IF NOT EXISTS mailboxes (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  target_role TEXT NOT NULL,
  sender_role TEXT NOT NULL,
  payload TEXT NOT NULL,
  read_status INTEGER NOT NULL DEFAULT 0,
  created_at INTEGER NOT NULL
);
CREATE INDEX IF NOT EXISTS idx_mail_target ON mailboxes(target_role, read_status);

CREATE TABLE IF NOT EXISTS file_locks (
  filepath TEXT PRIMARY KEY,
  holder_session TEXT NOT NULL,
  holder_role TEXT NOT NULL,
  acquired_at INTEGER NOT NULL,
  expires_at INTEGER NOT NULL
);

CREATE TABLE IF NOT EXISTS shared_tasks (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  title TEXT NOT NULL,
  status TEXT NOT NULL CHECK(status IN ('todo','in_progress','done')) DEFAULT 'todo',
  assignee TEXT,
  updated_at INTEGER NOT NULL
);
```
Use unix seconds everywhere (not `CURRENT_TIMESTAMP` strings) so expiry math is simple.

## Path normalization
Before storing or comparing any `filepath`: resolve to an absolute path, normalize separators to `/`, and lowercase it on Windows. `src/App.tsx` and `./src/../src/App.tsx` must be the same lock.

## Tools

Every tool returns JSON as text content. Errors return `{ "ok": false, "error": "..." }` and never crash the process.

### 1. `register_agent(role, current_task?)`
- Upserts this process's row in `agents` (session id from memory).
- Sets `last_heartbeat = now`.
- Returns `{ ok, session_id, role }`.
- Roles are free-form strings like `backend-agent`, `frontend-agent`. Several sessions may share a role.

### 2. `acquire_file_lock(filepath)`
Must be atomic. Inside one transaction:
1. Delete expired locks (`expires_at < now`).
2. `INSERT ... ON CONFLICT(filepath) DO UPDATE SET expires_at = excluded.expires_at WHERE file_locks.holder_session = excluded.holder_session` (re-acquire by the same session just refreshes the expiry).
3. Read the row back.
- If the holder is this session: `{ ok: true, acquired: true, expires_at }`
- Otherwise: `{ ok: true, acquired: false, held_by: <role>, expires_at }` and a message telling the agent to wait or work on something else.
- Requires the agent to be registered. If not, return an error telling it to call `register_agent` first.

### 3. `release_file_lock(filepath)`
- Deletes the lock only if `holder_session` is this session.
- Returns `{ ok, released: boolean }`.
- Also add `release_all_locks()`, which deletes every lock held by this session.

### 4. `send_peer_message(target_role, payload)`
- Inserts a row with `sender_role` = this agent's role.
- `payload` max 10,000 characters; reject longer.
- `target_role` may be `"*"` for broadcast. Broadcasts are delivered to each role once (see `poll_mailbox`).
- Returns `{ ok, message_id }`.

### 5. `poll_mailbox()`
- Atomic delivery: `UPDATE mailboxes SET read_status=1 WHERE target_role = ? AND read_status = 0 RETURNING *`.
- Broadcast handling (simple version): messages with `target_role = '*'` are included only if their id is greater than this session's `last_broadcast_id` (kept in memory, initialized to the max broadcast id at registration time so a new tab does not replay history).
- Also refreshes this agent's heartbeat and extends the expiry of all locks it holds.
- Returns `{ ok, messages: [{ id, from, payload, created_at }] }`.

### 6. `claim_and_update_task(task_id, status)`
- `status` is one of `todo`, `in_progress`, `done`.
- Moving to `in_progress` is a **claim**: it succeeds only if the task is `todo`, or already assigned to this role. Use `UPDATE ... WHERE id=? AND (status='todo' OR assignee=?)` and check `changes`. If it fails, return `{ ok: true, claimed: false, assignee: <who has it> }`.
- Moving to `done` is allowed only for the assignee.
- Moving back to `todo` clears the assignee (only the assignee may do this).

### 7. Helpers (small but needed)
- `create_task(title)`: adds a `todo` task. Returns the id.
- `list_tasks(status?)`: returns tasks.
- `list_agents()`: returns online agents (heartbeat within `SWARM_AGENT_TTL_SECONDS`) and the locks they hold.

## Cleanup
- On startup and every 60 seconds: delete expired locks and agents offline for more than 1 hour.
- On `SIGINT`, `SIGTERM`, and stdin close: release this session's locks and remove its agent row, then exit.

## Non-goals (do not build)
- No network server, no web UI, no auth.
- No automatic enforcement of locks on file edits (impossible from an MCP server).
- No message history search.

## Acceptance criteria
- Two processes started against the same DB file: only one can acquire a lock on the same path.
- A lock from a killed process becomes available after the TTL.
- A message is delivered exactly once, even if two polls race.
- Two sessions racing to claim the same task: exactly one wins.
- Server logs nothing to stdout besides MCP protocol output.
