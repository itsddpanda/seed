# **📑 Project Spec: OpenCode Projects (OCP) Control Tower**

**Version:** 0.10.0 (Release Candidate: Phase Verification & Conflict Protocols)

**Core Logic:** Autoresearch-inspired Iterative Engineering via Remote or Local OpenCode Server, powered by GSD as the underlying context engineering and planning engine.

**Changelog from v0.9 to v0.10.0:**

* Resolved @review agent open question via GSD Phase Verification (/gsd:review).  
* Resolved merge conflict open question via strict human escalation protocol (respecting the context funnel limits of the @build agent).  
* Crossed off previous targets from Section 7\.

## **1\. System Overview**

OCP is a Jira-style project management graphical interface that orchestrates OpenCode agents to execute an entire software development lifecycle autonomously.

Instead of a chat interface, users manage an "Autonomous Engineering Org" through Kanban boards and Epic planning. The UI is decoupled from the execution engine, allowing heavy AI tasks and code execution to run on a remote server while the user controls it from a lightweight local client, or run entirely locally.

OCP does **not** maintain its own planning data store. It fires GSD workflows and renders GSD's .planning/ directory as a Kanban board. GSD is the brain; OCP is the execution harness and UI on top of it.

## **2\. The Architecture & Connection Plumbing**

| Component | Technology | Role / Rationale |
| :---- | :---- | :---- |
| **Frontend UI** | Tauri (Rust/React) | Native desktop Kanban board and live streaming terminal feeds. Acts as the Orchestrator with full system privileges. Renders .planning/ as Kanban state. |
| **Planning Engine** | GSD (get-shit-done) | Context engineering, phase planning, and state management. OCP fires GSD commands to generate plans, but OCP manages the actual execution phase. |
| **Execution Engine** | OpenCode \+ Docker | Runs inside a standardized Workspace Container (DevContainer). Acts as the Worker, executing prompts and editing files within strict boundaries. |
| **Transport Layer** | SSH Tunnel / IPC | Secures the connection. For remote, local UI connects to localhost:4096, securely port-forwarded. For local, direct IPC/localhost. |
| **Data Storage** | Markdown \+ YAML | All planning state lives in .planning/ (managed by GSD). Execution metadata lives in .ocp/. Git is the ultimate state-tracker. |

### **GSD ↔ OCP Layer Boundary (The Execution Handshake)**

OCP and GSD map to each other directly. **GSD is the Planner; OCP is the Executioner.**

| OCP concept | GSD equivalent | Notes |
| :---- | :---- | :---- |
| Epic | Milestone (ROADMAP.md) | /gsd:new-milestone creates an Epic |
| Feature | Phase | /gsd:plan-phase N plans a Feature |
| Ticket | PLAN.md task | GSD's XML task structure is the Action Plan |
| SYSTEM\_STATE.md | STATE.md | Same file — OCP reads GSD's STATE.md |
| .ocp/ planning data | .planning/ | OCP drops its own planning dirs entirely |

OCP Kanban UI (Tauri)  
  └── fires: /gsd:plan-phase  
        └── GSD writes: .planning/ (STATE.md, PLAN.md, SUMMARY.md)  
              └── OCP reads XML Plan → Mounts Docker → Executes Agent → Runs Ratchet

## **3\. Configuration & Environment Switching**

The application maintains a **Global Configuration Registry** on the user's local machine stored in .ocp/config.json.

* **Load Existing Configuration:** Users select from saved project profiles to auto-populate UI settings.  
* **Enter New Configuration:** The UI requires:  
  * **Always:** Environment Type (Local/Remote), Local Project Path.  
  * **Remote Only:** Remote Host Address, Port Number, SSH Key Path.

## **4\. Data Storage: Unified File-Based Brain**

All planning state lives in .planning/ (owned and written by GSD). OCP's .ocp/ directory holds only execution metadata that GSD does not produce.

**Unified Directory Structure:**

