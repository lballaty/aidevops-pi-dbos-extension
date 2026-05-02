# Agent Implementation Instructions: Pi Native DBOS Durable Kernel
## 1. Project Intent
This repository exists to build a Pi native extension that adds durable, auditable, resumable agent execution using DBOS and Postgres.
The goal is not to replace Pi.
The goal is to extend Pi in the same spirit as Pi itself:
> If a capability is not needed, do not build it.
The extension should make Pi more suitable for serious engineering use by adding:
1. Durable execution
2. Crash recovery
3. Run state persistence
4. Tool call logging
5. Artifact tracking
6. Approval gates
7. Governance boundaries
8. Minimal CLI/TUI visibility
9. Later optional web UI
10. Safe self-adaptation support
## 2. Core Concept
Pi remains the agentic building framework.
DBOS becomes the durable execution substrate.
The UI becomes a Pi-facing control surface, not a separate DBOS dashboard.
The system should look like this:
```text
Developer / Engineer
   ↓
Pi CLI / Pi Chat / Pi TUI
   ↓
Pi Extension: aidevops-pi-dbos-extension
   ↓
DBOS Workflow Runtime
   ↓
Postgres
   ↓
Durable runs, steps, logs, artifacts, approvals
```

## 3. What This Project Is

This project is:

* A Pi extension
* A DBOS-backed durable execution layer
* A governance and approval layer
* A minimal visibility cockpit
* A foundation for self-adapting agentic engineering workflows

This project is not:

* A fork of Pi
* A replacement for Pi
* A replacement for Linux
* A standalone workflow SaaS
* A generic dashboard first product
* A large no-code workflow builder

## 4. Primary Users

The initial target users are:

* Developers
* Engineering teams
* DevOps engineers
* AI-assisted coding users
* Agentic engineering platform builders

The primary interaction model is:

1. CLI first
2. Chat second
3. Visual workflow/TUI in parallel
4. Web UI later only where needed

## 5. Offline and On-Prem Requirement

The system must be designed to run offline and on-prem from day one.

Initial required local stack:

Linux or macOS development machine
Node.js
Pi
DBOS TypeScript runtime
Postgres
Git
Local filesystem
Optional local LLM runtime

Cloud services may be added later, but the core design must not depend on them.

## 6. Implementation Principle

Use the smallest working loop first.

The first working loop is:

```text
User intent
   ↓
Create durable task
   ↓
Run Pi agent inside DBOS workflow
   ↓
Record tool calls and artifacts
   ↓
Validate result
   ↓
Complete or request approval
```

Do not build advanced features until the basic loop works.

## 7. Required First Milestone

The first milestone is a minimal but working durable Pi run.

Acceptance criteria:

1. The extension can be loaded by Pi.
2. A Pi command can start a durable run.
3. DBOS creates and tracks the workflow.
4. Postgres stores run state.
5. Tool calls or simulated tool calls are logged.
6. Artifacts can be recorded.
7. A run can finish as completed or failed.
8. The user can inspect run status.
9. The code builds and has at least basic tests.

## 8. Suggested Repository Structure

Create the following structure:

```
.
├── README.md
├── AGENT_IMPLEMENTATION_INSTRUCTIONS.md
├── package.json
├── tsconfig.json
├── .env.example
├── docs/
│   ├── architecture.md
│   ├── implementation-plan.md
│   ├── pi-extension-boundary-spike.md
│   ├── governance-model.md
│   └── ui-concept.md
├── migrations/
│   └── 001_initial_schema.sql
├── src/
│   ├── index.ts
│   ├── dbos/
│   │   ├── client.ts
│   │   ├── workflows.ts
│   │   └── queues.ts
│   ├── pi/
│   │   ├── extension.ts
│   │   ├── commands.ts
│   │   ├── tools.ts
│   │   └── events.ts
│   ├── governance/
│   │   ├── policy.ts
│   │   ├── approvals.ts
│   │   └── risk-classifier.ts
│   ├── persistence/
│   │   ├── repositories.ts
│   │   └── types.ts
│   ├── ui/
│   │   ├── tui.ts
│   │   └── cockpit.ts
│   └── validation/
│       ├── diff-validator.ts
│       └── command-validator.ts
└── tests/
    ├── durable-run.test.ts
    └── governance.test.ts
```

