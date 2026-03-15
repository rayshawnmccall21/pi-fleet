# Pi Fleet Architecture Review

**Reviewer:** Claude (Bridge Server Owner)  
**Date:** March 15, 2026  
**Focus:** Missing pieces, design changes, hard problems, and bridge server recommendations

---

## Executive Summary

Pi Fleet is a well-conceived distributed agent orchestration system. The PRD is thorough and the high-level architecture is sound. However, after reviewing the existing bridge-standalone.js prototype and the PRD's proposed architecture, I see several areas that need attention before we can build a production-quality system.

**My role in this project:** I will own and run the bridge server. This review reflects that perspective — I'm optimizing for something I can debug at 2am, keep running for weeks without restart, and extend without breaking.

**Overall Assessment:** Strong vision, needs tighter scope and more operational detail.

---

## What's Missing

### 1. **Process Lifecycle Guarantees**

The PRD says "tmux-backed Pi sessions" but doesn't address what happens when:

- Pi crashes mid-task (not just "auto-restart" — what state do we recover?)
- The bridge server itself restarts (do we reconnect to existing tmux sessions or recreate?)
- A Pi session hangs (not crashed, just stuck waiting for input)
- Multiple bridge restarts happen in quick succession

**What we need:** Explicit state recovery protocol. When bridge starts, it should:

1. Scan existing tmux sessions
2. Match them to persisted task state
3. Re-establish health monitoring
4. Decide: resume, restart, or mark as orphaned

### 2. **Rate Limiting and Backpressure**

The PRD mentions rate limiting in the security section but never specifies:

- What happens when 5 agents all try to run `pnpm test` simultaneously?
- How do we prevent a runaway agent from creating 1000 knowledge entries?
- What's the backpressure strategy when the task queue exceeds capacity?

**What we need:** Resource budgets per session, queue depth limits, and verification concurrency caps.

### 3. **Deployment and Installation**

There's no mention of how the bridge server gets installed on a worker device. Looking at bridge-standalone.js, it hardcodes paths like:

```javascript
const PI = "/Users/autumnraymccall/.nvm/versions/node/v24.14.0/bin/pi";
const TMUX = "/usr/local/bin/tmux";
```

This won't work on any other machine. We need:

- A setup script that discovers tool locations
- Environment validation on startup
- Graceful degradation when optional tools are missing

### 4. **Log Rotation and Disk Management**

The bridge server writes to `.bridge/` directory with no cleanup strategy. Knowledge files, task records, results, and mailboxes will grow unbounded. After a month of operation, this directory could be gigabytes.

### 5. **WebSocket Support**

The PRD mentions WebSocket transport in the protocol layer but the bridge server is HTTP-only. For real-time agent output streaming (critical for observability), we need WebSocket from day one — not as an afterthought.

### 6. **Graceful Shutdown**

No mention of how to shut down cleanly:

- Finish in-flight verifications?
- Wait for agents to reach a safe point?
- Persist pending mailbox messages?
- Close database connections?

---

## What I Would Change

### 1. **Drop tmux, Use node-pty Directly**

**Current approach:** Spawn Pi in tmux, interact via `tmux send-keys` and `tmux capture-pane`.

**Problems with tmux:**

- `capture-pane` returns formatted terminal output (ANSI codes, line wrapping) — parsing this for structured data is fragile
- `send-keys` with special characters requires careful escaping
- tmux sessions survive bridge restarts but we lose context about what was happening
- Debugging means SSHing in and attaching to tmux manually

**Proposed change:** Use `node-pty` (or `@aspect-build/pty`) to manage processes directly:

```typescript
class AgentSession {
  private pty: IPty;
  private buffer: CircularBuffer<string>;
  private state: SessionState;

  constructor(config: SessionConfig) {
    this.pty = spawn(config.command, [], {
      name: "xterm-256color",
      cwd: config.workingDirectory,
      env: { ...process.env, ...config.env },
      cols: 120,
      rows: 40,
    });

    this.pty.onData((data) => {
      this.buffer.push(data);
      this.emit("output", data);
    });

    this.pty.onExit(({ exitCode, signal }) => {
      this.state = exitCode === 0 ? "completed" : "crashed";
      this.emit("exit", { exitCode, signal });
    });
  }
}
```

**Benefits:**

- Direct access to raw output stream (no ANSI stripping needed)
- Process lifecycle events (onExit, onData)
- Can set resource limits via options
- WebSocket can stream output in real-time without polling
- Easier to test (mock the pty)

**Trade-off:** Lose session persistence across bridge restarts. But this is actually a feature — a clean restart with fresh state is better than half-recovering corrupted sessions.

