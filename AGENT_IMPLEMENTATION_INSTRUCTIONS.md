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

---

## 34. Cross-Phase Rule: Reuse Before Build

### 34.1 Intent

At every phase, the system must prefer:

```text
existing Pi extensions
existing Pi packages
existing open-source tools
existing DBOS capabilities
existing CLI tools
```

over building new components.

No new feature should be implemented without first checking if it already exists.

---

### 34.2 Mandatory Reuse Evaluation Step

Before implementing any feature, the agent must execute:

1. Define the required capability
2. Search for existing Pi extension or package
3. Search for existing open-source tool
4. Evaluate integration feasibility
5. Document decision
6. Only then implement if necessary

This must be documented in:

```
docs/reuse-evaluation/<feature-name>.md
```

---

### 34.3 Reuse Categories

The system should explicitly evaluate reuse in these categories:

Pi extensions
Pi packages
DBOS features
CLI tools (git, npm, etc.)
validation frameworks
UI components
agent orchestration tools
policy engines

---

### 34.4 Integration Preference Order

Always prefer:

1. Native Pi extension reuse
2. Pi package reuse
3. Thin wrapper around external tool
4. Direct integration
5. Custom implementation (last option)

---

### 34.5 Extension vs Build Decision Rules

Build only if:

no existing extension exists
existing extension cannot be adapted
integration cost exceeds build cost
control or governance requires custom implementation
offline requirement cannot be met otherwise

---

### 34.6 Extension Wrapping Pattern

When reusing external tools:

Do not expose raw tool.
Wrap it as a Pi tool or command.
Add governance checks.
Log usage in DBOS.
Attach artifacts to runs.

Example:

```text
External test runner
   ↓
Wrapped as Pi tool
   ↓
Executed via DBOS step
   ↓
Results stored as validation_results
```

---

### 34.7 Phase-Specific Reuse Guidance

**Phase 1**

Reuse:

DBOS workflow primitives
Pi command registration
basic CLI tooling

Avoid building:

custom workflow engine
custom scheduler
custom logging framework

---

**Phase 2**

Reuse:

existing validation tools (npm test, lint)
existing diff tools
existing Pi UI components

Avoid building:

custom test framework
custom diff engine
custom logging UI

---

**Phase 3**

Reuse:

graph visualization libraries
existing Pi UI components
existing multi-agent coordination patterns

Avoid building:

custom graph engine
custom rendering engine
complex UI framework from scratch

---

**Phase 4**

Reuse:

existing orchestration tools (if needed)
existing monitoring tools
existing policy engines if compatible

Avoid building:

distributed systems platform
custom metrics system
custom scheduler

---

### 34.8 Reuse Validation Requirement

Every implemented feature must include:

why reuse was not sufficient
what alternatives were evaluated
what trade-offs were considered

If this is missing, the implementation is incomplete.

---

### 34.9 Reusable Extension Output Requirement

All new functionality must be designed as:

reusable Pi extension
or reusable module within this extension

No feature should be tightly coupled to a single workflow.

---

### 34.10 Long-Term Goal

Over time, the system should:

discover reusable patterns
promote them into templates
package them as extensions
reuse them across tasks

The platform should evolve into:

```text
a composition of reusable extensions and workflows
```

not a monolithic system.

---

### 34.11 Architectural Implication

With this rule in place, the platform becomes:

```text
Not: a system that builds everything itself
But: a system that intelligently composes, wraps, and governs existing capabilities
```

Without this rule: you build another framework.

With this rule: you build an adaptive integration and governance layer over the ecosystem.

That is the correct interpretation of the core principles:

- "if it is not needed, it will not be built"
- "self-adapting/building means composing, not replacing"

---

## 35. Phase 5: Meta-Platform and Extension Ecosystem

### 35.1 Phase 5 Intent

Phase 5 begins only after Phase 4 proves:
- Autonomous orchestration is safe and bounded
- Continuous improvement loops are stable
- Governance and auditability are reliable at scale
- Reuse-first principles are consistently applied

The intent of Phase 5 is to evolve the system into a **meta-platform**:

```text
A platform that builds, evaluates, composes, and governs extensions and agentic capabilities
```

This phase introduces:

