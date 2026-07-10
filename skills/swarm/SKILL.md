---
name: swarm
description: Orchestrate parallel pipelines with planning, code & review using roach-pi agents via subagent dispatch. Each task flows through Architect → Coder → Reviewer → QA → Commit. Configurable parallel pipelines (default 4). Use when you need to coordinate multi-agent task pipelines.
---

# Swarm Skill

## Overview
Orchestrate parallel pipelines with planning, code & review using roach-pi agents via `subagent()`. Each task flows through: **Architect → Coder → Reviewer → QA → Commit**. Configurable parallel pipelines (default 4).

## Roles

| Role | Agent | Responsibility |
|------|-------|----------------|
| **Orchestrator** | Lead (you) | Task assignment, pipeline coordination, progress tracking, final commits |
| **Solution Architect** | `planner` | Implementation planning, interface design, **AC refinement**. Dispatched with `subagent({ agent: "planner", effort: "deep" })`. |
| **Coder** | `worker` | Code implementation based on architect's plan. Dispatched with `subagent({ agent: "worker" })`. |
| **Code Reviewer** | `qa-reviewer`, `security-reviewer` | Code review, edge cases, logic errors, security. Dispatched with `subagent({ agent: "qa-reviewer" | "security-reviewer", effort: "deep" })`. |
| **QA Tester** | `qa-reviewer` | Unit test implementation, coverage verification. Dispatched with `subagent({ agent: "qa-reviewer" })`. |

## Task Lifecycle

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    Task Pipeline Flow                                   │
│                                                                        │
│  Task ──> Architect ──> Plan + AC Refinement ──> Coder ──> Code        │
│                    ↑                                   │               │
│                    └──────── Feedback Loop ────────────┘               │
│                                                                       │
│  Reviewed Code ──> QA ──> Tests ──> Pass ──> Commit ──> Done        │
│                        ↑        │                                  │
│                        └── Fix Tests ───────────────────────────────┘  │
│                                                                       │
│  Configurable parallel pipelines (default 4)                         │
└─────────────────────────────────────────────────────────────────────────┘
```

## Pipeline Phases

### Phase 1: Architecture & AC Refinement
- **Input:** Task description, priority, affected files
- **Agent:** planner (solution architect)
- **Output:** Implementation plan with:
  - File list to modify/create
  - Interface contracts (API, data models)
  - Dependencies and assumptions
  - Test strategy
  - **AC Refinement:** If acceptance criteria are missing or underspecified, the architect ADDS them directly into the task definition before planning. The architect supplements the original task with any missing AC items.
- **Gate:** Plan must be approved before coding begins

**AC Refinement Protocol:**
1. Architect reviews the original task description
2. Identifies gaps: missing edge cases, untested paths, unclear contracts
3. Supplements the task with additional AC items
4. Documents all AC items (original + supplemented)
5. Passes refined task + plan to coder

Example:
```
Original Task: "Add vehicle registration endpoint"
Architect supplements:
  + AC: Validate vehicle VIN format (17 chars, standard regex)
  + AC: Tenant isolation (tenantId from auth context)
  + AC: Idempotent POST (same VIN = 200 OK or 409 Conflict)
  + AC: Return 201 Created with full vehicle object
  + AC: Propagate to reporting-service via Pub/Sub
```

### Phase 2: Implementation
- **Input:** Architect's plan with refined AC, current codebase state
- **Agent:** worker (coder)
- **Output:** Implemented code changes
- **Process:**
  - Read existing files
  - Apply changes per plan
  - Run initial build/lint checks
  - Verify against all AC items (original + supplemented)

### Phase 3: Code Review (Iterative)
- **Input:** Coder's implementation
- **Agent:** `qa-reviewer` (primary), `security-reviewer` (security pass)
- **Effort:** `deep` (review agents run with deep reasoning for thorough analysis)
- **Process:**
  1. Reviewer analyzes code for bugs, edge cases, consistency
  2. If issues found → feedback to coder with specific fixes
  3. Coder applies fixes
  4. Repeat until reviewer approves (max 3 iterations)
- **Output:** Reviewed and approved code

**Review → Fix Cycle:**
```
Iteration 1: Reviewer finds N bugs → Coder fixes → Submit
Iteration 2: Reviewer finds M bugs → Coder fixes → Submit
Iteration 3: Reviewer finds K bugs → Coder fixes → Submit
  → If still failing after 3 iterations → Escalate to architect