### 2. **Simplify the Bridge to Two Modes**

The PRD proposes many modules (sessions, tasks, verification, git, knowledge, mailbox, results). That's a lot of surface area for something that needs to be rock-solid.

**Proposed architecture — just two concerns:**

```
bridge/
├── src/
│   ├── api/
│   │   ├── server.ts        # HTTP + WebSocket server
│   │   ├── routes.ts        # All routes (keep it flat)
│   │   ├── middleware.ts     # Auth, logging, validation
│   │   └── websocket.ts     # Real-time output streaming
│   ├── core/
│   │   ├── sessions.ts      # Process lifecycle (node-pty)
│   │   ├── tasks.ts         # Task state machine
│   │   ├── verification.ts  # Run verification commands
│   │   └── git.ts           # Git operations via simple-git
│   ├── storage/
│   │   ├── sqlite.ts        # All persistent state (one DB)
│   │   └── filesystem.ts    # Large blobs (knowledge values, output logs)
│   └── index.ts             # Entry point with health checks
```

**Key decision:** Use SQLite (via better-sqlite3) for ALL persistent state:

- Sessions and their metadata
- Tasks and state transitions
- Knowledge entries
- Mailbox messages
- Verification results
- Audit log

This gives us:

- ACID transactions
- Fast queries with indexes
- Single file backup (`cp fleet.db fleet.db.bak`)
- No concurrent write issues (SQLite handles it)
- Built-in full-text search (FTS5) for knowledge

### 3. **Make the Bridge Stateless-ish**

The bridge should be designed to be killed and restarted cleanly:

```typescript
class BridgeServer {
  async start() {
    // 1. Initialize database (create tables if needed)
    await this.storage.initialize();

    // 2. Clean up any orphaned state from previous run
    await this.cleanup();

    // 3. Start HTTP server
    this.server = this.app.listen(PORT);

    // 4. Start WebSocket server
    this.wss = new WebSocketServer({ server: this.server });

    // 5. Log ready
    logger.info(`Bridge ready on port ${PORT}`);
  }

  async shutdown() {
    // 1. Stop accepting new connections
    this.server.close();

    // 2. Signal all sessions to save state
    for (const session of this.sessions.values()) {
      await session.checkpoint();
    }

    // 3. Wait for in-flight verifications (with timeout)
    await this.verification.drain(30_000);

    // 4. Close database
    await this.storage.close();

    process.exit(0);
  }
}
```

### 4. **Unified Configuration**

Instead of hardcoded paths, use a cascading config:

```typescript
// Discovery order:
// 1. Environment variables (PI_FOO_BAR)
// 2. ~/.pi-fleet/config.yaml
// 3. ./pi-fleet.yaml (project level)
// 4. Defaults

const config = {
  bridge: {
    port: parseInt(process.env.PI_BRIDGE_PORT || "4001"),
    host: process.env.PI_BRIDGE_HOST || "0.0.0.0",
  },
  tools: {
    pi: process.env.PI_BIN || (await which("pi")),
    tmux: process.env.TMUX_BIN || (await which("tmux")),
    node: process.env.NODE_BIN || process.execPath,
    git: process.env.GIT_BIN || (await which("git")),
  },
  project: {
    root: process.env.PI_PROJECT_ROOT || process.cwd(),
    // ...
  },
};
```

### 5. **Structured Event Stream**

Replace the ad-hoc result collection with a proper event stream:

```typescript
interface FleetEvent {
  id: string;
  timestamp: string;
  type:
    | "session.created"
    | "session.destroyed"
    | "session.output"
    | "task.assigned"
    | "task.started"
    | "task.completed"
    | "task.failed"
    | "verification.started"
    | "verification.passed"
    | "verification.failed"
    | "git.branch.created"
    | "git.committed"
    | "git.pr.created"
    | "knowledge.stored"
    | "mailbox.delivered";
  source: { device: string; session?: string };
  data: Record<string, unknown>;
}

// Bridge stores events in SQLite
// Orchestrator subscribes via WebSocket or polls via HTTP
// Human can tail events via CLI
```

---

## Hardest Parts to Build

### 1. **Reliable Output Capture and Parsing** — Difficulty: 9/10

**Why it's hard:** Pi agents output to a terminal. Terminal output contains:

- ANSI escape codes (colors, cursor movement)
- Line wrapping at terminal width
- Progress indicators that update in-place
- Mixed content (prompts, code, tool output, cost displays)

Extracting structured information (task completion, cost, errors) from this stream is extremely fragile.

**My approach:**

