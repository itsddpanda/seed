# 📑 Project Spec: OpenCode Projects (OCP) Control Tower

**Version:** 0.9 (GSD Unification + Concurrency Correctness + Schema Completeness)

**Core Logic:** Autoresearch-inspired Iterative Engineering via Remote or Local OpenCode Server, powered by GSD as the underlying context engineering and planning engine.

**Changelog from v0.8:**
- Unified `.ocp/` and `.planning/` into a single shared brain (Section 4 rewrite)
- Defined Action Plan schema explicitly (Section 5.4)
- Fixed FILE_MAP.md staleness in concurrent execution via branch-scoped snapshots (Section 5.1)
- Placed `@test` agent explicitly in the Ratchet loop (Section 6)
- Defined LLM Judge context budget (Section 5.6)
- Added SYSTEM_STATE.md / STATE.md compression trigger (Section 5.7, new)
- Wired `tokens_used` + `token_budget` as live enforced fields (Section 4)
- Defined GSD ↔ OCP layer boundary (Section 2)

---

## 1. System Overview

OCP is a Jira-style project management graphical interface that orchestrates OpenCode agents to execute an entire software development lifecycle autonomously.

Instead of a chat interface, users manage an "Autonomous Engineering Org" through Kanban boards and Epic planning. The UI is decoupled from the execution engine, allowing heavy AI tasks and code execution to run on a remote server while the user controls it from a lightweight local client, or run entirely locally.

OCP does **not** maintain its own planning data store. It fires GSD workflows and renders GSD's `.planning/` directory as a Kanban board. GSD is the brain; OCP is the execution harness and UI on top of it.

---

## 2. The Architecture & Connection Plumbing

| **Component** | **Technology** | **Role / Rationale** |
|---|---|---|
| **Frontend UI** | Tauri (Rust/React) | Native desktop Kanban board and live streaming terminal feeds. Acts as the Orchestrator with full system privileges. Renders `.planning/` as Kanban state. |
| **Planning Engine** | GSD (get-shit-done) | Context engineering, phase planning, subagent orchestration, state management, and milestone lifecycle. OCP fires GSD commands; GSD writes all planning artifacts. |
| **Execution Engine** | OpenCode + Docker | Runs inside a standardized Workspace Container (DevContainer). Acts as the Worker, executing prompts and editing files within strict boundaries. |
| **Transport Layer** | SSH Tunnel / IPC | Secures the connection. For remote, local UI connects to localhost:4096, securely port-forwarded. For local, direct IPC/localhost. |
| **Data Storage** | Markdown + YAML | All planning state lives in `.planning/` (managed by GSD). Execution metadata lives in `.ocp/`. Git is the ultimate state-tracker. |

### GSD ↔ OCP Layer Boundary

OCP and GSD map to each other directly. There is no duplication:

| OCP concept | GSD equivalent | Notes |
|---|---|---|
| Epic | Milestone (`ROADMAP.md`) | `/gsd:new-milestone` creates an Epic |
| Feature | Phase | `/gsd:plan-phase N` plans a Feature |
| Ticket | `PLAN.md` task | GSD's XML task structure is the Action Plan |
| `SYSTEM_STATE.md` | `STATE.md` | Same file — OCP reads GSD's STATE.md |
| `.ocp/` planning data | `.planning/` | OCP drops its own planning dirs entirely |

OCP **fires** GSD commands. GSD **writes** artifacts. OCP **renders** them.

```
OCP Kanban UI (Tauri)
  └── fires: /gsd:plan-phase, /gsd:execute-phase, /gsd:complete-milestone
        └── GSD writes: .planning/ (STATE.md, PLAN.md, SUMMARY.md, ROADMAP.md)
              └── OCP reads: .planning/ → renders as Kanban board
```

---

## 3. Configuration & Environment Switching

The application maintains a **Global Configuration Registry** on the user's local machine stored in `.ocp/config.json`.

- **Load Existing Configuration:** Users select from saved project profiles to auto-populate UI settings.
- **Enter New Configuration:** The UI requires:
  - **Always:** Environment Type (Local/Remote), Local Project Path.
  - **Remote Only:** Remote Host Address, Port Number, SSH Key Path.

**`.ocp/config.json` schema:**

```json
{
  "environment": "remote",
  "local_project_path": "/home/user/my-project",
  "remote_host": "dev.example.com",
  "remote_port": 4096,
  "ssh_key_path": "~/.ssh/id_ed25519",
  "judge_context_budget": 2000,
  "default_token_budget": 50000,
  "model_profile": "balanced"
}
```

---

## 4. Data Storage: Unified File-Based Brain

All planning state lives in `.planning/` (owned and written by GSD). OCP's `.ocp/` directory holds only execution metadata that GSD does not produce.