1. Extension discovery and composition
2. Standardized extension contracts
3. Extension packaging and reuse across environments
4. Agent-driven extension evaluation
5. Cross-platform integration (Pi + others)
6. Ecosystem-level governance
7. Reusable "capability graph" of the platform

This is where the system transitions from a platform to a platform that builds platforms.

---

### 35.2 Phase 5 Acceptance Criteria

Phase 5 is complete when:

The system can discover and evaluate reusable extensions.
Extensions follow a defined contract.
Extensions can be installed, enabled, disabled, and versioned.
The system can propose replacing custom logic with reusable extensions.
Extension composition is tracked and auditable.
The system maintains a capability graph of what it can do.
Cross-platform agent integration is possible (not only Pi).
The system remains fully operable offline/on-prem.

---

### 35.3 Extension Contract

Define a standard contract for all extensions.

Minimum requirements:

name
version
capabilities
inputs
outputs
risk profile
required permissions
supported environments (local/cloud)
validation requirements
rollback support

Every extension must declare:

what it does
what it touches
what risk it introduces
how it can be validated
how it can be rolled back

---

### 35.4 Extension Lifecycle

Each extension must support:

install
enable
disable
upgrade
downgrade
remove

Each action must:

be tracked in DBOS
create audit records
respect policy
support rollback

---

### 35.5 Capability Graph

Introduce a capability graph representing:

what the platform can do
which extension provides each capability
which workflows depend on which capabilities
which risks are associated with each capability

Example:

```text
"generate code"      → Pi agent
"durable execution"  → DBOS extension
"validation"         → test runner extension
"policy enforcement" → governance extension
```

This graph enables:

reuse decisions
dependency tracking
impact analysis
self-improvement planning

---

### 35.6 Extension Discovery

The system should discover extensions from:

local repositories
configured extension directories
approved remote sources (optional)
internal extension registry

Discovery must not auto-install.

All discovered extensions must be:

classified
validated
risk-assessed
approved before use

---

### 35.7 Extension Evaluation

Before using a new extension, the system must evaluate:

does it already solve the problem?
is it compatible with Pi?
is it compatible with DBOS workflows?
does it meet offline requirements?
what risks does it introduce?
what permissions does it require?

This evaluation must be stored as an artifact.

---

### 35.8 Extension Composition

The system should compose capabilities:

```text
task → workflow → capabilities → extensions
```

Example:

```text
"update documentation"
   ↓
workflow template
   ↓
uses:
   - Pi agent
   - file system tool
   - validation tool
   - approval system
```

Composition must be:

explicit
traceable
auditable

---

### 35.9 Replacement Strategy

The system should detect:

custom code that duplicates existing extension capability

Then propose:

replace custom implementation with extension

Flow:

```text
detect duplication
   ↓
identify candidate extension
   ↓
generate proposal
   ↓
validate compatibility
   ↓
request approval
   ↓
apply replacement
```

---

### 35.10 Cross-Platform Integration

Phase 5 should allow integration beyond Pi.

Examples:

other agent frameworks
CLI-based tools
local model runtimes
external orchestration tools

Requirement:

All integrations must be wrapped as extensions.
No direct uncontrolled integration is allowed.

---

### 35.11 Ecosystem Governance

Introduce governance at extension level:

which extensions are allowed
which versions are approved
which capabilities are restricted
which environments allow which extensions

Policies must support:

allowlist
denylist
version constraints
risk-based restrictions
environment-based rules

---

### 35.12 Phase 5 Non-Goals

Do not build:

public marketplace platform
uncontrolled plugin ecosystem
auto-installation of external extensions
opaque dependency chains
fully autonomous extension upgrades

---

### 35.13 Definition of Done for Phase 5

Phase 5 is done when:

Extensions are first-class entities.
The system can discover, evaluate, and compose extensions.
Capabilities are mapped and tracked.
Custom logic can be replaced by reusable extensions.
Extension lifecycle is managed and auditable.
Cross-platform integration works through extensions.
The system remains minimal, controlled, and offline-capable.

---

### Final System State (End of Phase 5)

```text
Pi              → adaptive agent layer
DBOS            → durable execution layer
Postgres        → system state and audit
Extension system → capability composition layer
Policy engine   → governance layer
UI/CLI          → control and visibility layer
```