```

### Phase 4: QA Testing
- **Input:** Reviewed code, test strategy from architect
- **Agent:** `qa-reviewer`
- **Process:**
  1. Implement unit tests covering all AC items
  2. Run test suite
  3. Verify coverage targets
  4. If gaps found → QA writes more tests or flags gaps to coder
- **Output:** Test suite with passing tests

### Phase 5: Commit
- **Agent:** Lead (orchestrator)
- **Process:**
  1. Final verification (build, lint, tests)
  2. Stage changes
  3. Commit with descriptive message referencing task ID
  4. Update task tracking
- **Output:** Committed changes, updated task status

## Stuck Detection: Fork-Deep Hint + Memory

When the coder is stuck on a problem for **5 turns** (attempts without progress), check workspace memories first, then fork-deep for a fresh hint. Once resolved, write the fix into memories so future pipelines benefit.

### What Counts as a Turn
A "turn" is one full cycle of: coder attempts → gets feedback/error → tries again. Examples:
- Coder writes code → reviewer rejects with feedback → coder fixes → still wrong
- Coder writes code → build fails on same line → coder adjusts → same error
- Coder writes code → unit test fails on same assertion → coder adjusts → same failure

### Stuck Flow
```
1. Check workspace memories for similar past issue
   → If found: apply known fix, skip fork-deep

2. If no memory match → fork-deep for hint:
   fork(task="""
   The coder is stuck on this task. Give a SHORT hint (2-4 sentences).

   TASK: [Task ID] - [Description]
   SERVICE: [Service name]
   BLOCKING ISSUE: [What's been tried 5 times]
   FILES: [Relevant files]
   Root cause and what to try next:
   """, effort="deep")

3. Pass hint to coder, apply fix
4. Write fix into workspace memories:
   memory_save(content="Problem: ... Root Cause: ... Fix: ...",
               tags=["stuck", "<service>", "<issue-type>"])
5. Reset turn counter to 0
6. If still stuck → escalate to architect for re-planning
```

### Turn Counter Reset
The 5-turn counter resets to 0 when:
- The coder moves to a different file or AC item
- The reviewer approves a fix
- The architect re-plans the task
- A fork-deep hint is applied

## Parallel Execution

### Rules
- **Parallelism:** Configurable (default 4 concurrent pipelines). Set higher to burn more tokens, lower to conserve.
- **Condition:** Tasks must be independent (different files, services, or repos)
- **Priority:** Follow task priority graph (see below)

### Independence Verification
Before launching 2 parallel pipelines, verify:
```
Task A files: [list]
Task B files: [list]
Overlap? → No → Safe to parallelize
Overlap? → Yes → Serialize or split files
```

### Parallelization Candidates by Service Type
```
Java Services (independent databases):
  - billing-service, carrier-service, delivery-service
  - driver-service, messaging-service, notification-service
  - reporting-service, tracking-bridge-service

Portals (independent Next.js apps):
  - erp-client-portal, erp-carrier-portal, erp-internal-portal

React Native Apps:
  - erp-driver-app, erp-carrier-app
```

## Task Priority Graph

### Priority Levels
```
P0: Foundation (Contract Fixtures & Visibility Rules)
    ↓
P1: Core Client (Client Contracts & Shipment Drilldown)
    ↓
P2: Internal Ops (Read Models & Dashboards)
    ↓
P3: Parity (Carrier/Driver Parity)
    ↓
P4: Migration (GraphQL Migration)
```

### Task Distribution Algorithm
```
1. Read task list with priority levels
2. Sort by priority (P0 → P4)
3. For each priority level:
   a. Find independent tasks (no file/service overlap)
   b. Assign up to N independent tasks to parallel pipelines (default 4)
   c. Remaining tasks queue for sequential processing
