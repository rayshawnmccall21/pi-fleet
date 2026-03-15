# Pi Fleet — Multi-Device Agent Orchestration Platform

## Product Requirements Document v1.0

---

## Vision

Pi Fleet is an open-source infrastructure that enables teams of AI agents (Pi agents) running across multiple physical devices to collaborate, coordinate, and build software together — controlled by an orchestrator agent on a separate machine. It is device-agnostic, company-agnostic, and designed for any team that wants to run autonomous AI development workflows across distributed hardware.

## Problem Statement

Today, AI coding agents run in isolation on a single machine. When you have multiple devices (a Mac Mini, a MacBook Pro, cloud VMs), there's no standard way to:

1. Have an agent on Device A orchestrate agents on Device B
2. Share context, knowledge, and discoveries across agent sessions
3. Track tasks, verify work, and manage git workflows across devices
4. Maintain persistent agent presence that survives session disconnects
5. Enable agent-to-agent communication without human intermediation

## Target Users

- **Solo developers** with multiple machines who want AI agents working in parallel
- **Teams** who want to deploy AI agents across development hardware
- **Companies** building AI-first development workflows
- **Open-source projects** wanting distributed AI contributions

## Core Principles

1. **Pi-native**: Built for Pi agents ([@mariozechner/pi-coding-agent](https://github.com/badlogic/pi-mono)), extensible to other agent frameworks
2. **Device-agnostic**: Works on any Unix-like system with SSH access
3. **Company-agnostic**: No hardcoded project references — fully configurable
4. **TDD-first**: Every module has tests before implementation
5. **Offline-resilient**: Agents can work independently and sync when connected
6. **Observable**: Full visibility into what every agent is doing at all times

---

## Architecture Overview

### System Topology

```
┌─────────────────────────────────────────────────────┐
│                  CONTROL PLANE                       │
│              (Orchestrator Device)                    │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Fleet Manager│  │ Task Planner │  │ Knowledge  │ │
│  │ (spawn/kill/ │  │ (break work  │  │ Graph      │ │
│  │  monitor)    │  │  into tasks) │  │ (shared    │ │
│  │              │  │              │  │  context)  │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │        │
│  ┌──────┴─────────────────┴─────────────────┴──────┐ │
│  │              Fleet Protocol (HTTP/WS)            │ │
│  └──────────────────────┬───────────────────────────┘ │
└─────────────────────────┼───────────────────────────┘
                          │ SSH + HTTP
                          │
┌─────────────────────────┼───────────────────────────┐
│                  WORKER PLANE                        │
│              (Remote Device(s))                      │
│                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌────────────┐ │
│  │ Bridge Server│  │ Agent Pool   │  │ Git Manager│ │
│  │ (API gateway │  │ (tmux-backed │  │ (branches, │ │
│  │  for agents) │  │  Pi sessions)│  │  commits,  │ │
│  │              │  │              │  │  PRs)      │ │
│  └──────┬───────┘  └──────┬───────┘  └─────┬──────┘ │
│         │                 │                 │        │
│  ┌──────┴─────────────────┴─────────────────┴──────┐ │
│  │           Verification Engine                    │ │
│  │    (lint, typecheck, test, build gates)          │ │
│  └──────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────┘
```

### Component Architecture

#### 1. Fleet Protocol (`@pi-fleet/protocol`)

The communication layer between control plane and worker planes.

```
fleet-protocol/
├── src/
│   ├── messages/           # Message type definitions
│   │   ├── types.ts        # Core message types (TaskAssign, TaskResult, Heartbeat, etc.)
│   │   ├── envelope.ts     # Message envelope (from, to, timestamp, correlationId)
│   │   └── validators.ts   # Zod schemas for all message types
│   ├── transport/          # Transport implementations
│   │   ├── http.ts         # HTTP REST transport
│   │   ├── websocket.ts    # WebSocket transport for real-time
│   │   └── ssh.ts          # SSH tunnel transport
│   ├── discovery/          # Device discovery
│   │   ├── mdns.ts         # mDNS/Bonjour for LAN discovery
│   │   ├── manual.ts       # Manual device registration
│   │   └── registry.ts     # Device registry
│   └── auth/               # Authentication
│       ├── token.ts        # JWT token management
│       └── keypair.ts      # SSH key-based auth
├── __tests__/
│   ├── messages.test.ts
│   ├── transport.test.ts
│   └── discovery.test.ts
└── package.json
```

#### 2. Bridge Server (`@pi-fleet/bridge`)

Runs on each worker device. Manages agent sessions and provides the API.

```
bridge/
├── src/
│   ├── server.ts           # Express server setup
│   ├── sessions/           # tmux-backed Pi agent sessions
│   │   ├── manager.ts      # Create/destroy/list sessions
│   │   ├── communicator.ts # Send messages, read output
│   │   ├── health.ts       # Health monitoring, auto-restart
│   │   └── types.ts        # Session types
│   ├── tasks/              # Task state machine
│   │   ├── tracker.ts      # Task lifecycle management
│   │   ├── queue.ts        # Priority queue for task assignment
│   │   ├── states.ts       # State definitions and transitions
│   │   └── types.ts
│   ├── verification/       # Code quality gates
│   │   ├── runner.ts       # Run verification commands
│   │   ├── presets.ts      # Common presets (lint, test, typecheck)
│   │   ├── reporter.ts     # Verification result formatting
│   │   └── types.ts
│   ├── git/                # Git operations
│   │   ├── branch.ts       # Branch management
│   │   ├── commit.ts       # Commit operations
│   │   ├── worktree.ts     # Git worktree support
│   │   └── pr.ts           # GitHub PR creation (via gh cli)
│   ├── knowledge/          # Shared knowledge store
│   │   ├── store.ts        # Key-value store with tags
│   │   ├── search.ts       # Full-text search over knowledge
│   │   └── types.ts
│   ├── mailbox/            # Agent-to-agent messaging
│   │   ├── queue.ts        # Per-agent message queue
│   │   ├── router.ts       # Message routing
│   │   └── types.ts
│   └── results/            # Structured result collection
│       ├── collector.ts    # Result aggregation
│       └── types.ts
├── __tests__/
│   ├── sessions/
│   ├── tasks/
│   ├── verification/
│   ├── git/
│   ├── knowledge/
│   └── mailbox/
└── package.json
```

#### 3. Fleet Manager (`@pi-fleet/manager`)

Runs on the control plane. Orchestrates everything.

```
manager/
├── src/
│   ├── orchestrator.ts     # Main orchestration loop
│   ├── devices/            # Remote device management
│   │   ├── connector.ts    # Connect to remote bridges
│   │   ├── monitor.ts      # Device health monitoring
│   │   └── types.ts
│   ├── planning/           # Task planning
│   │   ├── planner.ts      # Break epics into tasks
│   │   ├── assigner.ts     # Assign tasks to agents
│   │   ├── dependencies.ts # Task dependency resolution
│   │   └── types.ts
│   ├── teams/              # Agent team management
│   │   ├── composer.ts     # Compose teams for projects
│   │   ├── roles.ts        # Role definitions (dev, reviewer, tester)
│   │   └── types.ts
│   ├── review/             # Code review automation
│   │   ├── reviewer.ts     # Automated code review
│   │   ├── merge.ts        # Merge decision engine
│   │   └── types.ts
│   └── reporting/          # Progress reporting
│       ├── dashboard.ts    # Terminal dashboard
│       ├── metrics.ts      # Performance metrics
│       └── types.ts
├── __tests__/
│   ├── orchestrator.test.ts
│   ├── planning/
│   ├── teams/
│   └── review/
└── package.json
```

#### 4. CLI (`@pi-fleet/cli`)

Command-line interface for human operators.

```
cli/
├── src/
│   ├── index.ts            # CLI entry point
│   ├── commands/
│   │   ├── init.ts         # Initialize pi-fleet in a project
│   │   ├── device.ts       # Add/remove/list devices
│   │   ├── team.ts         # Spawn/manage agent teams
│   │   ├── task.ts         # Create/assign/track tasks
│   │   ├── status.ts       # Dashboard view
│   │   ├── knowledge.ts    # Manage shared knowledge
│   │   └── config.ts       # Configuration management
│   └── ui/
│       ├── dashboard.ts    # Rich terminal UI
│       └── table.ts        # Table formatting
├── __tests__/
└── package.json
```

#### 5. Configuration (`@pi-fleet/config`)

Project-agnostic configuration system.

```
config/
├── src/
│   ├── schema.ts           # Configuration schema (Zod)
│   ├── loader.ts           # Load from pi-fleet.yaml
│   ├── defaults.ts         # Sensible defaults
│   └── types.ts
└── package.json
```

### Configuration File (`pi-fleet.yaml`)

```yaml
# pi-fleet.yaml - placed in any project root
version: "1.0"

# Devices in the fleet
devices:
  - name: orchestrator
    host: mac-414.local
    role: control-plane

  - name: worker-1
    host: autumnrays-mbp.local
    role: worker
    ssh_user: autumnraymccall
    bridge_port: 4001

# Project configuration
project:
  name: my-project
  root: ~/my-project

  # Verification gates
  verify:
    typecheck: "pnpm typecheck"
    lint: "pnpm lint"
    test: "pnpm test"
    build: "pnpm build"

  # Git workflow
  git:
    default_branch: main
    branch_prefix: "agent/"
    auto_commit: true
    auto_pr: false

# Agent team templates
teams:
  full-stack:
    roles:
      - name: frontend-dev
        count: 1
        model: claude-sonnet-4-20250514
      - name: backend-dev
        count: 1
        model: claude-sonnet-4-20250514
      - name: code-reviewer
        count: 1
        model: claude-sonnet-4-20250514

  rapid:
    roles:
      - name: developer
        count: 3
        model: claude-sonnet-4-20250514

# Knowledge base seed
knowledge:
  - key: project-stack
    value: "Describe your tech stack here"
    tags: [architecture]
```

---

## Feature Specifications

### F1: Device Discovery & Connection

**As** an orchestrator agent,  
**I want** to discover and connect to worker devices on the network,  
**So that** I can deploy agent teams to available hardware.

**Acceptance Criteria:**

- mDNS discovery finds devices running pi-fleet bridge on LAN
- Manual device registration via config file
- SSH key-based authentication between devices
- Connection health monitoring with auto-reconnect
- Device capability reporting (CPU, memory, running agents)

### F2: Agent Session Management

**As** an orchestrator,  
**I want** to spawn, monitor, and kill Pi agent sessions on remote devices,  
**So that** I can manage a pool of workers.

**Acceptance Criteria:**

- Spawn Pi agents in tmux sessions with configurable model, prompt, and working directory
- List all active sessions with metadata (role, project, idle time)
- Send messages to agents and read their terminal output
- Auto-restart crashed sessions with same task context
- Graceful shutdown with cleanup
- Session persistence across bridge restarts

### F3: Task State Machine

**As** an orchestrator,  
**I want** structured task tracking with state transitions,  
**So that** I can manage work across multiple agents.

**Acceptance Criteria:**

- States: `pending` → `assigned` → `in_progress` → `review` → `verified` → `done` (or `failed`)
- Task assignment with title, description, verification command, and branch
- Priority levels: critical, high, normal, low
- Dependency tracking between tasks
- Task dashboard with summary statistics
- History and audit trail

### F4: Verification Engine

**As** an orchestrator,  
**I want** automated code verification after agents complete tasks,  
**So that** I don't rely on agent self-reporting.

**Acceptance Criteria:**

- Run arbitrary shell commands as verification (lint, test, typecheck, build)
- Configurable verification presets per project
- Timeout handling for long-running verifications
- Structured pass/fail results with output capture
- Gate progression: tasks can't move to `verified` without passing verification
- Parallel verification support

### F5: Git Workflow Manager

**As** an orchestrator,  
**I want** automated git branch and commit management,  
**So that** multiple agents can work in parallel without conflicts.

**Acceptance Criteria:**

- Auto-create feature branches per task or per agent
- Git worktree support for true parallel branch work
- Commit with structured messages
- Branch status tracking
- GitHub PR creation via `gh` CLI
- Merge conflict detection
- Branch cleanup after merge

### F6: Shared Knowledge Store

**As** an agent working on a project,  
**I want** to share discoveries with other agents,  
**So that** the team accumulates collective intelligence.

**Acceptance Criteria:**

- Key-value store with tags and full-text search
- Any agent can write knowledge entries
- Knowledge persists across sessions
- Tag-based filtering
- Author tracking
- Seed knowledge from project config
- Knowledge export/import

### F7: Agent-to-Agent Messaging

**As** an agent,  
**I want** to send structured messages to other agents,  
**So that** agents can coordinate without human intermediation.

**Acceptance Criteria:**

- Per-agent mailbox (queue)
- Message types: task, query, response, notification, broadcast
- Message routing by session name
- Read receipts
- Message TTL (auto-expire)
- Cross-device messaging (agent on device A messages agent on device B)

### F8: Fleet CLI

**As** a human operator,  
**I want** a command-line interface to manage the fleet,  
**So that** I can configure, monitor, and control everything.

**Acceptance Criteria:**

- `pi-fleet init` — Initialize in a project
- `pi-fleet device add/remove/list` — Manage devices
- `pi-fleet team spawn/status/kill` — Manage agent teams
- `pi-fleet task create/assign/list/verify` — Task management
- `pi-fleet status` — Rich dashboard view
- `pi-fleet knowledge set/get/search` — Knowledge management
- `pi-fleet config` — Configuration management
- Tab completion and help text

### F9: Observability & Reporting

**As** an orchestrator or human operator,  
**I want** full visibility into fleet operations,  
**So that** I can understand what's happening and debug issues.

**Acceptance Criteria:**

- Real-time terminal dashboard showing all agents and their status
- Per-agent output streaming
- Task progress tracking with ETA
- Cost tracking (parse Pi cost display)
- Performance metrics (task duration, verification pass rate)
- Event log for audit trail
- Exportable reports (JSON, Markdown)

### F10: Project Configuration

**As** a developer adopting pi-fleet,  
**I want** simple configuration that works with any project,  
**So that** I can get started quickly without StylePass-specific setup.

**Acceptance Criteria:**

- Single `pi-fleet.yaml` config file
- Sensible defaults for common setups
- Environment variable overrides
- Config validation with helpful error messages
- Example configs for common stacks (Next.js, Expo, Python, Rust)
- Config migration between versions

---

## Technical Specifications

### Tech Stack

| Component          | Technology                                      |
| ------------------ | ----------------------------------------------- |
| Runtime            | Node.js 20+ (ESM)                               |
| Language           | TypeScript 5.x (strict mode)                    |
| Package Manager    | pnpm with workspaces                            |
| Build              | tsup or tsc                                     |
| Test Framework     | Vitest                                          |
| HTTP Server        | Express 4.x (not 5 — too many breaking changes) |
| WebSocket          | ws                                              |
| Schema Validation  | Zod                                             |
| CLI Framework      | Commander.js                                    |
| Terminal UI        | Ink or blessed                                  |
| Git Operations     | simple-git                                      |
| Process Management | tmux via child_process                          |
| Discovery          | mdns / bonjour-service                          |
| CI/CD              | GitHub Actions                                  |

### Monorepo Structure

```
pi-fleet/
├── packages/
│   ├── protocol/       # @pi-fleet/protocol
│   ├── bridge/         # @pi-fleet/bridge
│   ├── manager/        # @pi-fleet/manager
│   ├── cli/            # @pi-fleet/cli
│   └── config/         # @pi-fleet/config
├── apps/
│   └── dashboard/      # Web dashboard (future)
├── docs/
│   ├── getting-started.md
│   ├── architecture.md
│   ├── api-reference.md
│   └── contributing.md
├── examples/
│   ├── nextjs/
│   ├── expo/
│   └── python/
├── scripts/
│   ├── setup.sh
│   └── dev.sh
├── pi-fleet.yaml       # Self-hosting config
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.json
├── vitest.config.ts
└── README.md
```

### API Contracts

All Bridge API endpoints follow this response format:

```typescript
interface ApiResponse<T> {
  success: boolean;
  data?: T;
  error?: {
    code: string;
    message: string;
    details?: unknown;
  };
  timestamp: string;
  requestId: string;
}
```

### Testing Strategy

| Level       | Tool               | Coverage Target                      |
| ----------- | ------------------ | ------------------------------------ |
| Unit        | Vitest             | 80%+ per module                      |
| Integration | Vitest + supertest | All API endpoints                    |
| E2E         | Custom harness     | Spawn → Task → Verify → Result cycle |
| Contract    | Zod schemas        | 100% message types                   |

### Security

- SSH key-based authentication between devices
- JWT tokens for bridge API access
- No secrets in config files (env vars or encrypted store)
- Rate limiting on all endpoints
- Input validation on all parameters
- Audit logging for all operations

---

## Implementation Phases

### Phase 1: Foundation (Tasks 1-50)

- Monorepo setup with pnpm workspaces
- TypeScript configuration
- Vitest setup
- Core message types and validators
- Configuration schema and loader
- Basic HTTP transport

### Phase 2: Bridge Server (Tasks 51-100)

- Express server with middleware
- Session manager (tmux integration)
- Session communicator (send/read)
- Health monitoring
- Task state machine
- Verification runner

### Phase 3: Fleet Protocol (Tasks 101-130)

- WebSocket transport
- Device discovery (mDNS + manual)
- Authentication (JWT + SSH keys)
- Cross-device message routing
- Connection health monitoring

### Phase 4: Fleet Manager (Tasks 131-170)

- Orchestrator loop
- Task planner and assigner
- Team composer
- Code review automation
- Git workflow manager
- Knowledge store with search

### Phase 5: CLI & Polish (Tasks 171-200+)

- CLI commands
- Terminal dashboard
- Documentation
- Example configs
- GitHub Actions CI
- npm publishing setup
- README and contributing guide

---

## Success Metrics

| Metric                   | Target          |
| ------------------------ | --------------- |
| Agent spawn time         | < 5 seconds     |
| Message delivery latency | < 500ms         |
| Task verification time   | < 2 minutes     |
| Bridge API response time | < 200ms         |
| Session crash recovery   | < 10 seconds    |
| Knowledge query time     | < 100ms         |
| Zero-config startup      | Under 2 minutes |
| Test coverage            | 80%+            |

---

## Non-Goals (v1)

- Web dashboard (future v2)
- Cloud VM support (future — focus on LAN first)
- Non-Pi agent support (future — plugin architecture)
- Mobile device orchestration (future)
- Multi-tenancy (future)
- Paid features / SaaS (this is open-source)

---

## Open Questions

1. Should we use git worktrees or separate clones for branch isolation?
2. WebSocket vs SSE for real-time updates from bridge?
3. How to handle agent model costs — budget limits per team?
4. Should knowledge store be SQLite or file-based JSON?
5. How to handle timezone differences for cron-scheduled tasks?

---

## References

- [Pi Coding Agent](https://github.com/badlogic/pi-mono)
- [Model Context Protocol](https://modelcontextprotocol.io/)
- [tmux Manual](https://man.openbsd.org/tmux)
- Existing prototype: `bridge-standalone.js` at `~/remote-mcp-server/`