- Use `node-pty` for raw output capture
- Stream output to a buffer AND to WebSocket simultaneously
- Don't try to parse in real-time — store raw, parse on-demand
- Define a simple "result protocol" that agents can optionally use:
  ```
  [PI-FLEET:RESULT] {"status": "done", "summary": "..."}
  ```
- Fall back to heuristic parsing only when structured output isn't present

### 2. **Task Verification Reliability** — Difficulty: 8/10

**Why it's hard:** Verification commands can:

- Hang indefinitely (test waiting for input)
- Consume all resources (infinite loop, memory leak)
- Succeed but produce misleading output
- Depend on environment state that changes
- Take wildly different times (10ms to 10 minutes)

**My approach:**

```typescript
class VerificationRunner {
  async run(command: string, options: VerifyOptions): Promise<VerifyResult> {
    const controller = new AbortController();
    const timeout = setTimeout(
      () => controller.abort(),
      options.timeout || 120_000
    );

    try {
      const proc = exec(command, {
        cwd: options.cwd,
        signal: controller.signal,
        maxBuffer: 10 * 1024 * 1024, // 10MB
        env: { ...process.env, ...options.env },
      });

      // Capture stdout and stderr separately
      let stdout = "";
      let stderr = "";
      proc.stdout?.on("data", (d) => (stdout += d));
      proc.stderr?.on("data", (d) => (stderr += d));

      const exitCode = await new Promise<number>((resolve, reject) => {
        proc.on("close", resolve);
        proc.on("error", reject);
      });

      clearTimeout(timeout);

      return {
        success: exitCode === 0,
        exitCode,
        stdout: stdout.slice(-50_000), // Last 50KB
        stderr: stderr.slice(-10_000),
        duration: Date.now() - start,
      };
    } catch (err) {
      clearTimeout(timeout);
      if (controller.signal.aborted) {
        return { success: false, error: "timeout", exitCode: -1 };
      }
      throw err;
    }
  }
}
```

### 3. **Concurrent Session Management** — Difficulty: 7/10

**Why it's hard:** Multiple agents on one machine means:

- Shared filesystem (potential conflicts)
- Shared git repo (branch collisions)
- Shared CPU/memory (resource contention)
- Shared environment variables (pollution)
- Process tree management (child processes of Pi)

**My approach:**

- One git worktree per session (enforced)
- Resource limits via ulimit (Linux) / launchctl (macOS)
- Session-specific environment variables
- Process group tracking for cleanup
- Hard limit on concurrent sessions (configurable, default 5)

### 4. **Graceful Degradation** — Difficulty: 7/10

**Why it's hard:** The bridge needs to work when:

- tmux is not installed (fall back to direct process management)
- git is not installed (skip git features, log warning)
- Disk is nearly full (reject new sessions, warn existing)
- Memory pressure (throttle new verifications)
- Network is slow (adjust timeouts)

**My approach:** Startup validation + runtime capability flags:

```typescript
interface Capabilities {
  tmux: boolean;
  git: boolean;
  gh: boolean;
  pnpm: boolean;
  diskSpace: number;
  memory: number;
  maxSessions: number;
}

// On startup, test each capability
// Expose via /api/bridge/capabilities
// Orchestrator checks before assigning tasks
```

---

## Bridge Server Recommendations

Since I will run this server, here are my specific recommendations:

### 1. **Keep It Simple — Really Simple**

The existing bridge-standalone.js is 400 lines and works. The PRD proposes 15+ modules. That's too much for v1.

**My recommendation:** Start with a single file (< 1000 lines) that handles:

- Session management (create, list, send, read, kill)
- Task tracking (assign, update, list)
- Verification (run command, return result)
- Health endpoint

Add knowledge, mailbox, and git features only after the core is solid.

### 2. **SQLite Over JSON Files**

The current prototype uses JSON files in `.bridge/`. This will break with concurrent access and doesn't scale.

**My recommendation:** Use better-sqlite3 from the start. One database file, one schema:

```sql
CREATE TABLE sessions (
  id TEXT PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  role TEXT,
  project TEXT,
  branch TEXT,
  model TEXT,
  status TEXT DEFAULT 'idle',
  pid INTEGER,
  created_at TEXT DEFAULT (datetime('now')),
  last_activity TEXT DEFAULT (datetime('now')),
  metadata TEXT -- JSON blob
);

CREATE TABLE tasks (
  id TEXT PRIMARY KEY,
  session_id TEXT REFERENCES sessions(id),
  title TEXT NOT NULL,
  description TEXT,
  verification_cmd TEXT,
  status TEXT DEFAULT 'pending',
  priority INTEGER DEFAULT 0,
  result TEXT,
  assigned_at TEXT,
  started_at TEXT,
  completed_at TEXT,
  verified_at TEXT
);

CREATE TABLE knowledge (
  key TEXT PRIMARY KEY,
  value TEXT NOT NULL,
  author TEXT,
  tags TEXT, -- JSON array
  created_at TEXT DEFAULT (datetime('now')),
  updated_at TEXT DEFAULT (datetime('now'))
);

CREATE TABLE mailbox (
  id TEXT PRIMARY KEY,
  recipient TEXT NOT NULL,
  sender TEXT NOT NULL,
  type TEXT DEFAULT 'message',
  payload TEXT, -- JSON
  read INTEGER DEFAULT 0,
  created_at TEXT DEFAULT (datetime('now')),
  expires_at TEXT
);

CREATE TABLE events (
  id TEXT PRIMARY KEY,
  type TEXT NOT NULL,
  source TEXT,
  data TEXT, -- JSON
  created_at TEXT DEFAULT (datetime('now'))
);

CREATE INDEX idx_tasks_session ON tasks(session_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_mailbox_recipient ON mailbox(recipient, read);
CREATE INDEX idx_events_type ON events(type, created_at);
```

### 3. **WebSocket for Real-Time Output**

The orchestrator (and human operators) need to see agent output in real-time. Polling `/output` every second is wasteful and laggy.

**My recommendation:** Add WebSocket endpoints:

```
ws://bridge:4001/ws/sessions/:name/output   # Stream agent output
ws://bridge:4001/ws/events                   # Stream all events
ws://bridge:4001/ws/health                   # Health status updates
```

Implementation is simple with the `ws` library:

```typescript
import { WebSocketServer } from "ws";

const wss = new WebSocketServer({ server });

wss.on("connection", (ws, req) => {
  const match = req.url?.match(/\/ws\/sessions\/(.+)\/output/);
  if (match) {
    const sessionId = match[1];
    const session = sessions.get(sessionId);
    if (!session) {
      ws.close(4004, "Session not found");
      return;
    }

    // Stream output to this WebSocket
    const handler = (data: string) => ws.send(data);
    session.on("output", handler);
    ws.on("close", () => session.off("output", handler));
  }
});
```

### 4. **Health Endpoint That Actually Helps**

The current `/health` endpoint just reports session activity. We need more:

```typescript
GET /api/bridge/health

Response:
{
  "status": "healthy",  // healthy | degraded | unhealthy
  "uptime": 3600,
  "version": "1.0.0",
  "capabilities": {
    "tmux": true,
    "git": true,
    "gh": true,
    "node": "24.14.0"
  },
  "resources": {
    "sessions": { "active": 3, "max": 5 },
    "disk": { "used": "2.1GB", "available": "48GB", "percent": 4 },
    "memory": { "used": "1.2GB", "available": "14.8GB", "percent": 8 }
  },
  "sessions": [
    {
      "name": "frontend-dev",
      "status": "working",
      "pid": 12345,
      "uptime": 1800,
      "cpu": 2.3,
      "memory": "512MB",
      "lastOutput": "2s ago"
    }
  ],
  "checks": {
    "database": "ok",
    "diskWritable": "ok",
    "tmuxResponsive": "ok"
  }
}
```

### 5. **Structured Logging**

Replace `console.log` with structured JSON logs:

```typescript
import pino from "pino";

const logger = pino({
  level: process.env.LOG_LEVEL || "info",
  transport:
    process.env.NODE_ENV === "development"
      ? { target: "pino-pretty" }
      : undefined,
});

// Usage:
logger.info({ session: "frontend-dev", event: "created" }, "Session started");
logger.error(
  { session: "frontend-dev", error: err.message, stack: err.stack },
  "Session crashed"
);
```

This makes debugging from log files actually possible.

### 6. **Configuration File**

```yaml
# ~/.pi-fleet/bridge.yaml
bridge:
  port: 4001
  host: "0.0.0.0"
  maxSessions: 5
  sessionTimeout: 1800 # 30 minutes idle timeout

tools:
  pi: "~/.nvm/versions/node/v24.14.0/bin/pi"
  node: "~/.nvm/versions/node/v24.14.0/bin/node"
  # tmux, git, gh auto-discovered via `which`

project:
  root: "~/my-project"
  defaultBranch: "main"
  branchPrefix: "agent/"

verification:
  timeout: 120000 # 2 minutes
  commands:
    typecheck: "pnpm typecheck"
    lint: "pnpm lint"
    test: "pnpm test"
    build: "pnpm build"

logging:
  level: "info"
  file: "~/.pi-fleet/bridge.log"
  maxSize: "10MB"
  maxFiles: 5

storage:
  database: "~/.pi-fleet/fleet.db"
  maxDiskUsage: "10GB"
```