The platform becomes:

```text
A governed, extensible, self-improving agentic engineering system
built from reusable capabilities rather than monolithic code
```

---

## 36. Phase 6: Multi-Environment Federation and Deterministic Trust Layer

### 36.1 Phase 6 Intent

Phase 6 begins only after Phase 5 proves:
- Extensions are composable and governed
- Capability graph is reliable
- Autonomy is bounded and auditable
- Cross-project orchestration works

The intent of Phase 6 is to evolve the platform into a **federated, trustable system**:

```text
Multiple independent deployments (local, on-prem, private cloud)
that can interoperate, share capabilities, and exchange verifiable outcomes
without losing control, privacy, or auditability
```

This phase introduces:

1. Multi-environment federation (local ↔ local, local ↔ private cloud)
2. Deterministic execution and replay guarantees
3. Cryptographic audit and provenance (decision receipts)
4. Secure capability sharing between environments
5. Policy synchronization with local override
6. Trust boundaries between agents and environments
7. Reproducible builds and runs

---

### 36.2 Phase 6 Acceptance Criteria

Phase 6 is complete when:

Two independent deployments can exchange tasks or artifacts securely.
A run executed in one environment can be verified or replayed in another.
All critical actions have verifiable provenance.
Policies can be synchronized but overridden locally.
Capabilities can be shared without exposing full systems.
Trust boundaries are explicit and enforced.
The system remains fully operable offline/on-prem.

---

### 36.3 Federation Model

Each deployment is an independent node:

```text
Node A (local)
Node B (on-prem)
Node C (private cloud)
```

Nodes may:

exchange artifacts
exchange proposals
request validation from another node
share approved extensions
verify runs from other nodes

Nodes must not:

implicitly trust each other
execute remote instructions without validation
share secrets by default

---

### 36.4 Deterministic Execution

Introduce deterministic constraints:

```text
same input + same environment + same versioned extensions
   → must produce reproducible result (within tolerance)
```

Store:

inputs
model/config version
extension versions
workflow definition
policy snapshot
environment metadata

This enables:

run replay
cross-node verification
audit compliance

---

### 36.5 Provenance and Decision Receipts

Every critical action must generate a decision receipt.

A receipt includes:

what action was taken
why it was taken (intent + reasoning summary)
which inputs were used
which tools/extensions were involved
which policies were evaluated
who/what approved it
timestamp
hash/signature

Receipts must be:

immutable
traceable
verifiable across nodes

---

### 36.6 Trust Boundaries

Define explicit trust zones:

local trusted
local restricted
external trusted
external untrusted

Rules:

untrusted inputs must be validated
external artifacts must be verified
external workflows must not execute directly
policy must gate all cross-boundary actions

---

### 36.7 Capability Sharing

Nodes may share capabilities via extensions.

Shared capability must include:

extension package
version
capability definition
risk profile
validation requirements
signature or checksum

Receiving node must:

evaluate
validate
approve before enabling

---

### 36.8 Policy Synchronization

Support:

global policy templates
local overrides
environment-specific rules

Example:

```text
global:         high-risk requires approval
local dev:      allow medium-risk auto-run
production:     require approval for all writes
```

Policy changes must be:

versioned
audited
reviewable

---

### 36.9 Cross-Node Workflows

Allow workflows such as:

```text
Node A generates artifact
   ↓
Node B validates artifact
   ↓
Node C reviews and approves
```

Requirements:

each step logged locally
receipts exchanged
no implicit trust
full traceability preserved

---

### 36.10 Secure Communication

All inter-node communication must be:

authenticated
authorized
encrypted
logged

Do not implement full enterprise security stack initially.

Start with:

signed messages
simple key-based identity
explicit allowlists

---

### 36.11 Reproducibility and Environment Capture

Each run must capture:

OS and runtime info
Node version
extension versions
model versions
configuration snapshot

This enables:

debugging
audit
cross-node replay
compliance reporting

---

### 36.12 Phase 6 Non-Goals

Do not build:

global always-on distributed system
complex multi-cloud orchestration
automatic trust between nodes
fully decentralized autonomous system
heavy blockchain-based infrastructure

---

### 36.13 Definition of Done for Phase 6

