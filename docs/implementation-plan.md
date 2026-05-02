# Implementation Plan: Vertical Modules M1–M12

## How to Use This Document

Each module follows the same structure:

1. **Pre-conditions** — what must exist before work begins
2. **Write tests first** — all tests written and failing before any implementation
3. **Implement** — write only enough code to make each test pass
4. **Verify end-to-end** — run the manual acceptance command
5. **Stabilize** — previous modules must still pass before moving on

Test stack: **Jest** + local Postgres (via `DATABASE_URL` in `.env`).  
No cloud services. No mocks for Postgres — use a real local database.  
DBOS workflows are tested via the DBOS testing utilities.

---

## M1 — Durable Task Kernel

**Goal:** Create a task, run it through DBOS, store state, inspect result.

### Pre-conditions

- Postgres running locally
- `DATABASE_URL` set in `.env`
- `npm install` complete
- Migration `001_initial_schema.sql` applied

### Tests First

**File:** `tests/m1-durable-task-kernel.test.ts`

```
1. createTask()
   - stores a task record in Postgres
   - assigns status: pending
   - returns a task id

2. startRun(taskId)
   - creates a run record linked to the task
   - assigns status: created
   - stores a dbos_workflow_id
   - returns a run id

3. DBOS workflow execution
   - workflow starts when startRun is called
   - run status transitions: created → executing → completed
   - each status transition is persisted

4. getRun(runId)
   - returns current status
   - returns started_at and finished_at
   - returns linked task id

5. getTask(taskId)
   - returns task record with status
   - returns linked run ids

6. End-to-end acceptance
   - create task → start run → poll until completed → inspect status
   - all state visible in Postgres after workflow finishes
```

### Implementation Order (after tests are written)

1. `migrations/001_initial_schema.sql` — `tasks` and `runs` tables
2. `src/persistence/types.ts` — Task, Run, RunStatus types
3. `src/persistence/repositories.ts` — createTask, createRun, updateRunStatus, getTask, getRun
4. `src/dbos/client.ts` — DBOS app init, Postgres connection
5. `src/dbos/workflows.ts` — minimal workflow: start → executing → completed
6. `src/pi/commands.ts` — `/dbos task create` and `/dbos status` stubs
7. `src/index.ts` — extension entry point, registers commands

### Manual Verification

```bash
npm run dbos task create "my first task"
npm run dbos status <run-id>
```

Expected: task and run records visible in Postgres, status `completed`.

### Definition of Done

- All M1 tests pass
- Run record persists through process restart
- Status inspectable via CLI command

---

## M2 — Tool Call Logging

**Goal:** Capture tool calls during a run and store them.

### Pre-conditions

- M1 complete and passing
- `tool_calls` table in schema

### Tests First

**File:** `tests/m2-tool-call-logging.test.ts`

```
1. recordToolCall(runId, stepId, toolName, input)
   - stores tool call record linked to run
   - assigns status: started
   - returns tool call id

2. completeToolCall(toolCallId, output)
   - updates status: completed
   - stores output
   - stores risk_level: low (placeholder)

3. failToolCall(toolCallId, error)
   - updates status: failed
   - stores error in output field

4. listToolCalls(runId)
   - returns all tool calls for a run in order
   - includes input, output, status, risk_level

5. Risk level placeholder
   - all tool calls default to risk_level: low
   - field is present and queryable

6. End-to-end acceptance
   - create task → start run → simulate tool call → complete tool call
   - inspect run shows tool calls with input/output
```

### Implementation Order

1. Add `tool_calls` table to `001_initial_schema.sql` (or new migration)
2. Add ToolCall, ToolCallStatus types to `src/persistence/types.ts`
3. Add recordToolCall, completeToolCall, failToolCall, listToolCalls to `src/persistence/repositories.ts`
4. Add `record_tool_call` DBOS step wrapper in `src/dbos/workflows.ts`
5. Add `src/pi/tools.ts` — expose `record_tool_call` as Pi tool
6. Add `/dbos inspect <run-id>` command showing tool calls

### Manual Verification

```bash
npm run dbos task create "test tool logging"
# observe tool calls in output of:
npm run dbos inspect <run-id>
```