**Unified Directory Structure:**

```
my-project/
├── .planning/                   ← GSD brain (OCP reads, GSD writes)
│   ├── STATE.md                 ← replaces SYSTEM_STATE.md
│   ├── ROADMAP.md               ← replaces epics/
│   ├── REQUIREMENTS.md
│   ├── 1-CONTEXT.md
│   ├── 1-1-PLAN.md              ← replaces tickets/ (= Action Plan)
│   ├── 1-1-SUMMARY.md
│   ├── archive/                 ← compressed milestone state (auto-generated)
│   └── research/
│
├── .ocp/                        ← OCP execution metadata only
│   ├── locks/                   ← .lock files + heartbeats
│   ├── FILE_MAP.md              ← codebase index (OCP-specific)
│   ├── snapshots/               ← branch-scoped FILE_MAP snapshots
│   ├── docker/                  ← container configs
│   └── config.json              ← connection profiles + budgets
│
├── src/
└── package.json
```

### Ticket / Action Plan Schema

Each ticket is a GSD `PLAN.md` file with extended OCP frontmatter. The Action Plan body is GSD's XML task structure, extended with OCP execution fields:

```yaml
---
# GSD fields
plan_id: 1-1-PLAN
phase: 1
status: To Do           # To Do | In Progress | In Review | Done | Blocked

# OCP execution fields
ticket_id: TKT-102
priority: High
assignee: "@build"
reviewer: "rokicool"
parent_feature: FEAT-1
depends_on:
  - TKT-101
retry_count: 0
max_retries: 3
token_budget: 50000     # enforced ceiling for this ticket's total agent spend
tokens_used: 0          # updated live after each agent call via yq patch
branch: ocp/TKT-102
---
```

**Action Plan body (GSD XML task structure):**

```xml
<task type="auto">
  <n>Create Next.js API route for user login</n>
  <objective>Single sentence. What done looks like.</objective>
  <context_budget>8000</context_budget>  <!-- max tokens @build reads from this plan -->
  <files>
    <file action="CREATE">src/app/api/auth/login/route.ts</file>
    <file action="MODIFY" lines="45-67">src/lib/auth.ts</file>
    <file action="READ_ONLY">tests/auth.test.ts</file>  <!-- written by @test, @build cannot edit -->
  </files>
  <snippets>
    <!-- @plan inlines only the relevant code sections, not full files -->
    <!-- src/lib/auth.ts lines 45-67 snippet here -->
  </snippets>
  <constraints>
    Use jose for JWT, not jsonwebtoken (CommonJS incompatibility).
    Do not modify any file outside the files list above.
  </constraints>
  <verify>npm run test:auth</verify>
  <done>Valid credentials return cookie. Invalid credentials return 401.</done>
</task>
```

**`tokens_used` is a live field.** After each agent call, the Ratchet loop patches the frontmatter:

```bash
NEW_TOKENS=1250  # returned by OpenCode API response
yq e ".tokens_used += $NEW_TOKENS" -i .planning/1-1-PLAN.md
```

If `tokens_used` approaches `token_budget`, the Ratchet loop surfaces a warning before the next retry. Reaching `max_retries` still halts execution, but token budget provides an earlier soft signal.

---

## 5. System Integrity & Concurrency Protocols

### 5.1. Handling Concurrency (Git Branching, Snapshots & Cycle Detection)

- **Execution:** When a ticket moves to In Progress, the system runs `git checkout -b ocp/[ticket-id]`.
- **FILE_MAP.md Snapshot (Concurrency Fix):** At branch creation time, the system immediately snapshots the current FILE_MAP.md scoped to this branch:

```bash
cp .ocp/FILE_MAP.md .ocp/snapshots/FILE_MAP.ocp-TKT-102.snapshot.md
```

`@plan` reads the **branch snapshot**, not the global FILE_MAP.md. This prevents the race condition where a parallel ticket merges and regenerates FILE_MAP.md while another ticket's `@plan` is mid-execution. The snapshot is deleted on merge. The global FILE_MAP.md is only regenerated on merge to main.

- **DAG Cycle Detection:** The Tauri backend evaluates `depends_on` arrays using a Directed Acyclic Graph (DAG) algorithm. If a circular dependency is detected, affected tickets are instantly marked `Blocked (Cycle Error)`.

### 5.2. Privilege Separation (Rogue Lobotomy Protection)

Strict OS-level privilege separation ensures the agent cannot delete or corrupt the planning brain:

- **The Orchestrator:** The Tauri UI owns `.planning/` and `.ocp/` and manages all state and git commands.
- **The Worker (Docker):** The OpenCode agent runs inside a container with explicit mount permissions:

```bash
-v $(pwd)/src:/workspace/src:rw
-v $(pwd)/.planning:/workspace/.planning:ro
-v $(pwd)/.ocp:/workspace/.ocp:ro
```

