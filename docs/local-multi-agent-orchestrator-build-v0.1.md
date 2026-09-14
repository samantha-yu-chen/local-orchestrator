# Local Multi-Agent CLI Orchestrator

## Build Specification v0.1

## 1. Background

The current development workflow already works well at the individual-agent level.

For a new project:

1. The project is planned first.
2. A detailed implementation plan is written under `docs/`.
3. The plan is divided into bounded work packages / chunks.
4. Each package may have:
   - an intended agent;
   - dependencies;
   - a goal;
   - scope;
   - checkpoints;
   - verification criteria.
5. Claude Code or Codex CLI is manually opened.
6. The selected agent is instructed to read the relevant plan section and implement only that slice.
7. When finished, the agent:
   - commits its work;
   - runs verification;
   - writes a handoff document;
   - records important implementation decisions, changed interfaces, test results, limitations, and instructions for the next package.
8. The human then manually opens the next agent and tells it to continue.

The agents themselves already work effectively.

The problem is the coordination layer between them.

Today the workflow looks like:

```text
Plan
  ↓
Human opens Codex
  ↓
Codex executes Chunk 1
  ↓
Human observes completion
  ↓
Human opens Claude
  ↓
Human tells Claude to read Chunk 1 handoff
  ↓
Claude executes Chunk 2
  ↓
Human repeats
```

The human is therefore acting as a message bus, scheduler, process manager, and dependency resolver.

That work should be automated.

---

# 2. Intent

Build a small, local-first orchestration runtime that allows multiple coding-agent CLIs to execute different slices of one project automatically.

Initial supported agents:

```text
Codex CLI
Claude Code CLI
```

The system must not replace either agent.

The agents should continue operating through their native CLI environments.

The orchestrator only owns:

```text
task scheduling
agent selection
dependency resolution
process lifecycle
observation
human intervention
verification
handoff
run completion
```

The intent is not to build a new coding agent.

The intent is to coordinate existing coding agents.

---

# 3. Core Problem

Claude and Codex currently have independent:

- CLI processes;
- context windows;
- subscription / quota pools;
- execution sessions.

This is desirable.

For example:

```text
Chunk 1 → Codex
Chunk 2 → Claude
Chunk 3 → Codex
Chunk 4 → Claude
```

This allows work to be distributed across independent agent capacity.

However, completion of one task currently requires human coordination before another agent can begin.

The target behaviour is:

```text
Codex executes T1
       ↓
T1 finishes
       ↓
deterministic verification
       ↓
T1 becomes VERIFIED
       ↓
T2 dependency becomes satisfied
       ↓
orchestrator launches Claude
       ↓
Claude receives T1 handoff
       ↓
Claude executes T2
```

No human relay should be required.

---

# 4. Primary Goal

The user should be able to start an entire project execution with one command.

For example:

```bash
orchestrator run docs/implementation-work-packages.md
```

After this command:

```text
the orchestrator owns execution until:

COMPLETED
FAILED
BLOCKED
or CANCELLED
```

The human should not need to manually:

- open Codex;
- open Claude;
- copy handoff text;
- identify the next package;
- tell the next agent to begin.

---

# 5. Critical Design Principle

Agents do not schedule other agents.

Do NOT implement:

```text
Codex finishes
→ Codex launches Claude
```

Instead:

```text
             Orchestrator
                 │
       ┌─────────┴─────────┐
       ▼                   ▼
    Codex                Claude
```

The orchestrator is the parent.

Agents are workers.

Agents should only know:

```text
their current assignment
relevant context
write boundaries
verification requirements
```

They should not need to know:

```text
who executes next
whether they are the final task
how the overall workflow is scheduled
```

---

# 6. Existing Planning Model Must Be Preserved

Do not replace the existing project planning workflow.

Existing repository structure may look like:

```text
repo/
├── docs/
│   ├── product-spec.md
│   ├── implementation-work-packages.md
│   ├── architecture-decisions.md
│   └── requirements-checklist.md
│
├── handoffs/
│   ├── chunk-1.md
│   ├── chunk-2.md
│   └── ...
│
├── src/
└── tests/
```