Expected: tool call records with input, output, and status visible.

### Definition of Done

- All M1 and M2 tests pass
- Tool calls visible under run inspect
- Risk level field present (value: low)

---

## M3 — Artifact Capture

**Goal:** Store generated files, diffs, or reports as run artifacts.

### Pre-conditions

- M1 and M2 complete and passing
- `artifacts` table in schema

### Tests First

**File:** `tests/m3-artifact-capture.test.ts`

```
1. recordArtifact(runId, type, path, summary)
   - stores artifact record linked to run
   - computes and stores content_hash from file at path
   - returns artifact id

2. recordArtifact with inline content (no file path)
   - stores artifact with null path
   - stores content_hash from provided content string

3. listArtifacts(runId)
   - returns all artifacts for a run
   - includes type, path, summary, content_hash, created_at

4. getArtifact(artifactId)
   - returns single artifact record
   - includes all fields

5. Artifact types
   - accepts: file, diff, report, patch, log
   - rejects unknown types with validation error

6. End-to-end acceptance
   - create task → start run → record artifact → list artifacts
   - artifact record visible with correct hash and summary
```

### Implementation Order

1. Add `artifacts` table to schema
2. Add Artifact, ArtifactType types to `src/persistence/types.ts`
3. Add recordArtifact, listArtifacts, getArtifact to `src/persistence/repositories.ts`
4. Add content hash utility (SHA-256 via Node `crypto`) to `src/persistence/repositories.ts`
5. Expose `record_artifact` as Pi tool in `src/pi/tools.ts`
6. Add artifact list to `/dbos inspect <run-id>` output

### Manual Verification

```bash
echo "test content" > /tmp/test-artifact.txt
npm run dbos artifact record <run-id> file /tmp/test-artifact.txt "test artifact"
npm run dbos inspect <run-id>
```

Expected: artifact record with path, hash, and summary visible under run.

### Definition of Done

- All M1–M3 tests pass
- Artifacts visible under run inspect
- Content hash computed and stored

---

## M4 — Approval Gate

**Goal:** Pause risky actions and require human approval before continuing.

### Pre-conditions

- M1–M3 complete and passing
- `approvals` table in schema
- Risk classifier exists (basic version)

### Tests First

**File:** `tests/m4-approval-gate.test.ts`

```
1. Risk classifier
   - classifyRisk("read_file") → low
   - classifyRisk("edit_source_code") → high
   - classifyRisk("delete_files") → critical
   - classifyRisk("edit_documentation") → medium

2. requestApproval(runId, type, reason)
   - creates approval record
   - assigns status: requested
   - run status transitions to: waiting_approval
   - returns approval id

3. approveApproval(approvalId, reason)
   - updates approval status: approved
   - stores resolved_at and decision_reason
   - run status returns to: executing

4. rejectApproval(approvalId, reason)
   - updates approval status: rejected
   - stores resolved_at and decision_reason
   - run status transitions to: failed

5. listPendingApprovals()
   - returns all approvals with status: requested

6. DBOS pause/resume
   - workflow pauses when approval is requested
   - workflow resumes when approval is resolved
   - state is not lost during pause

7. High-risk gate enforcement
   - high-risk tool call triggers automatic approval request
   - run cannot continue until approved

8. End-to-end acceptance
   - create task → start run → trigger high-risk action
   - run pauses → inspect shows pending approval
   - approve → run resumes → completes
```

### Implementation Order

1. Add `approvals` table to schema
2. Add Approval, ApprovalStatus, RiskLevel types to `src/persistence/types.ts`
3. Add `src/governance/risk-classifier.ts` — classifyRisk function
4. Add requestApproval, resolveApproval, listPendingApprovals to `src/persistence/repositories.ts`
5. Add DBOS workflow pause/resume behavior in `src/dbos/workflows.ts`
6. Add `src/governance/approvals.ts` — gate enforcement logic
7. Add `/dbos approvals`, `/dbos approve <id>`, `/dbos reject <id>` commands
8. Expose `request_approval` as Pi tool

### Manual Verification