## 9. Core Data Model

Start with the following logical objects:

tasks
runs
steps
tool_calls
artifacts
approvals
validation_results
system_changes

Do not add more tables unless the implementation needs them.

Minimum statuses:

pending
running
waiting_approval
completed
failed
cancelled

## 10. Initial Postgres Schema Requirements

Create tables for:

**tasks**

Stores user intent.

Fields:

id
title
intent
status
created_at
updated_at
created_by
metadata

**runs**

Stores execution instances.

Fields:

id
task_id
status
started_at
finished_at
dbos_workflow_id
pi_session_id
error_message
metadata

**steps**

Stores durable workflow steps.

Fields:

id
run_id
step_name
status
started_at
finished_at
error_message
metadata

**tool_calls**

Stores tool usage.

Fields:

id
run_id
step_id
tool_name
input
output
status
created_at
risk_level

**artifacts**

Stores generated outputs.

Fields:

id
run_id
artifact_type
path
content_hash
summary
created_at
metadata

**approvals**

Stores human approval requests.

Fields:

id
run_id
approval_type
status
requested_at
resolved_at
requested_reason
decision_reason
metadata

**validation_results**

Stores validation outcomes.

Fields:

id
run_id
validator_name
status
details
created_at

## 11. DBOS Responsibilities

DBOS should handle:

* Durable workflow execution
* Step checkpointing
* Retry behavior
* Crash recovery
* Workflow status
* Durable background execution

DBOS should not handle:

* Agent reasoning
* Prompt design
* UI logic
* Policy decisions
* Final human approval decisions

## 12. Pi Responsibilities

Pi should handle:

* Agent loop
* Tool execution
* Developer interaction
* Coding/building behavior
* Extension loading
* CLI/TUI integration
* Promptable self-improvement behavior

## 13. Extension Responsibilities

This extension should handle:

* Mapping Pi sessions to DBOS workflows
* Persisting run state
* Logging tool calls
* Capturing artifacts
* Enforcing approval gates
* Displaying run visibility
* Providing commands and tools to Pi

## 14. First Commands to Implement

Implement these first:

```
/dbos task create <intent>
/dbos run <task_id>
/dbos status <run_id>
/dbos inspect <run_id>
/dbos approvals
/dbos approve <approval_id>
/dbos reject <approval_id>
```

If Pi command registration syntax differs, adapt to the actual Pi extension API.

Do not invent unsupported Pi APIs. Inspect Pi documentation and examples first.

## 15. First Tools to Expose to Pi

Expose these as Pi tools if supported:

create_durable_task
start_durable_run
record_tool_call
record_artifact
request_approval
record_validation_result
complete_run
fail_run

Each tool must:

1. Validate input.
2. Write to Postgres.
3. Return structured JSON.
4. Avoid destructive side effects unless explicitly approved.

## 16. Governance Model

The extension must classify operations by risk.

Initial risk levels:

low
medium
high
critical

Low risk examples:

read file
inspect status
list runs
create draft artifact

Medium risk examples:

edit documentation
modify non-runtime config
generate migration proposal

High risk examples:

edit source code
change DBOS workflow code
change Pi extension code
change command policy
execute shell command

Critical risk examples:

delete files
run destructive database commands
modify approval policy
disable audit logging
change authentication or secrets handling

Initial rule:

High and critical risk actions require approval.
Critical actions should be blocked unless explicitly enabled in config.

## 17. Self-Adaptation Boundary

The long-term goal is that Pi can help improve this extension.

However, self-adaptation must be controlled.

Agents may propose changes to:

source code
DBOS workflows
Pi extension commands
UI components
policy files
documentation
tests

Agents must not directly apply high-risk platform changes without approval.

Required flow:

```text
agent proposes change
   ↓
change is recorded as artifact
   ↓
validation runs
   ↓
approval requested if required
   ↓
human approves
   ↓
change applied
   ↓
result logged
```

## 18. UI Requirements

Do not build a full dashboard first.

Start with minimal visibility.