4. Advance to next priority level when current level is complete
```

### Task Queue Format
```
Task Queue:
┌──────────────────────────────────────────────────────────┐
│ P0 (Foundation)                                         │
│   [ ] IPD-1: Vehicle Registry - delivery-service       │
│   [ ] IPB-1: Billing Setup - billing-service            │
│   → Independent: Different services, different DBs      │
│   → Action: Launch both in parallel                    │
└──────────────────────────────────────────────────────────┘
┌──────────────────────────────────────────────────────────┐
│ P1 (Core Client)                                         │
│   [ ] ICP-1: Client Orders - erp-client-portal          │
│   [ ] ICP-2: Shipment Drilldown - erp-client-portal     │
│   → Dependent: Same portal, overlapping files           │
│   → Action: Serialize (ICP-1 → ICP-2)                  │
└──────────────────────────────────────────────────────────┘
```

## Execution Protocol

### Starting a Pipeline

When the user invokes `swarm [tasks]`, follow this exact flow:

**Step 1: Parse Tasks**
```
User: "swarm IPD-13, IPD-14"
→ Parse task IDs
→ Resolve task descriptions
→ Identify services/repos involved
```

**Step 2: Check Independence**
```
Task IPD-13: delivery-service/fleet/vehicle/
Task IPD-14: delivery-service/fleet/availability/
→ Same service, different modules
→ Check file overlap
→ If independent → parallelize, else → serialize
```

**Step 3: Launch Pipeline(s)**
```
Pipeline 1 (Task A):
  1. subagent(agent="planner", task="Plan + AC refinement", effort="deep") → Architecture
  2. subagent(agent="worker", task="Implementation per plan") → Code
  3. subagent(agent="qa-reviewer", task="Code review, bug hunt", effort="deep") → Review
  4. subagent(agent="worker", task="Apply review fixes") → Fixes
  5. subagent(agent="qa-reviewer", task="QA tests covering all AC") → Tests
  6. Orchestrator → Final Commit

Pipeline 2 (Task B) — if independent:
  Same flow, runs concurrently
```

### Pipeline Execution with Subagent

```
# Single pipeline (Task A)
1. architect = subagent(agent="planner", task="Plan [Task A]. Refine AC. Output plan.md", effort="deep")
2. coder = subagent(agent="worker", task="Implement [Task A] per plan")
3. review1 = subagent(agent="qa-reviewer", task="Review [Task A] code. Find bugs, edge cases", effort="deep")
4. if review1.findings:
     fix1 = subagent(agent="worker", task="Fix review findings: [findings]")
     review2 = subagent(agent="qa-reviewer", task="Re-review fixes", effort="deep")
5. qa = subagent(agent="qa-reviewer", task="Write tests for [Task A]. Cover all AC items")
6. commit = orchestrator runs git commit

# Parallel pipelines (Task A + Task B)
pipeline_a = subagent(agent="planner", task="Plan Task A", effort="deep")
pipeline_b = subagent(agent="planner", task="Plan Task B", effort="deep")
```

### Parallel Pipeline Orchestration

For true parallelism across independent tasks:
```
# Phase 1: Parallel Architecture
arch_A = subagent(agent="planner", task="Plan Task A", effort="deep")
arch_B = subagent(agent="planner", task="Plan Task B", effort="deep")

# Phase 2: Parallel Implementation (after both plans ready)
code_A = subagent(agent="worker", task="Implement Task A per plan")
code_B = subagent(agent="worker", task="Implement Task B per plan")

# Phase 3: Parallel Review
review_A = subagent(agent="qa-reviewer", task="Review Task A", effort="deep")
review_B = subagent(agent="qa-reviewer", task="Review Task B", effort="deep")

# Continue through QA and Commit in parallel
```

> **Note:** `subagent()` spawns isolated child Pi processes. The roach-pi agentic-harness extension manages depth limits (default 3) and cycle prevention. Ensure roach-pi is installed before running pipelines.

## Communication Protocol

### Between Orchestrator and Architect
```
Orchestrator → Architect: "Plan implementation for [Task] in [Service]"
Architect → Orchestrator: "Plan: [file list, contracts, AC refinement, dependencies]"
```

### Between Coder and Reviewer
```
Reviewer → Coder: "Fix: [specific issue] in [file] at [line] - [reason]"
Coder → Reviewer: "Fixed: [change summary]"
```

### Between QA and Coder
```
QA → Coder: "Test gap: [test name] needs [coverage] - [missing assertion]"
Coder → QA: "Fixed: [test update]"
```

## Error Handling

| Error | Recovery |
|-------|----------|
| Architect plan rejected | Re-plan with feedback |
| Coder build fails | Coder fixes build, re-submits |
| Review finds blocker | Coder fixes, re-submits (max 3 iterations) |
| QA test failures | Coder fixes code or QA adjusts tests |
| Pipeline timeout | Pause, resume, or escalate |
| AC gap discovered late | Architect supplements AC, coder updates |
| Coder stuck (5 turns) | Fork-deep for outside-perspective hint → apply → retry |

## Progress Reporting Format

```
Pipeline Status:
┌─────────────────────────────────────────────────────────┐
│ Pipeline 1: [Task ID] - [Service]                    │
│   Phase: Implementation (Phase 2/5)                   │
│   Progress: Coder writing [file]                     │
│   Status: In Progress                                 │
│   AC Items: [N] original + [M] supplemented         │
│                                                        │
│ Pipeline 2: [Task ID] - [Service]                    │
│   Phase: Code Review (Phase 3/5)                    │
│   Progress: Reviewer analyzing [file]                │
│   Status: Awaiting Coder fixes (Iteration 2/3)       │
└─────────────────────────────────────────────────────────┘

