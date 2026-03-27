# **📑 Project Spec: OpenCode Projects (OCP) Control Tower**

**Version:** 0.9 (GSD Unification \+ Concurrency Correctness \+ Schema Completeness)

**Core Logic:** Autoresearch-inspired Iterative Engineering via Remote or Local OpenCode Server, powered by GSD as the underlying context engineering and planning engine.

**Changelog from v0.8 to v0.9:**

* Unified .ocp/ and .planning/ into a single shared brain (Section 4 rewrite)  
* Defined Action Plan schema explicitly (Section 5.4)  
* Fixed FILE\_MAP.md staleness in concurrent execution via branch-scoped snapshots (Section 5.1)  
* Placed @test agent explicitly in the Ratchet loop (Section 6\)  
* Defined LLM Judge context budget (Section 5.6)  
* Added SYSTEM\_STATE.md / STATE.md compression trigger (Section 5.7, new)  
* Wired tokens\_used \+ token\_budget as live enforced fields (Section 4\)  
* Defined GSD ↔ OCP layer boundary (Section 2\)  
* Added strict API contract requirements for @plan to align @test and @build (Section 5.4)  
* Clarified Orchestrator-only writes for yq tracking to prevent permission crashes (Section 6\)  
* Required \--skip-permissions for headless Docker execution (Section 6\)  
* Appended Appendix A: Architectural Rationale & Impact

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

**.ocp/config.json schema:**

{  
  "environment": "remote",  
  "local\_project\_path": "/home/user/my-project",  
  "remote\_host": "dev.example.com",  
  "remote\_port": 4096,  
  "ssh\_key\_path": "\~/.ssh/id\_ed25519",  
  "judge\_context\_budget": 2000,  
  "default\_token\_budget": 50000,  
  "model\_profile": "balanced"  
}

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
token\_budget: 50000     \# enforced ceiling for this ticket's total agent spend  
tokens\_used: 0          \# updated live after each agent call  
branch: ocp/TKT-102  
\---

**Action Plan body (GSD XML task structure):**

\<task type="auto"\>  
  \<n\>Create Next.js API route for user login\</n\>  
  \<objective\>Single sentence. What done looks like.\</objective\>  
  \<context\_budget\>8000\</context\_budget\>  
  \<files\>  
    \<file action="CREATE"\>src/app/api/auth/login/route.ts\</file\>  
    \<file action="MODIFY" lines="45-67"\>src/lib/auth.ts\</file\>  
    \<file action="READ\_ONLY"\>tests/auth.test.ts\</file\>  
  \</files\>  
  \<snippets\>  
    \<\!-- src/lib/auth.ts lines 45-67 snippet here \--\>  
  \</snippets\>  
  \<constraints\>  
    Strict API Signature: function login(req: Request): Promise\<Response\>  
    Use jose for JWT, not jsonwebtoken.  
  \</constraints\>  
  \<verify\>npm run test:auth\</verify\>  
  \<done\>Valid credentials return cookie. Invalid credentials return 401.\</done\>  
\</task\>

## **5\. System Integrity & Concurrency Protocols**

### **5.1. Handling Concurrency (Git Branching, Snapshots & Cycle Detection)**

* **Execution:** When a ticket moves to In Progress, the system runs git checkout \-b ocp/\[ticket-id\].  
* **FILE\_MAP.md Snapshot:** At branch creation time, the system immediately snapshots the current FILE\_MAP.md scoped to this branch (.ocp/snapshots/FILE\_MAP.ocp-TKT-102.snapshot.md). @plan reads the **branch snapshot**, not the global FILE\_MAP.md. The global FILE\_MAP.md is only regenerated on merge to main.  
* **DAG Cycle Detection:** The Tauri backend evaluates depends\_on arrays using a Directed Acyclic Graph (DAG) algorithm.

### **5.2. Privilege Separation (Rogue Lobotomy Protection)**

Strict OS-level privilege separation ensures the agent cannot delete or corrupt the planning brain:

* **The Orchestrator:** The Tauri UI owns .planning/ and .ocp/ and manages all state and git commands.  
* **The Worker (Docker):** The OpenCode agent runs inside a container with explicit mount permissions:

\-v $(pwd)/src:/workspace/src:rw  
\-v $(pwd)/.planning:/workspace/.planning:ro  
\-v $(pwd)/.ocp:/workspace/.ocp:ro

The agent has strictly **Read-Only** access to all state files. It can only write to src/.

### **5.3. Mandatory Ticket Locking & Heartbeats**

When a ticket is In Progress, the UI applies a Read-Only lock to the plan body.

* **Lock file location:** .ocp/locks/TKT-102.lock  
* **Heartbeat Mechanism:** Updated every 60 seconds by the Tauri process.  
* **Zombie Lock Recovery:** UI surfaces a "Force Unlock & Reset" button if older than 5 minutes.

### **5.4. Hierarchical Context Funnels (Token Efficiency)**

Context is strictly separated by agent role.

| Agent | Reads | Does not read |
| :---- | :---- | :---- |
| @plan | FILE\_MAP.md snapshot, full files for relevant scope | Unrelated source files |
| @test | Requirements section of ticket, \<verify\> command | Codebase, FILE\_MAP |
| @build | Action Plan XML \+ inlined snippets only | Full files, FILE\_MAP.md, test files |
| @judge | Error log (last 100 lines), failing test file, \<objective\> \+ \<verify\> | Everything else |

**Strict API Contracts:** To prevent the API Contract Gap between @test and @build, the @plan agent must explicitly define exact function signatures, class names, and data payloads in the \<constraints\> block. Both sub-agents build against this unified contract.

### **5.5. Adversarial QA (Validation Integrity)**