### 7. **Startup Validation**

The bridge should validate its environment and report problems clearly:

```typescript
async function validateEnvironment(): Promise<ValidationResult> {
  const checks = [];

  // Check required tools
  for (const [name, path] of Object.entries(config.tools)) {
    const exists = await fs
      .access(path)
      .then(() => true)
      .catch(() => false);
    checks.push({ tool: name, path, available: exists });
  }

  // Check disk space
  const disk = await checkDiskSpace(config.storage.database);
  checks.push({ check: "disk", available: disk.free > 1_000_000_000 }); // 1GB minimum

  // Check database connectivity
  try {
    db.pragma("journal_mode = WAL");
    checks.push({ check: "database", available: true });
  } catch (e) {
    checks.push({ check: "database", available: false, error: e.message });
  }

  // Check project directory
  const projectExists = await fs
    .access(config.project.root)
    .then(() => true)
    .catch(() => false);
  checks.push({
    check: "project",
    available: projectExists,
    path: config.project.root,
  });

  const failures = checks.filter((c) => !c.available);
  return {
    valid: failures.length === 0,
    checks,
    failures,
  };
}
```

---

## Implementation Priority (Bridge Server)

Since I'm building and running this, here's my recommended order:

### Phase 1: Core (Week 1)

1. **SQLite schema and storage layer** — everything depends on this
2. **Session management with node-pty** — create, list, send, read, kill
3. **Basic HTTP API** — the existing routes from bridge-standalone.js, cleaned up
4. **Health endpoint** — know if things are working
5. **Startup validation** — fail fast with clear errors

### Phase 2: Task System (Week 2)

6. **Task state machine** — pending → assigned → in_progress → done/failed
7. **Verification runner** — run commands, capture results, enforce timeouts
8. **WebSocket output streaming** — real-time agent output

### Phase 3: Integration (Week 3)

9. **Git operations** — branch creation, commit, status (via simple-git)
10. **Mailbox system** — agent-to-agent messaging via SQLite
11. **Knowledge store** — key-value with FTS search

### Phase 4: Polish (Week 4)

12. **Structured logging** — pino with file rotation
13. **Graceful shutdown** — handle SIGTERM/SIGINT properly
14. **Resource monitoring** — disk, memory, CPU tracking
15. **Configuration management** — file + env vars

---

## Risks and Mitigations

| Risk                            | Impact                 | Probability | Mitigation                                             |
| ------------------------------- | ---------------------- | ----------- | ------------------------------------------------------ |
| Pi process hangs (not crashed)  | Agent stops responding | Medium      | Output timeout detection, watchdog timer               |
| SQLite corruption               | All state lost         | Low         | WAL mode, regular backups, integrity checks            |
| Disk exhaustion                 | Bridge fails           | Medium      | Disk monitoring, auto-cleanup of old sessions          |
| Agent produces malicious output | Parsing breaks         | Low         | Don't parse raw output; use structured result protocol |
| Port conflicts                  | Bridge won't start     | Low         | Configurable port, startup validation                  |
| tmux version differences        | Inconsistent behavior  | Medium      | Version check on startup, or drop tmux entirely        |

---

## Questions for the Team

1. **tmux vs node-pty:** I strongly prefer node-pty. Anyone have strong objections? The PRD mentions tmux but I think we can do better.

2. **SQLite vs JSON files:** SQLite adds a dependency but gives us ACID transactions and queries. JSON files are simpler but will break with concurrent access. Vote?

3. **WebSocket from day one:** Or start HTTP-only and add WebSocket later? I vote day one — it's not much harder and the UX improvement is massive.

4. **Single file vs modules:** For v1 bridge, I want to keep it in one file (~1000 lines) for simplicity. Split into modules when it gets painful. Objections?

5. **pi-fleet.yaml location:** Should this live in the project root (like PRD suggests) or in `~/.pi-fleet/` (user-level)? I think we need both — user-level for bridge config, project-level for project config.

---

## Conclusion

The PRD is a solid foundation. The vision is right. But we're underestimating the operational complexity of running agents in production on shared hardware.

**My top 3 priorities as bridge server owner:**

1. **Reliability over features** — I'd rather have 3 rock-solid features than 10 fragile ones
2. **Observability from day one** — If I can't see what's happening, I can't fix it
3. **Simple state management** — SQLite for everything, one file, easy to backup and debug

Let's build something that works reliably first, then make it feature-rich.

---

_Next step: Prototype the SQLite schema and session manager. Get two agents running on one machine. Verify we can send tasks and get results back. Everything else is details._