my-project/  
├── .planning/                   ← GSD brain (OCP reads, GSD writes)  
│   ├── STATE.md                 ← replaces SYSTEM\_STATE.md  
│   ├── ROADMAP.md               ← replaces epics/  
│   ├── REQUIREMENTS.md  
│   ├── 1-CONTEXT.md  
│   ├── 1-1-PLAN.md              ← replaces tickets/ (= Action Plan)  
│   ├── 1-1-SUMMARY.md  
│   ├── archive/                 ← compressed milestone state (auto-generated)  
│   └── research/  
│  
├── .ocp/                        ← OCP execution metadata only  
│   ├── locks/                   ← .lock files \+ heartbeats  
│   ├── FILE\_MAP.md              ← codebase index (OCP-specific)  
│   ├── snapshots/               ← branch-scoped FILE\_MAP snapshots  
│   ├── docker/                  ← container configs  
│   └── config.json              ← connection profiles \+ budgets  
│  
├── src/  
└── package.json

### **Ticket / Action Plan Schema**

Each ticket is a GSD PLAN.md file with extended OCP frontmatter.

\---  
\# GSD fields  
plan\_id: 1-1-PLAN  
phase: 1  
status: To Do           \# To Do | In Progress | In Review | Done | Blocked

\# OCP execution fields  
ticket\_id: TKT-102  
priority: High  
assignee: "@build"  
reviewer: "rokicool"  
parent\_feature: FEAT-1  
depends\_on:  
  \- TKT-101  
retry\_count: 0  
max\_retries: 3  
token\_budget: 50000       
tokens\_used: 0            
branch: ocp/TKT-102  
\---

## **5\. System Integrity & Concurrency Protocols**

### **5.1. Handling Concurrency (Git Branching, Snapshots & Cycle Detection)**

* **Execution:** When a ticket moves to In Progress, the system runs git checkout \-b ocp/\[ticket-id\].  
* **FILE\_MAP.md Snapshot:** At branch creation time, the system immediately snapshots the current FILE\_MAP.md scoped to this branch (.ocp/snapshots/FILE\_MAP.ocp-TKT-102.snapshot.md). @plan reads the **branch snapshot**, not the global FILE\_MAP.md. The global FILE\_MAP.md is only regenerated on merge to main.  
* **DAG Cycle Detection:** The Tauri backend evaluates depends\_on arrays using a Directed Acyclic Graph (DAG) algorithm.

### **5.2. Privilege Separation (Rogue Lobotomy Protection)**

Strict OS-level privilege separation ensures the agent cannot delete or corrupt the planning brain:

* **The Orchestrator:** The Tauri UI owns .planning/ and .ocp/ and manages all state and git commands.  
* **The Worker (Docker):** The OpenCode agent runs inside a container with explicit mount permissions. It has strictly **Read-Only** access to all state files. It can only write to src/.

### **5.3. Mandatory Ticket Locking & Heartbeats**

When a ticket is In Progress, the UI applies a Read-Only lock to the plan body.

* **Lock file location:** .ocp/locks/TKT-102.lock  
* **Heartbeat Mechanism:** Updated every 60 seconds by the Tauri process.  
* **Zombie Lock Recovery:** UI surfaces a "Force Unlock & Reset" button if older than 5 minutes.

### **5.4. Hierarchical Context Funnels (Token Efficiency)**

Context is strictly separated by agent role to prevent token explosions.

| Agent | Reads | Does not read |
| :---- | :---- | :---- |
| @plan | FILE\_MAP.md snapshot, full files for relevant scope | Unrelated source files |
| @test | Requirements section of ticket, \<verify\> command | Codebase, FILE\_MAP |
| @build | Action Plan XML \+ inlined snippets only | Full files, FILE\_MAP.md, test files |
| @judge | Error log (100 lines), failing test file, \<objective\> \+ \<verify\> | Everything else |
| @review | Cross-AI phase check of all completed tickets in a Feature | Codebase outside of the feature's scope |

### **5.5. Adversarial QA (Validation Integrity)**

* **@test** reads the requirements and writes the validation script **before @build starts**.  
* **@build** is physically blocked from editing the test file via Docker mount and the READ\_ONLY file action.

### **5.6. The Fail-Fast Pre-Flight & LLM Judge (Flake Checks)**

Validation runs in two gates to prevent expensive Ratchet resets on flaky tests.