The agent has strictly **Read-Only** access to all state files. It can only write to `src/`.

### 5.3. Mandatory Ticket Locking & Heartbeats

When a ticket is In Progress, the UI applies a Read-Only lock to the plan body.

- **Lock file location:** `.ocp/locks/TKT-102.lock`
- **Heartbeat Mechanism:** The `.lock` file contains a timestamp updated every 60 seconds by the Tauri process.
- **Zombie Lock Recovery:** If the UI detects a `.lock` file older than 5 minutes with no heartbeat update, it surfaces a "Force Unlock & Reset" button to recover the ticket.

**Lock file schema:**

```json
{
  "ticket_id": "TKT-102",
  "branch": "ocp/TKT-102",
  "started_at": "2026-03-27T10:00:00Z",
  "heartbeat": "2026-03-27T10:04:32Z",
  "retry_count": 1
}
```

### 5.4. Hierarchical Context Funnels (Token Efficiency)

Context is strictly separated by agent role. No agent receives more than it needs.

| Agent | Reads | Does not read |
|---|---|---|
| `@plan` | FILE_MAP.md snapshot, full files for relevant scope | Unrelated source files |
| `@test` | Requirements section of ticket, `<verify>` command | Codebase, FILE_MAP.md |
| `@build` | Action Plan XML + inlined snippets only | Full files, FILE_MAP.md, test files |
| `@judge` | Error log (last 100 lines), failing test file, `<objective>` + `<verify>` fields | Everything else |

**`@plan` produces the Action Plan with a `context_budget` field** (max tokens `@build` may consume reading it). If the Action Plan exceeds budget, `@plan` must compress snippets before handing off. This prevents token bloat from leaking through the funnel.

### 5.5. Adversarial QA (Validation Integrity)

To prevent agents from writing trivial hallucinated tests to escape the Ratchet loop:

- **`@test`** reads the requirements and writes the validation script **before `@build` starts**.
- **`@build`** is physically blocked from editing the test file via Docker mount and the `READ_ONLY` file action in the Action Plan.
- **Dispute Mechanic:** If `@build` determines the test is mathematically impossible or flawed, it halts with `Status: Blocked (Bad Test)` for human review. The error log is preserved in the ticket file.

### 5.6. The Fail-Fast Pre-Flight & LLM Judge (Flake Checks)

Validation runs in two gates to prevent expensive Ratchet resets on flaky tests.

1. **Gate 1 (Lightning Check):** Fast linters and compilers (e.g. `tsc --noEmit`, `eslint`) run first. Catch 90% of syntax/import errors in seconds.
2. **Gate 2 (The LLM Judge):** If a heavy E2E test fails, a cheap/fast model reviews the failure.

**Judge context (hard limit — nothing else is passed):**

```
- Error log output: last 100 lines only
- Failing test file: full content
- From Action Plan: <objective> and <verify> fields only
- System prompt: "Classify as CODE_ERROR or FLAKE. Respond with one word only."
```

**Judge routing:**
- `CODE_ERROR` → `git reset --hard` → increment `retry_count` → retry from `@build`
- `FLAKE` → re-run Gate 2 only, no reset, no retry increment

**Judge model:** Use the budget model (Haiku or equivalent). This is a classification call, not a reasoning call. Configured via `judge_context_budget: 2000` in `.ocp/config.json`.

### 5.7. STATE.md Compression (New)

`STATE.md` accumulates decisions, blockers, and resolved context over the life of a project. Without compression, it becomes the largest token cost in the system.

**Compression is triggered automatically when OCP calls `/gsd:complete-milestone`:**

1. A compression agent reads full `STATE.md`
2. Resolved decisions and closed blockers are archived to `.planning/archive/milestone-N-state.md`
3. `STATE.md` is rewritten to contain only: open decisions, active blockers, current phase position, and a 3-line milestone summary

This fires naturally at the end of every Epic, keeping `STATE.md` lean for the next milestone. No manual intervention required.

---

## 6. The "Ratchet" Execution Loop

The complete execution sequence with all agents explicitly placed:

```
1.  TRIGGER
    └── UI sets ticket status → In Progress

2.  SETUP
    └── git checkout -b ocp/[ticket-id]
    └── Snapshot FILE_MAP.md → .ocp/snapshots/FILE_MAP.[ticket-id].snapshot.md
    └── Initialize Docker DevContainer mounts (src:rw, .planning:ro, .ocp:ro)
    └── Create .ocp/locks/[ticket-id].lock with initial timestamp

3.  PLAN
    └── @plan reads FILE_MAP branch snapshot
    └── @plan generates Action Plan (PLAN.md XML structure) with context_budget
    └── If Action Plan exceeds context_budget → @plan compresses snippets

4.  TEST AUTHORING (before @build starts)
    └── @test reads ticket Requirements
    └── @test writes validation script
    └── @build is BLOCKED from this file (Docker mount + READ_ONLY file action)

5.  BUILD
    └── @build reads Action Plan + inlined snippets only
    └── @build implements against the already-written test

6.  SYNC
    └── git pull --rebase origin main
    └── Conflict markers detected → auto-fail → back to step 5

7.  GATE 1 — LIGHTNING CHECK
    └── Run fast linter / compiler (tsc --noEmit, eslint)
    └── FAIL → increment retry_count, git reset --hard → back to step 5
    └── PASS → proceed to Gate 2

8.  GATE 2 — HEAVY VALIDATION
    └── @test validation script runs
    └── PASS → proceed to step 9
    └── FAIL → LLM Judge evaluates:
        ├── CODE_ERROR → git reset --hard, increment retry_count → back to step 5
        └── FLAKE → re-run Gate 2, no reset, no retry increment

    └── retry_count == max_retries:
        ├── Preserve last error log in ticket file
        ├── Status → Blocked
        └── HALT

9.  PASS — COMMIT & MERGE
    └── Commit code with message: feat([ticket-id]): [objective]
    └── Merge branch → main
    └── Regenerate global .ocp/FILE_MAP.md
    └── Delete branch snapshot: .ocp/snapshots/FILE_MAP.[ticket-id].snapshot.md
    └── Delete .ocp/locks/[ticket-id].lock
    └── GSD writes SUMMARY.md for this plan
    └── GSD updates STATE.md with outcome
    └── Ticket status → In Review (if reviewer set) or Done

10. MILESTONE COMPLETE (when all tickets in Epic are Done)
    └── OCP fires /gsd:complete-milestone
    └── STATE.md compression triggered (Section 5.7)
    └── Git tag created for milestone
    └── Epic status → Archived
```

### Token Tracking During Loop

After every agent call (steps 3, 4, 5, 8), the loop patches the ticket frontmatter:

```bash
yq e ".tokens_used += $NEW_TOKENS" -i .planning/[plan-id].md
```

If `tokens_used` approaches `token_budget` before `max_retries` is reached, the UI surfaces a warning: "Token budget at 80% — consider human review before next retry."

---

## 7. Architecture Diagram

```
┌───────────────────────────────────────────────────────┐
│                OCP Kanban UI (Tauri)                  │
│  reads .planning/ → renders as Kanban board           │
│  fires GSD commands on ticket state changes           │
└──────────────────────┬────────────────────────────────┘
                       │ /gsd:plan-phase
                       │ /gsd:execute-phase
                       │ /gsd:complete-milestone
┌──────────────────────▼────────────────────────────────┐
│              GSD Workflow Engine                      │
│  @plan → PLAN.md (= Action Plan)                      │
│  @build subagents → fresh 200k context each           │
│  /gsd:complete-milestone → STATE.md compression       │
└──────────────────────┬────────────────────────────────┘
                       │ writes artifacts
┌──────────────────────▼────────────────────────────────┐
│           .planning/  (shared brain, GSD-owned)       │
│  STATE.md, ROADMAP.md, PLAN.md, SUMMARY.md            │
│  REQUIREMENTS.md, archive/, research/                 │
└───────────────────────────────────────────────────────┘
┌───────────────────────────────────────────────────────┐
│           .ocp/  (execution metadata, OCP-owned)      │
│  locks/, FILE_MAP.md, snapshots/, docker/, config.json│
└───────────────────────────────────────────────────────┘
```

---

## 8. Agent Model Summary

| Agent | Maps to GSD | Context it receives | Cannot access |
|---|---|---|---|
| `@plan` | `/gsd:plan-phase` | FILE_MAP snapshot, relevant source files | Unrelated files |
| `@test` | GSD verifier agent | Ticket requirements, `<verify>` field | Codebase, FILE_MAP |
| `@build` | `/gsd:execute-phase` subagent | Action Plan XML + snippets only | Full files, test files, FILE_MAP |
| `@judge` | GSD LLM Judge | Error log (100 lines), failing test, `<objective>` + `<verify>` | Everything else |

---

## 9. Open Questions (v1.0 Targets)

- [ ] Define merge conflict resolution protocol for `@build` (auto-resolve vs. human escalation threshold)
- [ ] Define `@review` agent for the "In Review" status — what it reads, what it can block
- [ ] Specify FILE_MAP.md generation algorithm (static analysis vs. LLM-assisted vs. tree-sitter)
- [ ] Define `token_budget` defaults per ticket type (feature vs. bugfix vs. refactor)
- [ ] Multi-workspace support: running parallel Epics in separate git worktrees