```bash
npm run dbos task create "high risk task"
# run triggers approval gate automatically
npm run dbos approvals
npm run dbos approve <approval-id> "looks good"
npm run dbos status <run-id>
```

Expected: run completes after approval; full approval record with timestamps visible.

### Definition of Done

- All M1–M4 tests pass
- High-risk actions pause runs
- Approve/reject commands work
- Run state is not lost during pause

---

## M5 — Validation Runner

**Goal:** Run build/test/lint validation and store results.

### Pre-conditions

- M1–M4 complete and passing
- `validation_results` table in schema

### Tests First

**File:** `tests/m5-validation-runner.test.ts`

```
1. runValidation(runId, validatorName, command)
   - executes command in subprocess
   - captures stdout and stderr
   - stores result record with status: passed or failed
   - returns validation result id

2. Validation status
   - command exit 0 → status: passed
   - command exit non-zero → status: failed
   - output stored in details field

3. listValidationResults(runId)
   - returns all validation results for a run
   - includes validator_name, status, details, created_at

4. Built-in validators
   - npm_test: runs `npm test`
   - npm_build: runs `npm run build`
   - lint: runs lint command if available (skips if not found)

5. Validation timeout
   - commands timeout after configurable limit (default: 60s)
   - timeout stored as failed with details

6. End-to-end acceptance
   - create task → start run → run validation → inspect results
   - validation result visible under run with status and output
```

### Implementation Order

1. Add `validation_results` table to schema
2. Add ValidationResult, ValidatorName types to `src/persistence/types.ts`
3. Add `src/validation/command-validator.ts` — executes commands, captures output
4. Add recordValidationResult, listValidationResults to `src/persistence/repositories.ts`
5. Add built-in validator definitions (npm_test, npm_build, lint)
6. Add `/dbos validation run <run-id>` command
7. Expose `record_validation_result` as Pi tool

### Manual Verification

```bash
npm run dbos validation run <run-id> npm_test
npm run dbos inspect <run-id>
```

Expected: validation result with status (passed/failed) and full output visible under run.

### Definition of Done

- All M1–M5 tests pass
- Validation results stored and visible
- Built-in validators (npm_test, npm_build) work

---

## M6 — File Reservation

**Goal:** Prevent conflicting agent edits to the same files.

### Pre-conditions

- M1–M5 complete and passing
- `file_reservations` table in schema

### Tests First

**File:** `tests/m6-file-reservation.test.ts`

```
1. reserveFile(runId, path)
   - creates reservation record
   - assigns status: active
   - sets expires_at (default: 30 minutes)
   - returns reservation id

2. Conflict detection
   - reserveFile when path already has active reservation → throws conflict error
   - includes which run holds the reservation in error

3. releaseFile(reservationId)
   - updates status: released
   - allows another run to reserve the same path

4. Expiration
   - reservations past expires_at are treated as released
   - expired reservation does not block new reservation

5. listReservations(runId)
   - returns all active reservations for a run

6. Write scope enforcement
   - attempting to write to unreserved file → blocked with error
   - attempting to write to file reserved by another run → blocked

7. End-to-end acceptance
   - Run A reserves file → Run B fails to reserve same file
   - Run A releases → Run B succeeds
```

### Implementation Order

1. Add `file_reservations` table to schema
2. Add FileReservation, ReservationStatus types to `src/persistence/types.ts`
3. Add reserveFile, releaseFile, listReservations, checkReservation to `src/persistence/repositories.ts`
4. Add expiration query logic (WHERE expires_at > NOW() AND status = active)
5. Add write-scope check wrapper in `src/governance/policy.ts`
6. Add `/dbos reserve <run-id> <path>` and `/dbos release <reservation-id>` commands

### Manual Verification

```bash
npm run dbos reserve <run-a-id> src/index.ts
npm run dbos reserve <run-b-id> src/index.ts   # should fail with conflict
npm run dbos release <reservation-id>
npm run dbos reserve <run-b-id> src/index.ts   # should now succeed
```

Expected: conflict error on second reserve; success after release.

### Definition of Done

- All M1–M6 tests pass
- Reservation conflicts are enforced
- Expiration works correctly
- Write-scope check blocks unreserved writes

---

## M7 — Engineer Cockpit