Initial TUI/cockpit should show:

active runs
recent failed runs
pending approvals
latest artifacts
step timeline for one run

User actions:

inspect
approve
reject
retry
cancel

Web UI is optional later.

The UI should expose platform concepts:

task
run
step
artifact
approval
validation

Avoid exposing DBOS internals unless needed for debugging.

## 19. Documentation Requirements

Create and maintain:

docs/architecture.md
docs/implementation-plan.md
docs/pi-extension-boundary-spike.md
docs/governance-model.md
docs/ui-concept.md

Each document should be practical and implementation-focused.

Do not write marketing copy.

## 20. Pi Extension Boundary Spike

Before implementing advanced features, verify:

1. How Pi loads extensions.
2. How commands are registered.
3. How tools are registered.
4. Whether tool calls can be intercepted.
5. Whether session events are available.
6. Whether TUI components can be added.
7. Whether runs can be paused/resumed.
8. Whether Pi supports project-local extensions.
9. Whether extensions can persist external state.
10. Whether extensions can safely wrap tool execution.

Document findings in:

docs/pi-extension-boundary-spike.md

If an API does not exist, state that clearly.

Do not guess.

## 21. Forking Rule

Do not fork Pi unless the extension API blocks a mandatory capability.

A fork is allowed only if one of these is impossible through extensions:

intercept tool calls
pause or resume execution
persist session state externally
add approval gates
add visibility UI
wrap risky operations

If a fork becomes necessary, document:

why extension approach failed
which Pi core change is required
how small the fork change can be
whether upstream contribution is possible

## 22. Testing Requirements

Implement tests for:

task creation
run creation
step logging
tool call logging
approval creation
approval resolution
risk classification
validation result recording

Tests must not require cloud services.

Use local Postgres or a test database container.

## 23. Configuration Requirements

Use .env.example for configuration.

Minimum variables:

```
DATABASE_URL=
DBOS_APP_NAME=aidevops-pi-dbos-extension
AIDEVOPS_APPROVAL_REQUIRED_FOR_HIGH_RISK=true
AIDEVOPS_BLOCK_CRITICAL_ACTIONS=true
AIDEVOPS_LOCAL_MODE=true
```

Do not commit real secrets.

## 24. Security Requirements

Do not implement broad shell execution without guardrails.

Do not allow destructive database commands by default.

Do not allow agents to disable logging.

Do not allow agents to modify approval policy without approval.

Do not store secrets in logs.

All risky actions must be recorded.

## 25. Coding Standards

Use TypeScript.

Use clear types.

Use small modules.

Comment code enough for a junior developer to understand.

Avoid large abstractions until repeated need is proven.

Do not add unnecessary dependencies.

Prefer boring, understandable code.

## 26. Development Order

Follow this order:

**Step 1**

Create repo scaffold, package config, TypeScript config, README, and docs.

**Step 2**

Create initial Postgres schema.

**Step 3**

Create DBOS client and minimal workflow.

**Step 4**

Create Pi extension entry point.

**Step 5**

Add command registration.

**Step 6**

Add durable task/run creation.

**Step 7**

Add run inspection.

**Step 8**

Add tool call/artifact logging.

**Step 9**

Add approval model.

**Step 10**

Add minimal TUI/status output.

**Step 11**

Add tests.

**Step 12**

Document what works, what does not, and what requires Pi API changes.

## 27. Initial README Content

The README should explain:

what the extension does
what problem it solves
how to install locally
how to configure Postgres
how to run tests
how to start a durable Pi run
current limitations

## 28. Product Philosophy

This project must remain minimal.

Before adding any feature, ask:

Is this required for durable, auditable, safe agentic engineering?
Has the need appeared more than once?
Can it be represented as a task, run, step, artifact, approval, or validation?
Can it be added as an extension without changing Pi core?

If the answer is no, do not build it yet.

## 29. Definition of Done for Initial Version

The first version is done when:

A developer can create a task from Pi.
The task starts a DBOS-backed run.
The run records steps.
The run records at least simulated tool calls.
The run records artifacts.
The run can request approval.
The run status can be inspected.
The database contains enough state to audit what happened.
The project runs locally without cloud dependencies.