The implementation work package document is authoritative for execution planning.

Typical package structure:

```text
## 1. Shared contracts

Suggested owner: Codex.
Dependencies: none.

Goal:
...

Done when:
...

Checkpoint:
...
```

Then:

```text
## 2. Lifetime ledger

Suggested owner: Claude.
Dependencies: 1.
```

The orchestrator should initially support this convention rather than require users to manually maintain another execution manifest.

A future machine-readable execution file may be supported, but it is not required for V0.1.

---

# 7. Task Model

Internally normalize each work package to something similar to:

```yaml
id: T2

source:
  file: docs/implementation-work-packages.md
  section: 2

title: Deterministic lifetime ledger

agent: claude

depends_on:
  - T1

status: BLOCKED

goal: >
  Calculate a transparent, reconciled
  year-by-year lifetime projection.

verification:
  - npm run check

handoff:
  expected: handoffs/chunk-2.md
```

State should be maintained internally.

The source Markdown document should not need runtime mutation.

---

# 8. Task Lifecycle

Use an explicit state machine.

```text
PLANNED
   ↓
BLOCKED
   ↓
READY
   ↓
RUNNING
   ↓
AGENT_DONE
   ↓
VERIFYING
   ├──────── fail ────────┐
   │                      ↓
   │                    REPAIR
   │                      │
   │                 retry budget
   │                      │
   │              ┌───────┴───────┐
   │              ▼               ▼
   │          VERIFYING          FAILED
   │
   └──────── pass
              ↓
           VERIFIED
```

Dependencies are released only after:

```text
VERIFIED
```

Not merely after:

```text
agent says "done"
```

---

# 9. Verification Is Authoritative

Agent completion is not workflow completion.

Required model:

```text
agent exits
   ↓
collect result
   ↓
run deterministic verification
   ↓
PASS
   ↓
task = VERIFIED
```

Verification may include:

```text
tests
type checking
lint
build
required artifact existence
handoff existence
git cleanliness rules
commit existence
scope checks
```

The exact commands may initially come from project conventions or package configuration.

---

# 10. Agent Execution

Initial runners:

```text
Claude Code
Codex CLI
```

Each task should run as an independent interactive CLI process.

Conceptually:

```text
run_codex(task)
run_claude(task)
```

Agents should execute one bounded package at a time.

Generated assignment prompt should include:

```text
authoritative project plan
assigned package
relevant completed handoffs
dependencies already verified
scope boundaries
verification requirements
required handoff
completion protocol
```

Example:

```text
Read:
docs/implementation-work-packages.md
Package 5 only.

Also read:
handoffs/chunk-4.md
docs/architecture-decisions.md

Package 4 has been VERIFIED.

Execute Package 5 only.

Do not work on later packages.

When finished:
1. run required checks;
2. commit the implementation;
3. update handoffs/chunk-5.md;
4. report completion;
5. exit.
```

---

# 11. Native CLI Visibility

This requirement is essential.

The orchestrator must not turn agents into invisible background subprocesses.

Each running agent should have a real interactive terminal session.

Preferred initial implementation:

```text
tmux
```

For example:

```text
tmux session: project-run-001

├── orchestrator
├── T1-codex
├── T2-claude
└── verifier
```

The human must be able to see exactly what each agent sees and does.

---

# 12. Observe Running Agent

Required command:

```bash
orchestrator attach T5
```

This should attach the user to the native CLI session.

The user should see:

```text
Codex / Claude output
commands
tool calls visible in CLI
test output
errors
agent messages
current prompt interaction
```

This should feel equivalent to manually opening the agent CLI.

Detach must not kill the agent.

---

# 13. Human Intervention

The human must remain able to intervene while an agent is running.

Two mechanisms should exist.

## 13.1 Direct attach

```bash
orchestrator attach T5
```

Then directly type into the agent CLI.

Example:

```text
Do not change src/domain/contracts.ts.
Preserve the existing interface.
```

## 13.2 Send without attach

```bash
orchestrator message T5 \
  "Preserve the existing public API."
```

The instruction should be delivered into the running interactive session.

