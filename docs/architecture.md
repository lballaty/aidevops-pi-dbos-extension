# Architecture: What Pi + Temporal + Postgres Give You

## The One-Line Version

Pi does the thinking and acting.
Temporal makes sure it completes safely.
Postgres remembers everything.

---

## What You Are Actually Building

A system where you can give an AI agent a task and trust that:

- It will complete even if something crashes
- It will not do anything dangerous without your approval
- You will know exactly what it did and why
- You can prove it to an auditor
- You can stop it, inspect it, and resume it at any point
- It will not drift from what you asked it to do

That is the goal. Everything else is implementation detail.

---

## What You Should Get Out of Pi

Pi is your agent. It reads, writes, reasons, and acts.

### What Pi should give you

**1. Intent understanding**
You say what you want in plain language.
Pi figures out what needs to happen.

```text
"Add error handling to the payment workflow"
   ↓
Pi reads the file
Pi understands the problem
Pi proposes a solution
Pi implements it
```

**2. Tool execution**
Pi uses tools to interact with your system:
- read files
- write files
- run commands
- search code
- run tests

**3. Adaptive behavior**
If something fails, Pi adjusts.
If the first approach does not work, Pi tries another.
Pi does not need you to micromanage every step.

**4. Self-improvement**
Pi can be asked to improve this extension itself.
It can propose changes, generate diffs, and submit them for approval.

### What Pi should NOT do alone

- Make irreversible changes without a checkpoint
- Edit files without reserving them first
- Proceed past a risky action without approval
- Run without its work being recorded
- Operate outside a defined scope

Pi is powerful but unbounded by default.
The workflow is what bounds it.

---

## What You Should Get Out of Temporal

Temporal is your execution guarantee layer.

### What Temporal should give you

**1. Every agent task runs inside a workflow**

```text
Pi receives task
   ↓
Temporal workflow starts
   ↓
Pi's actions run as Temporal activities
   ↓
Every action is checkpointed
   ↓
Workflow completes or fails cleanly
```

No agent task runs outside a workflow.
If it is not in a workflow, it does not have durability, audit, or governance.

**2. Crash recovery**

If your server crashes mid-task, Temporal restarts the workflow
from the last completed activity. Pi picks up where it left off.
The user does not need to re-submit the task.

**3. Approval gates that actually work**

```text
Pi is about to edit source code (high risk)
   ↓
Workflow pauses
   ↓
Approval request written to Postgres
   ↓
You receive notification
   ↓
You approve or reject
   ↓
Temporal resumes or terminates the workflow
```

Without Temporal, you cannot reliably pause an agent mid-execution.
With Temporal, the pause is durable — it survives restarts.

**4. Enforced step sequence**

The workflow definition is the law.
Pi cannot skip the file reservation step.
Pi cannot apply a change before validation passes.
Pi cannot proceed past a failed approval.
The sequence is enforced structurally, not by hope.

**5. Retry without repeating completed work**

If an activity fails (network error, timeout, transient failure),
Temporal retries it automatically.
Activities that already completed are not re-run.
No duplicate writes, no double actions.

**6. Visibility into running workflows**

At any point you can ask:
- What workflows are currently running?
- What step is each workflow on?
- How long has it been running?
- Has it failed? Why?

This is live, not historical.

### What Temporal activities should map to

Every meaningful Pi action should be a Temporal activity:

| Pi action | Temporal activity |
|-----------|------------------|
| Read a file | `readFileActivity` |
| Write a file | `writeFileActivity` |
| Run tests | `runValidationActivity` |
| Request approval | `requestApprovalActivity` |
| Wait for approval | Workflow signal / wait condition |
| Record artifact | `recordArtifactActivity` |
| Reserve a file | `reserveFileActivity` |
| Release a file | `releaseFileActivity` |
| Complete task | `completeRunActivity` |

If Pi does something that is not an activity, it is invisible to the system.

---

## What You Should Get Out of Postgres

Postgres is your source of truth and your audit trail.

### What Postgres should give you

**1. The full history of every run**

Every task, every run, every step, every tool call, every artifact,
every approval request and decision — all in Postgres.