* **@test** reads the requirements and writes the validation script **before @build starts**.  
* **@build** is physically blocked from editing the test file via Docker mount and the READ\_ONLY file action.

### **5.6. The Fail-Fast Pre-Flight & LLM Judge (Flake Checks)**

Validation runs in two gates to prevent expensive Ratchet resets on flaky tests.

1. **Gate 1 (Lightning Check):** Fast linters and compilers (e.g. tsc \--noEmit, eslint) run first.  
2. **Gate 2 (The LLM Judge):** If a heavy E2E test fails, a cheap/fast model reviews the failure (Classifies as CODE\_ERROR or FLAKE).

### **5.7. STATE.md Compression**

Triggered automatically when OCP calls /gsd:complete-milestone. A compression agent archives resolved decisions to .planning/archive/milestone-N-state.md, keeping the active STATE.md lean and token-efficient.

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
    └── If Action Plan exceeds context\_budget → @plan compresses snippets

4\.  TEST AUTHORING (before @build starts)  
    └── @test reads ticket Requirements  
    └── @test writes validation script  
    └── @build is BLOCKED from this file (Docker mount \+ READ\_ONLY file action)

5\.  BUILD  
    └── Tauri Orchestrator launches Docker Execution Engine.  
    └── MUST include \`--skip-permissions\` (or equivalent yolo mode) to prevent headless hanging.  
    └── @build reads Action Plan \+ inlined snippets only and implements code.

6\.  SYNC  
    └── git pull \--rebase origin main  
    └── Conflict markers detected → auto-fail → back to step 5

7\.  GATE 1 — LIGHTNING CHECK  
    └── Run fast linter / compiler (tsc \--noEmit, eslint)  
    └── FAIL → increment retry\_count, git reset \--hard → back to step 5  
    └── PASS → proceed to Gate 2

8\.  GATE 2 — HEAVY VALIDATION  
    └── @test validation script runs  
    └── PASS → proceed to step 9  
    └── FAIL → LLM Judge evaluates:  
        ├── CODE\_ERROR → git reset \--hard, increment retry\_count → back to step 5  
        └── FLAKE → re-run Gate 2, no reset, no retry increment

    └── retry\_count \== max\_retries:  
        ├── Preserve last error log in ticket file  
        ├── Status → Blocked  
        └── HALT

9\.  PASS — COMMIT & MERGE  
    └── Commit code with message: feat(\[ticket-id\]): \[objective\]  
    └── Merge branch → main  
    └── \*\*Global State Updates:\*\* Tauri Orchestrator synchronously updates global \`STATE.md\` and regenerates global \`FILE\_MAP.md\` on the main branch. (Never updated by concurrent agents).  
    └── Delete branch snapshot & lock file.  
    └── Ticket status → In Review (if reviewer set) or Done.

### **Token Tracking During Loop**

After every agent call (steps 3, 4, 5, 8), the **Tauri Orchestrator (Host OS)** patches the ticket frontmatter:

yq e ".tokens\_used \+= $NEW\_TOKENS" \-i .planning/\[plan-id\].md

*(Note: The Orchestrator performs this action because the Worker Docker container has Read-Only permissions to .planning/)*.

## **Appendix A: Architectural Rationale & Impact Assessment**

| Problem Statement | Solution / Proposal | Impact Assessment | Score |
| :---- | :---- | :---- | :---- |
| **Context rot** and quality degradation occur as the agent's context window fills with irrelevant data during long sessions. | Implement GSD's context engineering layer and the 'Ratchet' execution loop. Use atomic task plans (PLAN.md) to ensure each execution runs in a fresh 200k token context window, keeping main session context under 40%. | **High:** Maintains consistent code quality and system responsiveness throughout large projects; prevents the 'I'll be more concise now' failure mode. | **9/10** (Production Grade) |
| The **project brain (STATE.md) accumulates** too much historical data over time, becoming a major token cost and noise source. | Implement Autoresearch-inspired **STATE.md Compression** triggered during /gsd:complete-milestone to archive resolved decisions and keep active state lean. | **High:** Reduces token spend and improves agent focus by ensuring only active blockers and open decisions are in the immediate context. | **9/10** (Genius/Efficient) |
| Risk of agents writing **trivial or hallucinated tests** to falsely pass the verification loop ('escaping the Ratchet'). | Implement **'Adversarial QA'** where @test agents write validation scripts before @build starts, with @build being physically blocked from editing the test via Docker mount permissions. | **High:** Ensures validation integrity and prevents the AI from 'cheating' its way through requirements. | **9/10** (Least Risk) |
| **Concurrent execution** can lead to FILE\_MAP.md staleness where one task merges changes that another parallel task is unaware of. | Utilize **branch-scoped snapshots** of the FILE\_MAP.md at the start of each ticket/action plan execution, only regenerating the global map on merge to main. | **Medium:** Solves race conditions in parallel AI engineering workflows, ensuring agents always work against a consistent view of the codebase. | **10/10** (Concurrency Correctness) |
| **Manual approval** of every small action (git commits, bash commands) defeats the purpose of autonomous automation and freezes headless containers. | Adopt GSD's **\--skip-permissions** or 'yolo' mode to allow the agent to perform surgical, traceable, and meaningful atomic commits automatically without terminal blocking. | **Medium:** Significantly increases development speed and enables hands-off execution while maintaining a clean, bisect-ready git history. | **8/10** (Practical) |
| A **chat-based interface is insufficient** for managing a complex autonomous engineering organization or 'Epic' planning. | Build the **OCP Control Tower** (Tauri-based UI) to render GSD's .planning/ directory as a Kanban board, decoupling the user interface from the heavy AI execution engine. | **Medium:** Provides professional project management oversight (Jira-style) for autonomous agents, making the tool production-ready. | **7/10** (User Experience) |