---

# 14. Human Intervention Must Be Recorded

Every human intervention must become execution evidence.

Example:

```json
{
  "timestamp": "...",
  "task": "T5",
  "event": "human_message",
  "message": "Do not modify shared contracts."
}
```

This matters because later evaluation should distinguish:

```text
fully autonomous success
```

from:

```text
success after human correction
```

---

# 15. Observability

The purpose of observability is not only debugging.

It is to improve agent execution quality.

For every task capture, where reasonably possible:

```text
start time
finish time
elapsed time

agent/provider
model/session identifier if available

files inspected
files changed

commands executed
test executions
test failures
retries

human interventions

handoff produced
commit SHA

verification result
repair attempts
final task state
```

Do not attempt to capture hidden chain-of-thought.

Capture only externally observable behaviour.

---

# 16. Execution Trace

Each task should produce an append-only event stream.

Example:

```text
10:31 TASK_STARTED
10:32 AGENT_SESSION_CREATED
10:34 COMMAND
10:38 FILE_CHANGED
10:42 HUMAN_MESSAGE
10:45 TEST_FAILED
10:52 FILE_CHANGED
10:58 TEST_PASSED
11:01 AGENT_DONE
11:02 VERIFY_STARTED
11:03 VERIFY_PASSED
11:03 TASK_VERIFIED
```

Suggested storage:

```text
.orchestrator/
└── runs/
    └── run-001/
        ├── run.json
        ├── events.jsonl
        ├── T1/
        │   ├── events.jsonl
        │   ├── terminal.log
        │   ├── result.json
        │   └── metrics.json
        └── T2/
```

SQLite may be used for authoritative runtime state.

JSONL may be used for human-readable event traces.

---

# 17. Status View

Required:

```bash
orchestrator status
```

Example:

```text
RUN fire-v03-001

T1 Codex   VERIFIED   34m
T2 Claude  VERIFIED   52m
T3 Codex   RUNNING    18m
T4 Claude  BLOCKED    waiting on T3
T5 Codex   BLOCKED    waiting on T4
```

---

# 18. Live Watch

Useful V0/V0.2 command:

```bash
orchestrator watch
```

Example:

```text
T3 — CODEX — RUNNING

elapsed:        18m
last activity:  8s ago

current command:
npm test

files changed:
src/engine/property.ts
tests/property.test.ts

test attempts:
3

latest result:
2 failed / 18 passed

human interventions:
0
```

The watch interface must not replace attach.

Attach remains the authoritative way to inspect the live CLI.

---

# 19. Handoff Model

Existing repository handoffs should remain authoritative cross-agent context.

Example:

```text
handoffs/chunk-1.md
handoffs/chunk-2.md
...
```

When T1 completes:

```text
Codex
 ↓
writes handoffs/chunk-1.md
 ↓
verification
 ↓
T1 VERIFIED
```

Then T2 automatically receives:

```text
read handoffs/chunk-1.md
```

This avoids large agent-to-agent chat transcripts.

The repository itself remains the durable communication medium.

---

# 20. Handoff Requirements

Where projects use this convention, each handoff should contain:

```text
delivered scope

changed public interfaces

commands executed

test results

implementation decisions

known limitations

next checkpoint

remaining requirements
```

Agents should not depend on another agent's private conversational memory.

Only repository-visible artifacts should become durable cross-agent context.

---

# 21. Parallelism

V0.1 may execute sequential dependencies first.

However, architecture should allow:

```text
T1
 ├── T2
 └── T3
```

If T2 and T3 are independent:

```text
Claude → T2
Codex  → T3
```

may execute concurrently.

Concurrency is constrained by:

```text
dependency graph
agent availability
repository/worktree isolation
configured limits
```

---

# 22. Git Isolation

The orchestrator must avoid uncontrolled simultaneous editing of one working tree.

Recommended model:

```text
one git worktree per concurrently running task
```

Example:

```text
.worktrees/
├── T2-claude/
└── T3-codex/
```

Sequential execution may initially use a simpler strategy if safe.

But concurrent tasks must not write into the same working directory.