Phase 6 is done when:

Multiple deployments can interoperate securely.
Runs are reproducible and verifiable.
Provenance is captured for critical actions.
Policies can be shared and overridden.
Capabilities can be exchanged safely.
Trust boundaries are enforced.
The system remains minimal and offline-capable.

---

### Final Evolution State (End of Phase 6)

```text
Local Node (Pi + DBOS + Extensions)
   ↕
Federation Layer (secure exchange)
   ↕
Other Nodes (independent systems)
```

The platform becomes:

```text
A federated, deterministic, and auditable agentic engineering system
capable of safe collaboration across environments without losing control
```

---

## 37. Phase 7: Self-Evolving Agentic Systems with Controlled Autonomy

### 37.1 Phase 7 Intent

Phase 7 begins only after Phase 6 proves:
- Federation is secure and deterministic
- Provenance and audit are reliable
- Policies enforce trust boundaries across environments
- Extension ecosystem is stable and governed

The intent of Phase 7 is to enable **controlled, self-evolving systems**:

```text
Systems that can redesign parts of themselves,
optimize their own architecture,
and adapt to new requirements,
while remaining fully governed, auditable, and reversible
```

This phase introduces:

1. Architecture-level self-evolution
2. System-wide optimization loops
3. Adaptive policy refinement (bounded)
4. Cross-node learning (without data leakage)
5. Capability-level evolution
6. Long-term memory of system performance
7. Safe experimentation frameworks

---

### 37.2 Phase 7 Acceptance Criteria

Phase 7 is complete when:

The system can propose structural changes to its own architecture.
Architectural changes follow proposal → validation → approval → apply flow.
The system can optimize workflows based on historical performance.
Policy refinement is possible but controlled and auditable.
Learning can be shared across nodes without exposing sensitive data.
Experiments can be run safely and rolled back.
All changes remain reproducible and traceable.

---

### 37.3 Architecture-Level Self-Evolution

The system may propose changes to:

extension structure
workflow templates
agent roles and coordination
policy configurations
validation strategies
UI interaction patterns

Restrictions:

No direct application without validation and approval.
All changes must include rollback plan.
All changes must be versioned.

---

### 37.4 Optimization Loop

Introduce system-wide optimization:

```text
observe performance
   ↓
detect inefficiencies
   ↓
generate optimization proposal
   ↓
simulate impact
   ↓
validate
   ↓
approve
   ↓
apply
   ↓
measure outcome
```

Optimization targets:

execution time
failure rates
validation success
approval frequency
resource usage
developer friction

---

### 37.5 Adaptive Policy Refinement

Policies may evolve based on:

historical approvals
failure patterns
risk outcomes
environment constraints

Rules:

Policy changes must always require approval.
Policy changes must be explainable.
Policy rollback must be available.
Critical safety rules must never be auto-relaxed.

---

### 37.6 Cross-Node Learning

Nodes may exchange:

anonymized performance metrics
validated workflow templates
approved extensions
optimization strategies

Nodes must not exchange:

raw sensitive data
secrets
private artifacts without approval

---

### 37.7 Capability Evolution

The system can evolve its capabilities:

replace extensions
combine capabilities
deprecate unused components
introduce improved implementations

Each change must follow:

evaluation
validation
approval
deployment
monitoring

---

### 37.8 Experimentation Framework

Introduce safe experimentation:

```text
run experiment in isolated context
compare against baseline
evaluate results
promote if successful
rollback if not
```

Experiments must:

be isolated from production runs
be fully logged
have clear success criteria
be reversible

---

### 37.9 Long-Term Memory

Store structured historical data:

run outcomes
approval decisions
validation failures
optimization results
policy changes
extension performance

Use this for:

trend detection
decision support
future optimization proposals

Do not create opaque or untraceable learning systems.

---

### 37.10 Human Oversight Model

Humans remain:

final authority for high-risk changes
policy approvers
system boundary definers

The system may assist but not replace human judgment in critical decisions.

---

### 37.11 Phase 7 Non-Goals

Do not build:

fully autonomous system without oversight
self-modifying system without audit
opaque learning mechanisms
unbounded policy changes
uncontrolled cross-node learning

---

### 37.12 Definition of Done for Phase 7