```sql
SELECT * FROM runs WHERE task_id = 'x' ORDER BY started_at;
SELECT * FROM tool_calls WHERE run_id = 'y' ORDER BY created_at;
SELECT * FROM approvals WHERE status = 'requested';
```

**2. Drift detection by query**

```sql
-- What did we expect vs what actually happened
SELECT
  s.step_name,
  s.status,
  vr.status as validation_status
FROM steps s
LEFT JOIN validation_results vr ON vr.run_id = s.run_id
WHERE s.run_id = 'x';
```

**3. Approval and governance state**

All pending approvals are in Postgres.
All policy decisions are in Postgres.
All risk classifications are in Postgres.
This is queryable, auditable, and exportable.

**4. Artifact tracking**

Every file Pi writes, every diff it produces, every report it generates
is recorded as an artifact with path, hash, and summary.
You can always answer: what did the agent produce and when?

**5. Compliance evidence**

When an auditor asks what happened on a given run,
you query Postgres and export the result.
No manual reconstruction. No guessing.

---

## What You Should Get Out of the Combination

This is the important part. The three together give you something
none of them provide alone.

### Safe agent execution

```text
You give Pi a task
   ↓
Temporal starts a workflow
   ↓
Pi reasons and acts inside that workflow
   ↓
Every action is a Temporal activity
   ↓
Every activity result is written to Postgres
   ↓
Risky actions pause for approval
   ↓
File writes are reserved before touching
   ↓
Validation runs before changes are accepted
   ↓
Everything completes or fails cleanly with full state preserved
```

### Drift detection at every step

```text
Before run:   define target state (what should be true after)
During run:   each activity records observed state
After run:    compare target vs observed
              if different → drift detected → classified → acted on
```

### A complete audit trail without extra work

Because every action is a Temporal activity and every activity writes to Postgres,
the audit trail is a natural output of execution.
You do not build audit logging separately.
It is a structural consequence of how the system runs.

### Self-improvement under governance

```text
Pi proposes an improvement to this extension
   ↓
Proposal stored as artifact in Postgres
   ↓
Risk classified
   ↓
Validation runs (build, tests, lint)
   ↓
Approval requested if high risk
   ↓
You approve
   ↓
Change applied
   ↓
Result logged
```

Pi can help build the system that governs Pi.
But every step is recorded and nothing critical happens without approval.

---

## The Outcomes You Should Be Able to Demonstrate

After building this system you should be able to answer yes to all of these:

### Reliability
- If the server crashes mid-task, does the agent resume where it left off?
- If an activity fails transiently, does it retry without repeating completed steps?

### Safety
- Can risky actions be paused for approval?
- Can you cancel a running workflow cleanly?
- Can files be protected from concurrent agent edits?

### Visibility
- Can you see all running agent tasks right now?
- Can you see every action an agent took in a completed run?
- Can you see what artifacts were produced?

### Governance
- Is every action classified by risk level?
- Do high-risk actions require approval before proceeding?
- Is every approval decision recorded with reason and timestamp?

### Auditability
- Can you export the full history of a run to show an auditor?
- Can you answer "what did the agent do between 2pm and 3pm yesterday"?
- Can you trace every artifact back to the run and intent that produced it?

### Drift
- Can you detect when a run deviated from its expected behavior?
- Can you detect when an agent edited a file it should not have touched?
- Can you detect when policy changed without going through approval?

### Self-improvement
- Can Pi propose improvements to the extension?
- Are those proposals governed (risk classified, validated, approved)?
- Is the system better after each accepted improvement?

---

## What This Is Not

This is not:

- A replacement for Pi (Pi is the agent, this wraps it)
- A replacement for Temporal (Temporal is the execution engine, this uses it)
- A workflow builder for non-technical users
- A cloud SaaS platform
- A monitoring dashboard

It is a **governance and durability layer** that makes Pi safe enough to trust
with real engineering work.

---

## Summary

| Layer | What it gives you |
|-------|------------------|
| Pi | The agent that does the work |
| Temporal | Durability, crash recovery, enforced step sequence, approval gates |
| Postgres | Audit trail, drift detection, governance state, compliance evidence |
| This extension | The glue: maps Pi actions to Temporal activities, writes state to Postgres, enforces governance |