**Goal:** Provide minimal user visibility across runs, approvals, artifacts, and validations.

### Pre-conditions

- M1–M6 complete and passing

### Tests First

**File:** `tests/m7-engineer-cockpit.test.ts`

```
1. getCockpitSummary()
   - returns active runs (status: executing or waiting_approval)
   - returns pending approvals
   - returns last 5 artifacts across all runs
   - returns last 5 validation results

2. Active runs view
   - includes run id, task title, status, started_at
   - excludes completed and failed runs

3. Pending approvals view
   - includes approval id, run id, type, reason, requested_at

4. Recent artifacts view
   - includes artifact id, run id, type, path, summary, created_at

5. Recent validations view
   - includes validation id, run id, validator_name, status, created_at

6. Inspect action from cockpit
   - cockpit can trigger inspect for any listed run

7. Approve/reject action from cockpit
   - cockpit can approve or reject any listed approval

8. End-to-end acceptance (manual)
   - open cockpit → see at least one active run and one pending approval
   - approve from cockpit → approval resolves, run resumes
```

### Implementation Order

1. Add `src/ui/cockpit.ts` — getCockpitSummary query (joins tasks, runs, approvals, artifacts, validation_results)
2. Add `src/ui/tui.ts` — CLI rendering of cockpit summary (plain text table format)
3. Add `/dbos cockpit` command that renders TUI output
4. Wire approve/reject actions directly from cockpit output
5. Add auto-refresh option (poll every N seconds)

### Manual Verification

```bash
npm run dbos cockpit
```

Expected: formatted output showing active runs, pending approvals, recent artifacts, and recent validations. Actions (inspect, approve, reject) available inline.

### Definition of Done

- All M1–M7 tests pass (cockpit query tests)
- Cockpit renders without error
- Approve/reject from cockpit works
- No separate dashboard required

---

## M8 — Improvement Proposal Flow

**Goal:** Allow the system to propose changes to itself safely.

### Pre-conditions

- M1–M7 complete and passing
- `improvement_proposals` table in schema

### Tests First

**File:** `tests/m8-improvement-proposal.test.ts`

```
1. createProposal(runId, intent, affectedFiles, riskLevel)
   - stores proposal record
   - assigns status: draft
   - links to run
   - returns proposal id

2. Proposal fields
   - intent
   - affected_files (array)
   - risk_level
   - implementation_plan
   - validation_plan
   - rollback_plan
   - diff (optional at creation)

3. promoteProposalToArtifact(proposalId)
   - creates artifact record linked to proposal
   - artifact type: diff or report
   - proposal status → awaiting_validation

4. validateProposal(proposalId)
   - runs validation plan steps
   - stores validation results
   - proposal status → awaiting_approval (if high risk) or approved

5. Risk gate for proposals
   - high or critical risk → approval required
   - low or medium risk → can auto-approve (if policy allows)

6. approveProposal / rejectProposal
   - delegates to approval gate (M4)
   - updates proposal status accordingly

7. listProposals(status)
   - returns proposals filtered by status

8. End-to-end acceptance
   - request improvement → proposal created → risk classified
   - high risk: approval requested → user approves → proposal approved
```

### Implementation Order

1. Add `improvement_proposals` table to schema
2. Add ImprovementProposal, ProposalStatus types to `src/persistence/types.ts`
3. Add createProposal, updateProposal, listProposals to `src/persistence/repositories.ts`
4. Add proposal → artifact promotion logic in `src/governance/approvals.ts`
5. Wire proposal validation to M5 validation runner
6. Wire proposal approval to M4 approval gate
7. Add `/dbos proposals`, `/dbos proposal inspect <id>`, `/dbos proposal approve <id>`, `/dbos proposal reject <id>` commands

### Manual Verification

```bash
npm run dbos task create "improve error handling in workflows.ts"
# system generates proposal during run
npm run dbos proposals
npm run dbos proposal inspect <proposal-id>
npm run dbos proposal approve <proposal-id> "plan looks correct"
```

Expected: proposal record with full fields, risk classification, and approval trail.

### Definition of Done