Phase 7 is done when:

The system can propose and apply architectural improvements safely.
Optimization loops operate with measurable benefit.
Policy refinement is controlled and auditable.
Experiments are isolated and reversible.
Cross-node learning improves system behavior without data leakage.
All changes remain traceable and reproducible.

---

### Final Evolution State (End of Phase 7)

```text
Federated Nodes
   ↓
Shared Knowledge (bounded)
   ↓
Self-Optimizing Systems
   ↓
Governed Evolution
```

The platform becomes:

```text
A governed, self-evolving agentic engineering ecosystem
capable of continuous improvement without losing control, trust, or auditability
```

---

## 38. Phase 8: Autonomous Business Systems and Meta-Agent Fabric

### 38.1 Phase 8 Intent

Phase 8 begins only after Phase 7 proves:
- Self-evolution is safe, controlled, and auditable
- Optimization loops produce measurable improvements
- Federation and trust layers are stable
- Extension ecosystem is mature and governed

The intent of Phase 8 is to extend the platform beyond engineering into:

```text
Fully governed, AI-operated systems capable of running complete business or operational domains
```

This is where the platform evolves into a meta-agent fabric:

```text
A system that can design, deploy, operate, and improve entire agentic systems (including businesses)
within strict governance, auditability, and safety constraints
```

---

### 38.2 Phase 8 Acceptance Criteria

Phase 8 is complete when:

The system can define and execute full domain-level workflows (not just engineering tasks).
Multiple coordinated agent systems can operate continuously.
Business-level processes can be modeled as workflows.
The system can manage its own operational lifecycle (within policy).
All actions remain auditable and governed.
Autonomy is bounded and configurable per domain.
Systems can be deployed, replicated, and maintained automatically.

---

### 38.3 Domain-Level Workflows

Extend from engineering workflows to domain workflows:

compliance management
operations automation
content generation pipelines
customer interaction flows
monitoring and remediation systems
training and knowledge generation

Each domain workflow must still map to:

```text
task → run → steps → artifacts → validation → approval
```

---

### 38.4 Meta-Agent Fabric

Introduce coordinated agent systems:

```text
agent system A → engineering
agent system B → operations
agent system C → compliance
agent system D → monitoring
```

These systems must:

coordinate via DBOS workflows
share artifacts via controlled channels
respect policy boundaries
remain independently governable

---

### 38.5 Business-Level Autonomy

Autonomy expands but remains bounded.

Examples:

auto-generate compliance documentation
maintain system configuration
optimize workflows
respond to detected issues
propose product or system improvements

Constraints:

high-risk actions require approval
financial or legal decisions always require approval
external communication requires policy checks

---

### 38.6 Deployment Templates

Introduce system templates:

"AI compliance system"
"AI DevOps system"
"AI content pipeline"
"AI monitoring system"

Templates define:

agents
workflows
extensions
policies
validation rules
deployment configuration

Templates must be:

versioned
auditable
reproducible

---

### 38.7 System Lifecycle Management

The platform must manage:

deploy system
monitor system
update system
optimize system
decommission system

All lifecycle actions must:

be logged
be reversible
respect policy
produce artifacts

---

### 38.8 Continuous Operation

Support long-running systems:

24/7 workflows
scheduled operations
event-driven triggers
reactive remediation

Requirements:

checkpointing
safe restart
bounded resource usage
failure isolation

---

### 38.9 Multi-System Coordination

Allow multiple systems to interact:

```text
System A produces artifact
System B validates
System C reports
System D improves
```

Rules:

no implicit trust
all exchanges logged
policies enforced at boundaries

---

### 38.10 Observability Across Systems

Expose:

system-level performance
workflow success rates
resource usage
failure patterns
policy violations
improvement trends

Keep implementation minimal:

CLI + simple views first
exportable logs
no heavy dashboards initially

---

### 38.11 Safety and Governance

Strict constraints remain:

no unrestricted autonomy
no opaque decision-making
no unapproved external impact
no silent failures

System must always:

log
validate
request approval when required
allow rollback

---

### 38.12 Phase 8 Non-Goals

Do not build:

fully autonomous business with no human oversight
financial automation without controls
legal decision-making systems
unbounded agent ecosystems
complex SaaS marketplace