## 30. Final Direction

Build a Pi-native durable kernel.

Do not build a standalone DBOS app.

Do not build a dashboard before the execution loop works.

Do not fork Pi unless absolutely necessary.

The desired outcome is:

Pi remains the adaptive agentic builder.
DBOS provides durable execution.
Postgres stores auditable state.
The extension provides governance, visibility, and controlled self-improvement.

---

## 31. Phase 2: Controlled Self-Adaptation and Engineer Cockpit

### 31.1 Phase 2 Intent

Phase 2 begins only after Phase 1 proves the durable execution loop works.
The intent is to let the system start improving itself safely.

Phase 2 should add:
1. Real Pi integration beyond simulated tool calls
2. Human approval workflow
3. Minimal engineer cockpit
4. Change proposal artifacts
5. Validation before applying changes
6. Safer self-modification boundaries
7. Repeatable workflow templates

Do not build a full visual workflow editor yet.

---

### 31.2 Phase 2 Acceptance Criteria

Phase 2 is complete when:

```text
A developer can request an improvement to the extension.
Pi generates a proposed change.
The change is stored as an artifact.
The system classifies the risk.
The system runs validation.
The system requests approval if required.
The developer can approve or reject.
Approved changes can be applied manually or semi-automatically.
All steps are logged in Postgres.
The cockpit shows task, run, artifact, validation, and approval state.
```

---

### 31.3 New Capabilities

Implement these capabilities in order:

1. Real Pi tool-call capture
2. Artifact diff capture
3. Approval workflow
4. Validation runner
5. Engineer cockpit
6. Improvement proposal flow
7. Workflow template promotion

---

### 31.4 New Commands

Add:

```
/dbos proposals
/dbos proposal inspect <proposal_id>
/dbos proposal approve <proposal_id>
/dbos proposal reject <proposal_id>
/dbos validation run <run_id>
/dbos artifacts <run_id>
/dbos retry <run_id>
/dbos cancel <run_id>
```

---

### 31.5 Engineer Cockpit

The cockpit should remain minimal.

Initial views:

Runs
Approvals
Artifacts
Validations
Proposals

Each view should support:

list
inspect
filter by status
open artifact path
approve/reject where relevant

Avoid charts, complex dashboards, and drag/drop workflow editing.

---

### 31.6 Improvement Proposal Flow

Self-adaptation must use proposals.

Flow:

```text
user requests platform improvement
   ↓
durable task created
   ↓
Pi analyzes repo
   ↓
Pi generates implementation proposal
   ↓
proposal stored as artifact
   ↓
risk classified
   ↓
validation plan generated
   ↓
approval requested
   ↓
human decides
```

An improvement proposal should include:

intent
affected files
risk level
implementation plan
expected behavior change
validation plan
rollback plan
generated diff or patch

---

### 31.7 New Tables

Add only if needed:

improvement_proposals
workflow_templates
file_reservations
policy_decisions

Do not add a large schema prematurely.

---

### 31.8 File Reservation

Before any code-writing task, reserve files.

Minimum fields:

id
run_id
path
reserved_by
reserved_at
expires_at
status

Rules:

A run may only edit files it reserved.
Reservations expire automatically.
Conflicting reservations block execution.
Platform files require approval before reservation.

---

### 31.9 Validation Runner

The validation runner should support:

npm test
npm run build
lint command if available
custom repo validation command
dry-run migration validation

Validation results must be stored.

Never treat generated code as complete without validation.

---

### 31.10 Policy Decisions

Every risky action should create a policy decision record.

Capture:

action
risk level
allowed/blocked/requires approval
reason
run_id
timestamp

This becomes the basis for auditability and later policy learning.

---

### 31.11 Workflow Templates

Only promote a workflow to a template after it repeats.

Initial template candidates:

documentation update
source code change
test fix
extension improvement
schema migration proposal
UI component update

Do not create templates before repeated need is observed.

---

### 31.12 Phase 2 Non-Goals

Do not build:

full no-code workflow builder
multi-tenant SaaS UI
cloud sync
marketplace
complex analytics
autonomous production self-modification
unrestricted shell execution