---

# 23. Agent Registry

Do not hard-code the architecture permanently around only two agents.

Initial registry:

```yaml
agents:
  codex:
    command: codex
    type: codex

  claude:
    command: claude
    type: claude-code
```

Future providers may include:

```text
Gemini CLI
OpenCode
local model
custom coding harness
```

V0 only needs Claude and Codex.

---

# 24. Routing

V0 should use explicit assignment from the work package.

For example:

```text
Suggested owner: Codex
```

or:

```text
Suggested owner: Claude
```

Do not initially build an AI routing system.

Future versions may route based on:

```text
task type
historical performance
remaining quota
complexity
cost
latency
specialisation
```

But this is explicitly not required for V0.

---

# 25. Quota Philosophy

One motivation is efficient use of independent coding-agent quotas.

The system should treat Claude and Codex as independent worker capacity.

Example:

```text
Codex capacity
Claude capacity
```

The orchestrator does not need to understand provider billing initially.

It only needs to avoid unnecessarily routing all tasks through one agent.

Future versions may model:

```text
rate limits
quota cooldowns
cost
capacity
preferred agent
```

---

# 26. Last Slice / Run Completion

The final agent must not decide that the project is finished.

After every verified task:

```text
check graph
```

If another task becomes READY:

```text
dispatch it
```

If:

```text
all required tasks == VERIFIED
```

then:

```text
run.status = COMPLETED
```

and exit successfully.

Expected exit:

```text
COMPLETED → process exit code 0
```

No special "tell orchestrator to exit" instruction should be needed for the last agent.

---

# 27. Run Terminal States

A run may finish as:

```text
COMPLETED
FAILED
BLOCKED
CANCELLED
```

Suggested process exit codes:

```text
0   COMPLETED
1   FAILED
2   BLOCKED
130 CANCELLED
```

---

# 28. Failure Handling

Do not silently continue after failure.

Possible failure categories:

```text
AGENT_PROCESS_FAILED
VERIFICATION_FAILED
REPAIR_EXHAUSTED
DEPENDENCY_FAILED
HUMAN_DECISION_REQUIRED
INVALID_PLAN
HANDOFF_MISSING
```

Blocked downstream tasks must remain blocked.

---

# 29. Repair

A failed verification may return the same bounded task to the same agent.

Example:

```text
T3
 ↓
agent done
 ↓
verify failed
 ↓
repair attempt 1
 ↓
verify
```

Use a configurable repair budget.

Example:

```yaml
max_repairs: 2
```

Do not allow infinite autonomous repair loops.

---

# 30. Session Retention

Do not immediately destroy completed agent sessions.

The user wants to inspect behaviour after execution.

Keep:

```text
terminal logs
event traces
task metadata
handoff
verification output
```

Optionally keep completed tmux sessions temporarily.

A separate command can remove them:

```bash
orchestrator clean run-001
```

---

# 31. Local-Only Requirement

V0 must run completely locally.

Do not require:

```text
AWS
Azure
GCP
hosted orchestration
hosted queues
hosted database
```

Allowed local dependencies:

```text
Python or equivalent runtime
tmux
SQLite
Git
Claude CLI
Codex CLI
```

The only external network use should be whatever Claude Code and Codex themselves normally require.

---

# 32. No MCP Requirement

Do not require MCP for V0.

MCP may later become useful for agent-to-agent tools such as:

```text
send_message
claim_task
query_status
```

But the first version should prefer simple local primitives.

---

# 33. Minimal Architecture

Recommended initial architecture:

```text
                    CLI ENTRYPOINT
                         │
                         ▼
                   Orchestrator
                         │
              ┌──────────┴──────────┐
              │                     │
              ▼                     ▼
         Plan Parser          Runtime State
                                SQLite
              │                     │
              └──────────┬──────────┘
                         ▼
                    Scheduler
                         │
              ┌──────────┴──────────┐
              ▼                     ▼
         Codex Runner          Claude Runner
              │                     │
              ▼                     ▼
           tmux pane             tmux pane
              │                     │
              └──────────┬──────────┘
                         ▼
                       Git
                         │
                         ▼
                     Verifier
                         │
                         ▼
                 dependency release
```

