# **📑 Project Spec: OpenCode Projects (OCP) Control Tower**

**Version:** 0.20.0 (The GSD 2.0 Paradigm Shift: SQLite, Native Execution, & RPC Orchestration)

**Core Logic:** Autoresearch-inspired Iterative Engineering via local or remote GSD 2.0 Daemon (Pi SDK). OCP acts as the Enterprise Kanban Visualization and Multi-Workspace Orchestrator on top of GSD's native autonomous execution engine.

**Changelog from v0.10.1 to v0.20.0 (Major Pivot):**

* **Removed OpenCode Dependency:** GSD 2.0 is now a standalone runtime built on the Pi SDK. OCP no longer requires OpenCode.  
* **Removed Docker Execution Sandboxes:** GSD 2.0 natively enforces strict phase pipelines and context clearing. The custom "Ratchet Loop" and lobotomized agents have been deprecated in favor of GSD's /gsd auto engine.  
* **Shifted Data Layer to SQLite:** Replaced the pure Markdown .planning/ directory with GSD's atomic SQLite database (.gsd/).  
* **Solved Multi-Workspace Concurrency:** Inherited GSD 2.0's native Git Worktree isolation.

## **1\. System Overview**

OCP is a Jira-style project management graphical interface for Autonomous Software Engineering.

With the advent of GSD 2.0, the heavy lifting of code execution, AST parsing, Git isolation, and test validation is now handled natively by the gsd daemon. **OCP's role has elevated to a pure Enterprise Control Tower.** Instead of writing complex file-locking and Docker orchestration logic, OCP focuses on what a CLI cannot do: Multi-project visualization, graphical Kanban boards, cross-repo dependency tracking (DAGs), and intuitive drag-and-drop roadmap planning.

## **2\. The Architecture & Connection Plumbing**

The architecture is now a streamlined, two-tier Client/Server model communicating via JSON-RPC.

| Component | Technology | Role / Rationale |
| :---- | :---- | :---- |
| **The Control Tower (Frontend)** | Tauri (Rust/React) | Native desktop UI. Reads the .gsd/ SQLite DB to render boards. Triggers actions via JSON-RPC. |
| **The Engine (Backend Daemon)** | GSD 2.0 (Pi SDK / Node) | Runs gsd \--mode rpc. Handles all agentic logic, file system operations, and native Git isolation. |
| **The Source of Truth** | SQLite (.gsd/) | Replaces .planning/ markdown files. Ensures atomic, corruption-free state transitions. |
| **Execution Sandbox** | Native Git Worktrees | GSD 2.0 isolates execution natively. No Docker required. |

### **GSD ↔ OCP Layer Boundary**

| OCP Kanban Concept | GSD 2.0 Equivalent | Notes |
| :---- | :---- | :---- |
| Epic | Milestone | Highest level grouping in the DB |
| Feature / Ticket | Slice / Task | Mapped natively in SQLite. Classified as Light, Standard, or Heavy by GSD. |
| "In Progress" Status | /gsd auto \--target \[id\] | Dragging a ticket triggers the daemon to run the auto-pipeline on an isolated worktree. |
| Board State | .gsd/gsd.sqlite | Tauri reads the DB directly or via RPC queries to render the UI. |

## **3\. Configuration & RPC Environment**

The Tauri app connects to the GSD daemon either locally or remotely.

1. **Local Mode:** Tauri automatically spins up the gsd \--mode rpc background process upon opening a project directory.  
2. **Remote Mode:** Tauri connects to an external server IP over a secured WebSocket/JSON-RPC tunnel, allowing users to run heavy agent workloads on dedicated cloud hardware while controlling it from a lightweight laptop.

## **4\. Data Storage: The SQLite Shift**

The complex unified file-based brain has been simplified. Markdown is now a fallback, and atomic transactions rule the system.

my-project/    
├── .gsd/                        ← The entire Brain & State (Owned by GSD)  
│   ├── gsd.sqlite               ← Atomic state (Milestones, Tasks, Logs)  
│   ├── agent/KNOWLEDGE.md       ← Global architectural rules injected into prompts  
│   └── models.json              ← AI model configs (OpenAI, Anthropic, Ollama)  
│    
├── .ocp/                        ← Lightweight UI configs (Owned by Tauri)  
│   └── ui-state.json            ← Kanban column widths, saved filters, theme  
│    
└── src/                         ← Source code

## **5\. The "Auto" Execution Pipeline (Replacing the Ratchet Loop)**

OCP no longer manually scripts the steps of execution. When a user moves a ticket to "In Progress", OCP dispatches the task to the GSD daemon, which enforces its own strict, hallucination-resistant pipeline:

1. **Brainstorm & Design:** For "Heavy" tasks (\>300 lines), GSD forces the agent to write a design doc before coding.  
2. **Test-First (Adversarial QA):** GSD naturally enforces writing the validation script before implementation.  
3. **Implement:** GSD uses its native Rust gsd\_engine.node to efficiently parse AST and context.  
4. **Self-Review & Verify:** GSD natively blocks completion if tests, linters, or type-checkers fail, running auto-fix loops autonomously.  
5. **Commit & Sync:** GSD ensures dirty-tree protections and squashes commits.

## **6\. Multi-Workspace & Concurrency (Native Git Isolation)**

OCP leverages GSD 2.0's native git management to run parallel tickets seamlessly.

* **Configuration:** GSD is configured with git.isolation: worktree.  
* **The Flow:** When OCP triggers two tickets concurrently, GSD creates separate, isolated Git worktrees under the hood.  
* **Dependency Dispatch:** OCP evaluates the visual DAG (Directed Acyclic Graph) of the Kanban board and dispatches independent tasks to the GSD RPC daemon in parallel. Dependent tasks remain "Blocked" in the UI until the GSD daemon emits a slice\_completed event via WebSocket.

## **7\. Open Questions (v1.0 Targets)**

* \[ \] **The Rust/Node Deadlock (Issue \#453):** GSD 2.0 currently suffers from a critical bug where async zlib callbacks into the native Rust engine cause an infinite loop/freeze. OCP must implement a watchdog in Tauri to detect a frozen GSD daemon and force-restart it gracefully.  
* \[ \] **Remote RPC Security:** Define the authentication mechanism (e.g., mTLS or JWT) for when the Tauri UI connects to a remote GSD daemon over the internet.  
* \[ \] **Visual DAG Editor:** Design the UI interface for creating depends\_on relationships between tickets graphically before dispatching them to GSD.