1. **Gate 1 (Lightning Check):** Fast linters and compilers run first.  
2. **Gate 2 (The LLM Judge):** If a heavy E2E test fails, a cheap/fast model classifies the failure as CODE\_ERROR or FLAKE.

### **5.7. STATE.md Compression**

Triggered automatically when OCP calls /gsd:complete-milestone. A compression agent archives resolved decisions to .planning/archive/milestone-N-state.md, keeping the active STATE.md lean and token-efficient.

### **5.8. Phase Verification (The @review Agent Protocol)**

Because an OCP Feature maps directly to a GSD Phase, verification is handled holistically at the Feature level:

1. When all Tickets (PLAN.md tasks) in a Feature complete their individual Ratchet loops, they move to the "In Review" status.  
2. The UI triggers GSD's native phase verification (/gsd:review).  
3. The @review agent performs a cross-AI peer review, ensuring "must-haves were delivered after execution".  
4. **Pass:** Tickets move to "Done". Feature is closed.  
5. **Fail:** GSD automatically diagnoses failures and creates verified fix plans. OCP renders these natively as new "To Do" tickets, restarting the execution cycle securely.

## **6\. The "Ratchet" Execution Loop**

1\.  TRIGGER  
    └── UI sets ticket status → In Progress

2\.  SETUP  
    └── git checkout \-b ocp/\[ticket-id\]  
    └── Snapshot FILE\_MAP.md → .ocp/snapshots/FILE\_MAP.\[ticket-id\].snapshot.md  
    └── Initialize Docker DevContainer mounts (src:rw, .planning:ro, .ocp:ro)  
    └── Create .ocp/locks/\[ticket-id\].lock with initial timestamp

3\.  PLAN  
    └── @plan reads FILE\_MAP branch snapshot  
    └── @plan generates Action Plan (PLAN.md XML structure) with strict API signatures

4\.  TEST AUTHORING  
    └── @test writes validation script (@build is BLOCKED from this file)

5\.  BUILD  
    └── Tauri Orchestrator launches Docker Execution Engine.  
    └── MUST include \`--skip-permissions\` (yolo mode) to prevent headless hanging.  
    └── @build reads Action Plan \+ inlined snippets only and implements code.

6\.  SYNC & CONFLICT PROTOCOL  
    └── git pull \--rebase origin main  
    └── Conflict markers detected → \*\*HALT AND ESCALATE TO HUMAN.\*\*  
        \*Rationale: GSD's "Wave Execution" ensures conflicting plans run sequentially.   
        A merge conflict implies an anomaly bypassed the planning engine. Because @build operates   
        with a lobotomized context funnel, it is mathematically unsafe for it to resolve complex conflicts.\*

7\.  GATE 1 — LIGHTNING CHECK  
    └── Run fast linter / compiler (tsc \--noEmit, eslint)  
    └── FAIL → increment retry\_count, git reset \--hard → back to step 5

8\.  GATE 2 — HEAVY VALIDATION  
    └── @test validation script runs  
    └── FAIL → LLM Judge evaluates (CODE\_ERROR resets; FLAKE re-runs).

9\.  PASS — COMMIT & MERGE  
    └── Commit code with message: feat(\[ticket-id\]): \[objective\]  
    └── Merge branch → main  
    └── Tauri Orchestrator synchronously updates global \`STATE.md\` & \`FILE\_MAP.md\`.  
    └── Ticket status → In Review.

10\. PHASE REVIEW  
    └── When all Feature tickets hit In Review → Fire /gsd:review (Section 5.8)

## **7\. Open Questions (v1.0 Targets)**

* \[x\] \~\~Define merge conflict resolution protocol for @build\~\~ *(Resolved: Strict human escalation)*  
* \[x\] \~\~Define @review agent for the "In Review" status\~\~ *(Resolved: Mapped to /gsd:review Phase Verification)*  
* \[ \] Specify FILE\_MAP.md generation algorithm (static analysis vs. LLM-assisted vs. tree-sitter)  
* \[ \] Define token\_budget defaults per ticket type (feature vs. bugfix vs. refactor)  
* \[ \] Multi-workspace support: running parallel Epics in separate git worktrees