---

### 38.13 Definition of Done for Phase 8

Phase 8 is done when:

The platform can define and run full domain systems.
Multiple agent systems can coordinate safely.
Deployment templates exist and are reusable.
System lifecycle is managed through workflows.
Autonomy is bounded and governed.
All actions remain auditable and reproducible.
Offline/on-prem operation remains intact.

---

### Final Evolution State (End of Phase 8)

```text
Meta-Agent Fabric
   ↓
Domain Systems (engineering, compliance, operations)
   ↓
Coordinated Agent Systems
   ↓
Governed Execution (DBOS + policy)
   ↓
Auditable State (Postgres)
```

The platform becomes:

```text
A governed, extensible, self-operating agentic system platform
capable of running entire domains while preserving control, auditability, and trust
```

---

## 39. Phase 9: Economic Autonomy and External Interaction Layer

### 39.1 Phase 9 Intent

Phase 9 begins only after Phase 8 proves:
- Domain-level systems operate reliably
- Meta-agent fabric is stable
- Governance, auditability, and policy enforcement are consistent
- Multi-system coordination works safely

The intent of Phase 9 is to enable **controlled external interaction and economic activity**:

```text
Systems that can interact with external entities (users, services, markets)
and perform value-generating activities
while remaining fully governed, auditable, and bounded
```

This phase introduces:

1. External interaction layer (APIs, messaging, integrations)
2. Economic action capability (bounded)
3. Contract-based interactions
4. External identity and trust handling
5. Revenue/service workflows
6. Risk-aware external execution
7. External auditability and traceability

---

### 39.2 Phase 9 Acceptance Criteria

Phase 9 is complete when:

The system can safely interact with external systems via defined interfaces.
External actions are governed by policy and risk classification.
All external interactions are logged and auditable.
Economic or value-generating workflows can be defined and executed.
Contracts or agreements can be modeled and enforced.
External identity and trust are validated.
All actions remain reversible where possible.
Offline/on-prem operation remains intact for core system.

---

### 39.3 External Interaction Layer

Introduce controlled interfaces:

HTTP APIs
webhooks
message queues
file exchange
email or notification systems (optional)

All external interactions must:

be wrapped as extensions
be logged as artifacts or tool calls
pass policy checks
be traceable to a task/run

---

### 39.4 Economic Actions

Enable bounded economic behavior:

offer services (e.g. compliance analysis, report generation)
trigger billing or usage tracking (conceptual, not full billing system)
manage subscriptions or access rules (minimal)

Constraints:

no autonomous financial decisions without approval
no irreversible financial actions
clear policy gating

---

### 39.5 Contract-Based Workflows

Model interactions as contracts:

input expectations
output guarantees
validation criteria
approval requirements
risk classification

Contracts must be:

versioned
auditable
enforceable via validation

---

### 39.6 External Identity and Trust

Support:

external user identities
API keys or tokens
signed requests
trusted partner nodes

Rules:

no implicit trust
all identities must be validated
permissions must be enforced

---

### 39.7 Service Workflows

Define workflows such as:

```text
user submits request
   ↓
system validates input
   ↓
system executes workflow
   ↓
system generates artifact
   ↓
validation
   ↓
optional approval
   ↓
result delivered
```

---

### 39.8 Risk-Aware External Execution

Every external action must be classified:

low risk: read-only responses
medium risk: content generation
high risk: system changes
critical risk: financial/legal impact

Rules:

high and critical require approval
critical must be restricted by default

---

### 39.9 External Observability

Expose:

request volume
success/failure rates
external errors
policy violations
latency

Keep implementation minimal:

CLI outputs
log exports
simple summaries

---

### 39.10 Audit and Traceability

All external interactions must link to:

task
run
step
artifact
policy decision
approval (if any)

This ensures full traceability from request to outcome.

---

### 39.11 Phase 9 Non-Goals

Do not build:

full payment systems
unrestricted financial automation
autonomous legal systems
complex SaaS billing platform
unbounded external integrations

---

### 39.12 Definition of Done for Phase 9

Phase 9 is done when:

External interactions are safe and governed.
Economic workflows are possible but bounded.
Contracts are enforceable via validation.
Identity and trust are handled securely.
All actions remain auditable and reversible where possible.
The system remains minimal and offline-capable at core.