- All M1–M8 tests pass
- Proposals follow draft → awaiting_validation → awaiting_approval → approved flow
- High-risk proposals require approval
- Proposal artifact is created and linked

---

## M9 — Workflow Template Promotion

**Goal:** Promote repeated workflows into reusable templates.

### Pre-conditions

- M1–M8 complete and passing
- `workflow_templates` table in schema

### Tests First

**File:** `tests/m9-workflow-templates.test.ts`

```
1. promoteToTemplate(runId, name, description)
   - captures steps and metadata from completed run
   - creates workflow_templates record
   - assigns version: 1
   - returns template id

2. Template fields
   - name
   - description
   - steps (captured from run)
   - required_inputs
   - validation_requirements
   - risk_profile
   - version

3. createRunFromTemplate(templateId, inputs)
   - creates new task and run
   - instantiates steps from template definition
   - returns run id

4. Template versioning
   - updating a template creates new version, does not overwrite
   - previous version remains accessible

5. listTemplates()
   - returns all templates with name, version, description

6. End-to-end acceptance
   - complete run → promote to template → create new run from template
   - new run executes same steps as original
```

### Implementation Order

1. Add `workflow_templates` table to schema
2. Add WorkflowTemplate types to `src/persistence/types.ts`
3. Add promoteToTemplate, createRunFromTemplate, listTemplates to `src/persistence/repositories.ts`
4. Add step capture logic from completed runs in `src/dbos/workflows.ts`
5. Add template instantiation in `src/dbos/workflows.ts`
6. Add `/dbos templates`, `/dbos template promote <run-id>`, `/dbos template run <template-id>` commands

### Manual Verification

```bash
npm run dbos template promote <run-id> "documentation-update" "Updates project docs"
npm run dbos templates
npm run dbos template run <template-id>
npm run dbos status <new-run-id>
```

Expected: new run starts from template, same steps execute.

### Definition of Done

- All M1–M9 tests pass
- Template promotion captures steps correctly
- New run from template executes successfully
- Versioning works

---

## M10 — Visual Workflow Inspection

**Goal:** Inspect workflow execution as a structured graph.

### Pre-conditions

- M1–M9 complete and passing

### Tests First

**File:** `tests/m10-workflow-inspection.test.ts`

```
1. getRunGraph(runId)
   - returns ordered list of steps with status
   - each step includes: name, status, started_at, finished_at
   - each step links to: tool_calls, artifacts, validation_results, approvals

2. Step status representation
   - pending, running, completed, failed are all represented
   - failed steps include error message

3. Graph data completeness
   - all steps for a run are present
   - all linked entities are present per step

4. CLI graph rendering
   - renders steps as ordered text timeline
   - shows status symbol per step (✓ ✗ ⏳)
   - shows linked artifact count, tool call count per step

5. End-to-end acceptance (manual)
   - complete a multi-step run
   - inspect graph → all steps, artifacts, and approvals visible
```

### Implementation Order

1. Add getRunGraph query to `src/persistence/repositories.ts` (joins steps, tool_calls, artifacts, approvals, validation_results)
2. Add `src/ui/cockpit.ts` graph rendering (text-based timeline)
3. Add `/dbos graph <run-id>` command
4. Optional: export graph data as JSON for future web UI

### Manual Verification

```bash
npm run dbos graph <run-id>
```

Expected: timeline of steps with status, linked counts, and any failures highlighted.

### Definition of Done

- All M1–M10 tests pass
- Graph data query is complete and correct
- CLI rendering is readable
- JSON export available

---

## M11 — Multi-Agent Coordination

**Goal:** Support planner, builder, validator, and reviewer roles without file conflicts.

### Pre-conditions

- M1–M10 complete and passing
- M6 (file reservation) active

### Tests First

**File:** `tests/m11-multi-agent-coordination.test.ts`