---

# 34. Suggested Repository Structure

```text
local-agent-orchestrator/
├── README.md
├── pyproject.toml
│
├── orchestrator/
│   ├── cli.py
│   ├── planner.py
│   ├── scheduler.py
│   ├── state.py
│   ├── verifier.py
│   ├── events.py
│   │
│   ├── agents/
│   │   ├── base.py
│   │   ├── claude.py
│   │   └── codex.py
│   │
│   ├── terminal/
│   │   └── tmux.py
│   │
│   └── git/
│       └── worktree.py
│
└── tests/
```

Exact language is not mandated if another implementation is materially better.

Prefer simplicity.

---

# 35. V0 CLI

Target commands:

```bash
orchestrator run <plan>

orchestrator status

orchestrator watch

orchestrator attach <task>

orchestrator message <task> "<message>"

orchestrator stop

orchestrator resume <run>

orchestrator inspect <task>

orchestrator clean <run>
```

Not all need to exist in the first commit.

Priority:

```text
run
status
attach
message
stop
```

---

# 36. V0 Scope

V0 must prove one scenario:

```text
Package 1 assigned to Codex
       ↓
Codex automatically launched
       ↓
human can observe Codex live
       ↓
human can message Codex live
       ↓
Codex finishes
       ↓
verification passes
       ↓
Package 2 automatically unlocked
       ↓
Claude automatically launched
       ↓
Claude reads prior handoff
       ↓
human can observe Claude live
       ↓
Claude finishes
       ↓
verification passes
       ↓
no remaining packages
       ↓
run exits COMPLETED
```

That is the primary acceptance test.

---

# 37. V0 Non-Goals

Do not build yet:

```text
web UI
cloud deployment
distributed workers
LLM-based task planning
LLM-based routing
complex swarm behaviour
agent voting
shared conversational memory
vector database
RAG platform
MCP server
cost optimiser
provider billing engine
full CI/CD integration
enterprise authentication
```

The goal is a reliable local execution primitive.

---

# 38. Key Product Principle

The system should remove the human from the **coordination loop**, not from the **observation loop**.

Target:

```text
Human:
plans
observes
intervenes when useful
reviews outcomes

Orchestrator:
schedules
launches
tracks
verifies
hands off
terminates

Agents:
execute bounded work
produce repository evidence
exit
```

---

# 39. Success Criteria

V0 is successful when the user can:

```text
1. write the implementation plan as they do today;

2. start one orchestrator command;

3. watch Codex and Claude working in their actual CLI sessions;

4. enter or message either running agent at any time;

5. leave the session without stopping execution;

6. have completion of one verified package automatically release the next;

7. have repository handoffs automatically become context for downstream packages;

8. inspect the full execution afterwards;

9. distinguish autonomous runs from human-intervened runs;

10. reach a deterministic final COMPLETED / FAILED / BLOCKED / CANCELLED state.
```

---

# 40. Longer-Term Purpose

This project should eventually make it possible to evaluate agent execution empirically.

Example future questions:

```text
Which agent performs better for which task shape?

Which task slices cause repeated human intervention?

Which handoff format reduces rediscovery?

How much time is spent reading versus editing versus repairing?

Which verification failures recur?

Does better context reduce token consumption?

When should work be parallelised?

How often can a project complete without human intervention?
```

The orchestrator therefore becomes not only a scheduler, but an experimental platform for improving agentic software delivery.

However:

Do not overbuild this capability in V0.

First make the execution loop reliable.

---

# 41. Core Principle

The final system should be understandable as:

```text
PLAN
  ↓
TASK GRAPH
  ↓
DISPATCH
  ↓
NATIVE CODING AGENT
  ↓
REPOSITORY ARTIFACTS
  ↓
DETERMINISTIC VERIFY
  ↓
STATE TRANSITION
  ↓
NEXT TASK
```

Not:

```text
LLMs casually talking to each other until they believe the project is finished.
```

Durable state, repository artifacts, explicit dependencies and deterministic verification should control execution.