---

### 31.13 Definition of Done for Phase 2

Phase 2 is done when:

The system can propose improvements to itself.
All proposed changes are artifacts.
Risk is classified.
Validation is executed.
Approval is required for risky changes.
The engineer cockpit can inspect and resolve approvals.
The system remains runnable offline/on-prem.
The implementation does not require a Pi fork unless documented.

---

## 32. Phase 3: Workflow Intelligence and Visual Control

### 32.1 Phase 3 Intent

Phase 3 begins only after Phase 2 proves controlled self-adaptation works.
The intent is to make repeated agent workflows easier to understand, reuse, and control.

Phase 3 should add:
1. Visual workflow inspection
2. Workflow template editing
3. Run replay
4. Multi-agent task coordination
5. Policy-aware workflow execution
6. Better offline model support
7. More structured validation and rollback

Do not turn the product into a generic no-code automation platform.

---

### 32.2 Phase 3 Acceptance Criteria

Phase 3 is complete when:

```text
A developer can inspect a workflow visually.
A repeated workflow can be promoted into a reusable template.
A workflow template can be edited safely.
A prior run can be replayed or forked.
Multiple agents can coordinate on one task without file conflicts.
Policy decisions are visible at workflow-step level.
Rollback instructions are generated for approved changes.
The system still runs offline/on-prem.
```

---

### 32.3 New Capabilities

Implement in this order:

1. Workflow graph view
2. Template promotion
3. Template editing with validation
4. Run replay
5. Run fork
6. Multi-agent coordination
7. Rollback generation
8. Policy-aware workflow simulation

---

### 32.4 Visual Workflow View

The visual workflow must be inspection-first.

Show:

task
run
steps
status
agent responsible
tool calls
artifacts
validation results
approval gates
policy decisions

Allowed actions:

inspect step
inspect artifact
inspect policy decision
retry failed step
fork run
open approval

Do not add drag/drop editing first.

---

### 32.5 Workflow Template Editing

Template editing is allowed only after template promotion exists.

A template should include:

name
description
trigger type
required inputs
steps
required validations
approval rules
rollback expectations
risk profile

Every template change must create:

proposal
diff
validation plan
approval request

---

### 32.6 Run Replay and Forking

Replay means:

re-run the same workflow with the same inputs where safe

Fork means:

create a new run from an earlier run with modified input or strategy

Replay/fork must not reuse secrets, stale credentials, or unsafe filesystem assumptions.

---

### 32.7 Multi-Agent Coordination

Introduce multiple agents only when needed.

Initial roles:

planner
builder
validator
reviewer
documenter

Rules:

Only one agent may reserve a file for writing at a time.
Validator agents may inspect but not modify source files.
Reviewer agents may propose changes but not apply them.
Planner agents create tasks and plans, not code changes.
Builder agents create artifacts and diffs.

---

### 32.8 Rollback Generation

Every approved source-code or workflow change should include a rollback plan.

Rollback artifact should include:

changed files
commit or patch reference
database changes if any
manual rollback steps
known limitations

---

### 32.9 Policy-Aware Simulation

Before executing high-risk workflows, support simulation.

Simulation should answer:

Which files may be touched?
Which tools may be called?
Which approvals will be required?
Which validations will run?
What risks are expected?

Simulation does not execute destructive actions.

---

### 32.10 Offline Model Support

Add explicit support for local model configuration.

Minimum requirements:

local model endpoint
model capability profile
context window
tool-calling support flag
structured output reliability notes
default use cases
fallback behavior

Do not assume all local models can perform all roles.

---

### 32.11 Phase 3 Non-Goals

Do not build:

generic marketplace
public SaaS
complex billing
visual programming language
fully autonomous production deployment
unrestricted self-modification
large analytics dashboard

---

### 32.12 Definition of Done for Phase 3

Phase 3 is done when:

Repeated workflows are reusable.
Workflow execution is visually inspectable.
Templates can be safely edited.
Runs can be replayed or forked.
Multiple agents can coordinate safely.
Rollback planning exists.
Policy simulation exists for risky workflows.
Offline/on-prem operation remains intact.