```
1. Agent role assignment
   - assignRole(runId, role) stores role metadata on run
   - valid roles: planner, builder, validator, reviewer, documenter

2. Role permissions
   - planner: can create tasks and plans, cannot edit source files
   - builder: can reserve and edit files
   - validator: can read files, cannot modify
   - reviewer: can create proposals, cannot apply changes
   - documenter: can edit docs directory only

3. Permission enforcement
   - builder attempts to edit unreserved file → blocked
   - validator attempts to write file → blocked
   - reviewer attempts to apply change directly → blocked

4. Handoff records
   - recordHandoff(fromRunId, toRunId, description)
   - stores handoff in steps table with type: handoff
   - both runs linked in handoff record

5. Planner → Builder handoff
   - planner creates plan artifact
   - handoff triggers builder run
   - builder reserves files from plan

6. Builder → Validator handoff
   - builder completes artifact
   - handoff triggers validator run
   - validator reads but does not write

7. End-to-end acceptance
   - planner run creates plan → hands off to builder
   - builder reserves file, creates artifact → hands off to validator
   - validator validates without editing file
```

### Implementation Order

1. Add `agent_role` column to `runs` table
2. Add AgentRole type to `src/persistence/types.ts`
3. Add assignRole, recordHandoff to `src/persistence/repositories.ts`
4. Add role permission checks to `src/governance/policy.ts`
5. Wire permission checks into file reservation (M6) and tool call recording (M2)
6. Add `/dbos handoff <from-run-id> <to-run-id>` command

### Manual Verification

```bash
# start planner run
npm run dbos task create --role planner "plan error handling improvement"
# start builder run after planner completes
npm run dbos task create --role builder "implement error handling"
# validator run
npm run dbos task create --role validator "validate error handling changes"
npm run dbos graph <validator-run-id>
```

Expected: three runs with distinct roles, handoffs recorded, validator has no write artifacts.

### Definition of Done

- All M1–M11 tests pass
- Role permissions enforced
- Handoffs recorded and queryable
- No cross-role write violations

---

## M12 — Policy Simulation

**Goal:** Preview expected risks, approvals, affected files, and validations before execution.

### Pre-conditions

- M1–M11 complete and passing

### Tests First

**File:** `tests/m12-policy-simulation.test.ts`

```
1. simulateWorkflow(templateId or workflowDefinition)
   - returns expected steps
   - returns expected risk level per step
   - returns expected approvals required
   - returns expected validations
   - returns expected files affected
   - does NOT execute any real actions

2. Simulation accuracy
   - high-risk steps show approval_required: true
   - file writes show in affected_files
   - validations triggered by step type appear in expected_validations

3. Simulation result stored as artifact
   - result linked to task or template
   - includes all expected steps and decisions

4. Simulation vs execution comparison
   - after execution, compare actual steps to simulation
   - deviations are logged and classified

5. End-to-end acceptance (manual)
   - simulate template → review expected risks and approvals
   - execute template → compare to simulation output
```

### Implementation Order

1. Add `src/governance/policy.ts` simulation function — walks workflow definition, applies risk classifier, outputs expected decisions without executing
2. Add simulation result artifact type in `src/persistence/types.ts`
3. Store simulation results as artifacts via M3
4. Add post-execution comparison logic (actual steps vs simulation)
5. Add `/dbos simulate <template-id>` command
6. Surface deviations in `/dbos inspect <run-id>` output

### Manual Verification

```bash
npm run dbos simulate <template-id>
```

Expected: list of steps with expected risk, approval requirements, files, and validations — no actual execution.

### Definition of Done

- All M1–M12 tests pass
- Simulation produces accurate preview
- Simulation result stored as artifact
- Post-execution deviation comparison works

---

## Cross-Module Rules

1. **Each module's tests must pass before moving to the next module.**
2. **Previous module tests must still pass after each new module is added.**
3. **No module may be marked done without a passing manual verification command.**
4. **No new table or type may be added outside of the module that needs it.**
5. **Section 34 reuse evaluation is required before implementing any component within a module.**

---

## Milestone Summary

| Milestone | Modules | Phase |
|-----------|---------|-------|
| Core execution loop working | M1–M3 | Phase 1 |
| Governance and approval active | M4–M6 | Phase 2 (early) |
| Visibility and proposals working | M7–M8 | Phase 2 (late) |
| Workflow intelligence active | M9–M12 | Phase 3 |

After M12: system is fully governed, visible, and self-proposing.  
Subsequent modules (M13+) follow the same test-first vertical slice structure defined here.