Completed Tasks: [List]
Queued Tasks: [List]
```

## Success Criteria

A task is **complete** when ALL are met:
- [ ] Architect plan approved (including refined AC)
- [ ] Coder implementation passes build/lint
- [ ] Reviewer approves (max 3 review iterations)
- [ ] QA tests pass covering all AC items
- [ ] Orchestrator commits with descriptive message
- [ ] Task status updated in tracking

## Prerequisites

This skill relies on **roach-pi's agentic-harness extension** for its agent dispatch system. Pipeline roles map to named agents defined in roach-pi's `agents/` directory:

- `planner` — Implementation planning and architecture
- `worker` — General-purpose execution with full tools
- `qa-reviewer` — Code review, edge-case hunting, and QA verification
- `security-reviewer` — Security-focused code review

Agents are dispatched via `subagent({ agent: "<name>", task: "..." })`. Each call spawns an isolated child Pi process with the agent's specialized system prompt. The orchestrator (you) coordinates the pipeline, collects results, and drives the next phase.

## Usage

### Triggers
```
"swarm [task_id]"
"swarm [task_1, task_2]"
"run pipeline for [task] in [service]"
"orchestrate [task_list]"
```

### Example Workflow
```
User: "swarm IPD-13, IPB-1"

Orchestrator:
1. Verify IPD-13 (delivery-service) and IPB-1 (billing-service) are independent
2. Check task priority levels
3. Start Pipeline 1: IPD-13
   → Architect (planner): Plan + AC refinement
   → Coder (worker): Implement
   → Reviewer (qa-reviewer): Review + pushback loop
   → QA (qa-reviewer): Tests
   → Commit
4. Start Pipeline 2: IPB-1
   → Same flow, parallel execution
5. Track both pipelines
6. Report completion status
```

## Key Principles

1. **AC Refinement First:** Architect always supplements missing AC before coding begins
2. **Independent Tasks Only:** Never run dependent tasks in parallel
3. **Sequential Phases:** Each pipeline phase completes before next begins
4. **Feedback Loops:** Review → Coder → Review cycles within each pipeline (max 3)
5. **Quality Gates:** Each phase must pass before advancing
6. **Parallel Limit:** Configurable (default 4). Go higher to burn more tokens, lower to conserve.
7. **Task Priority:** Follow the priority graph (P0 → P1 → P2 → P3 → P4)
8. **No Orphaned Code:** Every committed change has tests and review approval
9. **Fork-Deep Hint:** When the coder is stuck for 5 turns without progress, fork a deep-reasoning child for an outside-perspective hint

## Files Affected Per Task

Track which files each task modifies to ensure independence:

```
Task: [ID]
Service: [service-name]
Files:
  - src/main/kotlin/com/erp/[service]/[File].kt
  - src/test/kotlin/com/erp/[service]/[File]Test.kt
  - schema/[service]/[migration].sql

Dependency Check:
  - Overlapping files with other active pipelines?
  - Shared interfaces that might diverge?
  - Common utilities being modified?
```

## Skill Metadata

```yaml
skill: swarm
description: Orchestrate parallel pipelines with planning, code & review.
version: 4.0
dependencies:
  - roach-pi agentic-harness — provides named agents (planner, worker, qa-reviewer, security-reviewer)
  - subagent tool — dispatches named agents as isolated child Pi processes
triggers:
  - "swarm"
  - "run pipeline"
  - "orchestrate tasks"
roles:
  - orchestrator (lead)
  - architect (planner agent)
  - coder (worker agent)
  - reviewer (qa-reviewer agent, security-reviewer agent)
  - qa-tester (qa-reviewer agent)
max_parallel: 4 (default, configurable)
pipeline_phases:
  - architecture_with_ac_refinement
  - implementation
  - code_review_iterative
  - qa_testing
  - commit
stuck_detection:
  trigger: 5 turns without progress
  action: fork_deep_for_hint → apply_hint → reset_counter
  escalation: architect_replan_if_hint_fails
```