---

### Final Evolution State (End of Phase 9)

```text
Meta-Agent Fabric
   ↓
Domain Systems
   ↓
External Interaction Layer
   ↓
Governed Economic Activity
   ↓
Auditable Execution (DBOS + policy)
```

The platform becomes:

```text
A governed, extensible agentic system platform capable of interacting with the outside world
and generating value, while maintaining strict control, auditability, and safety
```

---

## 40. Phase 10: Standardization, Interoperability, and Ecosystem Alignment

### 40.1 Phase 10 Intent

Phase 10 begins only after Phase 9 proves:
- External interactions are safe and governed
- Economic workflows operate within policy
- Contracts and identity handling are reliable
- Federation and extension ecosystem are stable

The intent of Phase 10 is to transition from a powerful system to a **standardized, interoperable platform**:

```text
A system that can interoperate with other platforms, tools, and ecosystems
through well-defined, open, and enforceable standards
```

This phase introduces:

1. Standardized interfaces for tasks, runs, artifacts, and policies
2. Interoperability across agent frameworks and tools
3. Portable workflows and templates
4. Open extension and capability contracts
5. External validation and certification readiness
6. Vendor-neutral architecture alignment
7. Long-term ecosystem compatibility

---

### 40.2 Phase 10 Acceptance Criteria

Phase 10 is complete when:

Workflows can be exported and imported across environments.
Extensions follow a standardized, documented contract.
The system can interoperate with multiple agent frameworks.
Artifacts and results are portable and verifiable.
Policies can be represented in a standard format.
External systems can validate outputs.
The system remains vendor-neutral and offline-capable.

---

### 40.3 Standardized Data Models

Define canonical models for:

task
run
step
artifact
approval
validation result
policy decision
extension
capability

Requirements:

JSON-based representation
versioned schemas
backward compatibility where possible
clear semantics

---

### 40.4 Workflow Portability

Enable workflows to be:

exported
imported
versioned
validated across systems

A portable workflow must include:

steps
required capabilities
inputs and outputs
validation requirements
policy constraints
environment assumptions

---

### 40.5 Extension Interoperability

Extensions must be:

self-describing
versioned
compatible with standard contracts
portable between nodes

Support:

Pi-native extensions
wrapped external tools
cross-framework adapters

---

### 40.6 Cross-Framework Integration

Allow integration with:

other agent frameworks
workflow engines
external orchestration tools
local and cloud model runtimes

Rules:

all integrations must be wrapped as extensions
must follow governance rules
must expose standard interfaces

---

### 40.7 Policy Standardization

Represent policies in a structured, portable format.

Policy must include:

rules
risk classifications
conditions
actions
approval requirements

Policies must be:

versioned
auditable
exportable
importable

---

### 40.8 External Validation and Certification

Support validation by external systems:

audit tools
compliance frameworks
third-party validators

Provide:

traceable logs
decision receipts
reproducible runs
clear data lineage

---

### 40.9 Vendor-Neutral Design

Ensure:

no dependency on a single vendor
replaceable components (LLM, DB, orchestration)
configurable integrations

All components should be:

swappable
abstracted
documented

---

### 40.10 Ecosystem Alignment

Align with:

open standards where applicable
industry best practices
emerging agent protocols
data exchange standards

Do not implement speculative or unstable standards prematurely.

---

### 40.11 Phase 10 Non-Goals

Do not build:

complex standards organization
overly rigid schemas
heavy compliance frameworks beyond need
vendor lock-in features
unnecessary abstraction layers

---

### 40.12 Definition of Done for Phase 10

Phase 10 is done when:

The system can interoperate with other platforms.
Workflows and extensions are portable.
Policies are standardized and portable.
External validation is possible.
The platform remains minimal, flexible, and vendor-neutral.

---

### Final Evolution State (End of Phase 10)

```text
Standardized Core
   ↓
Portable Workflows
   ↓
Interoperable Extensions
   ↓
Federated Systems
   ↓
Governed Execution Layer
```

The platform becomes:

```text
A standardized, interoperable agentic system platform
capable of integrating into a broader ecosystem
while maintaining control, auditability, and flexibility
```
