# Agent Flow Specification

**Version**: 0.2.0 (Draft)

## 1. Introduction

### 1.1 Purpose

Agent Flow is a specification for defining agentic AI workflows as Markdown files with structured annotations. It provides a standard way to describe multi-step, multi-agent processes that are:

- **Readable** by business stakeholders without technical background
- **Parseable** by machines for validation, visualization, and execution
- **Auditable** with structured event logs verifiable by both humans and machines
- **Resilient** to the inherent non-determinism of large language models

### 1.2 The Non-Determinism Problem

Large language models produce different outputs for the same input. This is fundamental to how they work — temperature, sampling, and context window variations all contribute. Rather than fighting this property, Agent Flow embraces it by applying **structural guardrails**:

- **Deterministic structure**: Step ordering, data flow, budgets, gates, and tool permissions are fixed
- **Non-deterministic reasoning**: LLM creativity, analysis, and generation within each step are unconstrained
- **Verification at boundaries**: Inputs and outputs are validated at step edges, not during LLM reasoning

This separation means workflows are predictable and auditable at the process level while preserving LLM capability at the task level.

### 1.3 Relationship to Agent Skills Spec

Agent Flow is a **strict superset** of the [Agent Skills Spec](https://github.com/anthropics/agent-skills-spec). Any valid `SKILL.md` file is a valid Agent Flow workflow with a single implicit step. Agent Flow adds:

- Multi-step orchestration
- Agent definitions with roles and constraints
- Parallel execution with merge rules
- Budget enforcement
- Audit event streams
- Long-running semantics with checkpoints and waitpoints

### 1.4 Layered Complexity

Not every workflow needs every feature. Agent Flow is organized into four additive layers:

| Layer | Name | Adds | Minimum Frontmatter |
|-------|------|------|---------------------|
| 0 | Skill | Single implicit step | `name`, `description` |
| 1 | Linear | Ordered step blocks | `kind: agent-flow/workflow` |
| 2 | Graph | Conditions, parallelism, branching | Step `when`, `branches`, `bundle` |
| 3 | Long-running | Checkpoints, waitpoints, signals | `runtime` block |

A workflow's layer is inferred from its content — there is no explicit layer field.

---

## 2. File Format

### 2.1 Overview

An Agent Flow workflow is a single Markdown file (`.md`) with:

1. **YAML frontmatter** — machine-parseable metadata, budgets, tool governance, I/O schemas
2. **Well-known headings** — human-friendly sections (`# Purpose`, `## Steps`, `## Agents`, etc.)
3. **Fenced blocks** — structured YAML inside labeled code fences (`\`\`\`step`, `\`\`\`agent`, `\`\`\`bundle`, etc.)

### 2.2 Document Structure

```
┌─────────────────────────────────┐
│  YAML Frontmatter               │  metadata, budgets, tools, I/O
│  (--- delimited)                │
├─────────────────────────────────┤
│  # Purpose                      │  business-friendly description
├─────────────────────────────────┤
│  ## Agents                      │  agent definitions (optional)
├─────────────────────────────────┤
│  ## Steps                       │  step blocks (Layer 1+)
├─────────────────────────────────┤
│  ## Bundles                     │  parallel worker groups (optional)
├─────────────────────────────────┤
│  ## Runtime                     │  long-running config (optional)
├─────────────────────────────────┤
│  ## Observability               │  audit configuration (optional)
├─────────────────────────────────┤
│  # Base Skill                   │  extended skill content (if overlay)
└─────────────────────────────────┘
```

Headings can appear in any order. Parsers identify sections by heading text, not position.

### 2.3 Frontmatter Schema

```yaml
---
# === Required (Layer 0) ===
name: string                     # kebab-case identifier
description: string              # what this workflow does and when to use it

# === Required for workflows (Layer 1+) ===
kind: agent-flow/workflow        # identifies this as an Agent Flow workflow
version: string                  # semver (e.g., "1.0.0")

# === Optional — Skill compatibility ===
category: string                 # e.g., "document-processing", "analysis"
icon: string                     # unicode name or emoji
status: string                   # draft | active | deprecated
owner: string                    # team or person responsible for this workflow

# === Optional — Overlay ===
extends: string                  # path to base skill this workflow wraps

# === Optional — Risk ===
risk_profile: string             # low | medium | high | critical

# === Optional — Budgets ===
budgets:
  max_steps: integer             # hard cap on step executions
  max_tool_calls: integer        # hard cap on tool invocations
  max_tokens: integer            # total token budget across all LLM calls
  deadline_seconds: integer      # wall-clock timeout for entire run

# === Optional — Tool governance ===
tools:
  allowlist: [string]            # only these tools may be called
  denylist: [string]             # these tools are prohibited
  workers_can_call_tools: boolean  # whether bundle workers can use tools (default: false)

# === Optional — I/O contracts ===
input:
  schema: object                 # inline JSON Schema
  schema_ref: string             # or path to external JSON Schema file

output:
  schema: object                 # inline JSON Schema
  schema_ref: string             # or path to external JSON Schema file
---
```

### 2.4 Triggers

Trigger configuration defines how and when a workflow is executed. This lives in the frontmatter so the workflow file is fully self-contained — no separate schedule or task record needed.

```yaml
# === Optional — Triggers ===
triggers:
  schedule: string             # cron expression (e.g., "0 6 * * *")
  timezone: string             # IANA timezone (default: UTC)
  manual: boolean              # can be run on demand (default: true)
  api: boolean                 # can be triggered via API (default: false)
  event: string                # event type that triggers this workflow (e.g., "file_uploaded")
```

**Design rationale**: The workflow file IS the task definition. A platform reads the frontmatter, registers triggers, and manages executions. This eliminates the need for separate "schedule" or "task" database records pointing at workflow files.

### 2.5 Execution Model

A workflow has two concepts:

| Concept | What it is | Persistence |
|---------|-----------|-------------|
| **Workflow** | The definition of work and how it's triggered | `.md` file on disk |
| **Execution** | A single run of a workflow | Database record |

The workflow file is the source of truth. Platforms may cache workflow metadata in a database table for fast listing and querying, but the file is canonical.

All frontmatter fields beyond `name` and `description` are optional. A file with only those two fields is a valid Layer 0 workflow (plain skill).

---

## 3. Steps

### 3.1 Step Blocks

Steps are defined as fenced YAML blocks under the `## Steps` heading:

````markdown
## Steps

```step
id: validate_input
type: transform
description: Check the brief has enough detail to proceed
reads: [input]
writes: [state.validated]
on_error: stop
reason_code: BRIEF_VALIDATED
reason_code_on_fail: BRIEF_INSUFFICIENT
```
````

### 3.2 Step Properties

| Property | Required | Type | Description |
|----------|----------|------|-------------|
| `id` | Yes | string | Stable identifier, unique within workflow |
| `type` | Yes | string | Step type (see §3.3) |
| `description` | Yes | string | Business-friendly explanation of what this step does |
| `reads` | No | [string] | State keys this step requires as input |
| `writes` | No | [string] | State keys this step produces |
| `when` | No | string | Condition expression; step is skipped if false |
| `on_error` | No | string | `stop` \| `skip` \| `fallback` \| `retry` |
| `fallback` | No | string | Step ID to execute if this step fails (when `on_error: fallback`) |
| `retry` | No | object | Retry configuration (see §3.5) |
| `expected_output` | No | string | Description of what the step should produce (guides LLM output) |
| `stop_condition` | No | string | Condition expression; workflow succeeds when met |
| `reason_code` | No | string | Reason code emitted on success |
| `reason_code_on_fail` | No | string | Reason code emitted on failure |
| `agent` | No | string | Agent ID to execute this step (for skill/gate types) |
| `skill_ref` | No | string | Path to skill file (for `type: skill`) |
| `tool` | No | string | Tool identifier (for `type: tool`) |
| `bundle` | No | string | Bundle name (for `type: parallel`) |
| `branches` | No | object | Named routes (for `type: decision`) |
| `goto` | No | string | Explicit next step ID (overrides sequential ordering) |
| `gate_method` | No | string | `human_review` \| `automated` \| `critic_agent` |
| `output_files` | No | [string] | Files this step creates (e.g. `report.pdf`, `summary.md`) |
| `audit_output` | No | string | Path to JSON audit file for reconciliation |

### 3.3 Step Types

| Type | Purpose | Business Analogy |
|------|---------|-----------------|
| `transform` | Validate, reshape, or compute data | "Check the form is filled in correctly" |
| `skill` | Execute another skill or workflow | "Hand this to the specialist" |
| `tool` | Call an external tool or API | "Look up the file / search the web" |
| `decision` | Choose a path based on state | "Decide which department handles this" |
| `gate` | Approval checkpoint (human or automated) | "Manager signs off before proceeding" |
| `parallel` | Fan out to a bundle of workers | "Split the work across the team" |
| `subagent_bundle` | Alias for `parallel` | Same as `parallel` — alternative naming for clarity |
| `end` | Explicit terminal step | "Done — close the file" |

### 3.4 Execution Model

**Layer 1 (linear):** Steps execute top-to-bottom. The output of step N is available to step N+1 via shared `state`.

**Layer 2 (graph):** Steps can use:
- `when` to conditionally skip
- `goto` to jump to a non-adjacent step
- `branches` (on `decision` steps) to route to different paths
- `parallel` type to fan out to a bundle

**Implicit edges:** If step B `reads` what step A `writes`, there is an implicit dependency edge A → B.

**Explicit edges:** `goto` and `branches` create explicit edges that override sequential ordering.

### 3.5 Retry Configuration

```yaml
retry:
  max_attempts: 3              # total attempts (including first)
  backoff_ms: [500, 2000, 5000]  # delay before each retry
  retry_on: [TIMEOUT, API_ERROR]  # specific error types to retry (optional)
```

### 3.6 Decision Branches

```step
id: route_by_type
type: decision
description: Choose processing path based on document type
reads: [state.doc_type]
branches:
  transcript: process_transcript
  proposal: process_proposal
  default: process_generic
```

The `branches` map evaluates `reads` values and routes to the named step ID. A `default` key handles unmatched values.

---

## 4. Agents

### 4.1 Agent Blocks

Agents define the "who" — named specialists with roles, constraints, and expected outputs:

````markdown
## Agents

```agent
id: action_extractor
role: Extract action items from meeting content
goal: Identify every commitment, owner, and deadline mentioned
tools: []
model: claude-sonnet-4-6
max_tokens: 4000
expected_output: |
  List of objects: { owner, action, deadline, verbatim_quote }
```
````

### 4.2 Agent Properties

| Property | Required | Type | Description |
|----------|----------|------|-------------|
| `id` | Yes | string | Unique identifier |
| `role` | Yes | string | What this agent does (reads like a job title + description) |
| `goal` | Yes | string | What outcome this agent aims for |
| `tools` | No | [string] | Tools this agent can use (empty = no tools) |
| `model` | No | string | LLM model to use |
| `max_tokens` | No | integer | Token budget for this agent's outputs |
| `expected_output` | No | string | Description of the output format and content |

### 4.3 Design Intent

Agent definitions are deliberately **business-readable**. They read like job descriptions:

- **Role** = job title and responsibilities
- **Goal** = success criteria
- **Tools** = what resources they have access to
- **Expected output** = deliverable format

A product manager can review agents and adjust `role`, `goal`, and `expected_output` without understanding the underlying LLM mechanics. This is the primary mechanism for steering non-deterministic LLM behavior — clear role definition rather than prompt engineering.

---

## 5. Bundles

### 5.1 Bundle Blocks

Bundles define parallel execution groups — multiple agents working simultaneously on different aspects of the same input:

````markdown
## Bundles

```bundle
name: extract_pack
version: 1.0

budgets:
  max_steps_per_worker: 4
  deadline_seconds_per_worker: 45
  max_tokens_per_worker: 5000

workers:
  - id: W1_ACTIONS
    agent: action_extractor
  - id: W2_RISKS
    agent: risk_extractor
  - id: W3_THEMES
    agent: theme_extractor

merge:
  strategy: combine_by_type
  dedupe_key: [type, label]
  conflict: send_to_critic

worker_output:
  format: agent-flow/worker.v1
  required_fields: [type, items, confidence]
```
````

### 5.2 Bundle Properties

| Property | Required | Type | Description |
|----------|----------|------|-------------|
| `name` | Yes | string | Bundle identifier, referenced by step `bundle` field |
| `version` | No | string | Semver for the bundle definition |
| `budgets` | No | object | Per-worker resource limits |
| `workers` | Yes | [object] | List of workers with `id` and `agent` reference |
| `merge` | Yes | object | How to combine worker outputs |
| `worker_output` | No | object | Common output schema for all workers |

### 5.3 Merge Strategies

When multiple agents process the same input in parallel, their outputs must be combined. The `merge` block makes this explicit:

| Strategy | Behavior | Use Case |
|----------|----------|----------|
| `combine_by_type` | Group outputs by `type` field, concatenate | Different specialists, different output types |
| `union` | Keep all outputs, deduplicate by key | Overlapping specialists |
| `vote` | Majority wins on conflicting values | Same task, multiple attempts for reliability |
| `send_to_critic` | Route conflicts to a critic agent for resolution | High-stakes disagreements |

The `conflict` field on the merge block specifies what happens when workers produce contradictory outputs:
- `first_wins` — keep the first worker's value
- `last_wins` — keep the last worker's value
- `send_to_critic` — route to a designated critic agent
- `fail` — treat the conflict as an error

### 5.4 Per-Worker Budgets

Each worker in a bundle gets its own budget. One runaway agent cannot consume the resources allocated to other workers. This is a critical structural guardrail for parallel non-deterministic execution.

---

## 6. State Model

### 6.1 State Dictionary

Workflows use a **typed state dictionary** that flows through the execution graph. Each step declares what it reads from and writes to state.

The state has three namespaces:

| Namespace | Description | Example |
|-----------|-------------|---------|
| `input` | Read-only workflow input | `input.transcript_text` |
| `state` | Mutable intermediate state | `state.validated`, `state.extracted` |
| `output` | Workflow output (written by final step) | `output.report` |

### 6.2 Reads and Writes

- `reads: [state.validated]` — this step requires `state.validated` to exist before it can execute
- `writes: [state.extracted]` — this step produces `state.extracted`

The engine uses these declarations to:
1. **Validate** that all dependencies are satisfied before a step runs
2. **Derive implicit edges** in the execution graph
3. **Checkpoint** state after each step for resumability
4. **Audit** what data each step consumed and produced

### 6.3 State Scoping

State is flat within each namespace. Keys use dot notation for readability but are not deeply nested:

```
state.validated          ← the validated input object
state.extracted.actions  ← actions extracted from the transcript
state.qa.approved        ← whether QA passed
```

---

## 7. Audit & Observability

### 7.1 Design Philosophy

Every workflow run must produce an audit trail that is:

- **Verifiable by humans** — reason codes, timestamps, and gate evidence in plain language
- **Verifiable by machines** — structured NDJSON events with consistent schema, validatable against the workflow definition

This is not optional logging. It is a core requirement of the specification. A workflow that cannot be audited after the fact cannot be trusted.

### 7.2 Observability Block

````markdown
## Observability

```observability
audit_level: full
log_format: ndjson

required_events:
  - run_start
  - step_start
  - step_output
  - step_complete
  - step_skipped
  - gate_decision
  - budget_check
  - run_complete
  - run_failed

required_ids:
  - run_id
  - trace_id
  - step_id

redaction:
  pii: true
  secrets: true
  patterns: ["SSN", "credit_card"]
```
````

### 7.3 Audit Event Schema

Every audit event is a JSON object with these base fields:

```json
{
  "run_id": "string (uuid)",
  "trace_id": "string (uuid)",
  "step_id": "string",
  "event": "string (event type)",
  "timestamp": "string (ISO 8601)",
  "data": {}
}
```

### 7.4 Event Types

| Event | Emitted When | Key Data |
|-------|-------------|----------|
| `run_start` | Workflow begins | `workflow_name`, `version`, `input_summary`, `budgets` |
| `step_start` | Step begins | `step_id`, `type`, `reads` |
| `step_output` | Step produces output | `step_id`, `writes`, `output_summary` |
| `step_complete` | Step finishes | `step_id`, `status`, `duration_ms`, `tokens`, `reason_code` |
| `step_skipped` | Step skipped (condition false) | `step_id`, `condition`, `reason_code` |
| `gate_decision` | Gate resolved | `step_id`, `result`, `actor`, `method`, `evidence` |
| `budget_check` | After each step | `tokens_used`, `tokens_remaining`, `steps_used`, `steps_remaining` |
| `run_complete` | Workflow finishes | `status`, `total_duration_ms`, `total_tokens`, `output_summary` |
| `run_failed` | Workflow fails | `error`, `last_step`, `reason_code` |

### 7.5 Reason Codes

Every step completion emits a `reason_code` — a workflow-defined, human-readable string explaining the outcome. The spec defines a standard set:

| Code | Meaning |
|------|---------|
| `COMPLETED` | Step finished normally |
| `FAILED_VALIDATION` | Input validation failed |
| `BUDGET_EXCEEDED` | Token, step, or time budget hit |
| `TIMEOUT` | Deadline reached |
| `FALLBACK_USED` | Primary step failed, fallback executed |
| `GATE_APPROVED` | Human or automated reviewer approved |
| `GATE_REJECTED` | Reviewer rejected |
| `SKIPPED_CONDITION` | Step skipped because `when` condition was false |

Workflows extend this set with domain-specific reason codes declared in their step blocks.

### 7.6 Gate Audit Records

Gates are the highest-verification steps. Their audit records include:

```json
{
  "event": "gate_decision",
  "step_id": "stakeholder_approval",
  "result": "approved",
  "actor": "lindsay@company.com",
  "method": "human_review",
  "evidence": "Reviewer confirmed report covers all agenda items",
  "timestamp": "2026-02-28T14:32:15Z"
}
```

This is the link between structure and trust. When a stakeholder asks "who approved this and why?", the answer is in the audit log.

---

## 8. Runtime (Long-Running Workflows)

### 8.1 Runtime Block

For workflows that pause and resume — waiting for human approval, file uploads, external events:

````markdown
## Runtime

```runtime
checkpoint_after_each_step: true
resume_supported: true
max_concurrency: 4
approval_required: false
human_in_the_loop: false

checkpoints:
  - every: 3_steps

waitpoints:
  - id: WP_APPROVAL
    after_step: stakeholder_approval
    event: human_approval
    timeout_seconds: 172800
    on_timeout: notify_then_stop

  - id: WP_UPLOAD
    after_step: request_materials
    event: file_uploaded
    timeout_seconds: 86400
    on_timeout: stop
```
````

### 8.2 Runtime Properties

| Property | Type | Description |
|----------|------|-------------|
| `checkpoint_after_each_step` | boolean | Persist state after every step (default: false) |
| `resume_supported` | boolean | Whether the workflow can be resumed from a checkpoint |
| `max_concurrency` | integer | Maximum parallel workers (default: unbounded) |
| `approval_required` | boolean | Whether human approval is needed before execution (default: false) |
| `human_in_the_loop` | boolean | Whether a human can intervene at any point (default: false) |
| `checkpoints` | [object] | Periodic state snapshots (e.g. `every: 3_steps`) |

### 8.3 Waitpoints

| Property | Type | Description |
|----------|------|-------------|
| `id` | string | Unique identifier for this waitpoint |
| `after_step` | string | Step ID after which to pause |
| `event` | string | External event type that resumes execution |
| `timeout_seconds` | integer | Maximum wait time |
| `on_timeout` | string | `stop` \| `notify_then_stop` \| `skip` \| `fallback` |

### 8.4 Checkpoint State

When a checkpoint is taken, the full `state` dictionary is persisted. On resume, the engine:

1. Loads the checkpoint state
2. Identifies the next step to execute
3. Continues execution from that point

The audit log records checkpoint and resume events for traceability.

---

## 9. Overlay Model

### 9.1 Extending a Base Skill

Agent Flow workflows can **wrap** an existing skill, adding orchestration around it:

```yaml
---
name: transcript-to-report
kind: agent-flow/workflow
extends: skills/transcript-summariser.md
---
```

The `extends` field references a base skill. The workflow uses the base skill as one step among many (fetch, validate, process, QA, render).

### 9.2 Wrap vs. Override

**Wrap (recommended):** The base skill remains untouched. The workflow adds steps around it — validation before, QA after, formatting at the end.

**Override (rare):** The workflow can override specific properties of the base skill using an override block:

````markdown
```override
target: base_skill.output
replace_with: schemas/report-output.json
```
````

Use sparingly. Wrapping is almost always cleaner.

---

## 10. Semantic Conventions

### 10.1 Agent Flow Attributes

Following the OpenTelemetry pattern of semantic conventions, Agent Flow defines standard attribute names for interoperability:

| Attribute | Type | Description |
|-----------|------|-------------|
| `agent_flow.workflow.name` | string | Workflow name from frontmatter |
| `agent_flow.workflow.version` | string | Workflow version |
| `agent_flow.run.id` | string | Unique run identifier |
| `agent_flow.step.id` | string | Current step identifier |
| `agent_flow.step.type` | string | Step type |
| `agent_flow.agent.id` | string | Agent identifier |
| `agent_flow.agent.role` | string | Agent role |
| `agent_flow.tokens.input` | integer | Input tokens consumed |
| `agent_flow.tokens.output` | integer | Output tokens produced |
| `agent_flow.budget.tokens.remaining` | integer | Remaining token budget |
| `agent_flow.budget.steps.remaining` | integer | Remaining step budget |
| `agent_flow.reason_code` | string | Outcome reason code |

---

## 11. Conformance Levels

Implementations can claim conformance at different levels:

| Level | Requirements |
|-------|-------------|
| **Parser** | Parse frontmatter + fenced blocks, produce typed AST |
| **Validator** | Validate AST against this spec's schema |
| **Visualizer** | Render steps as graph nodes with edges derived from reads/writes |
| **Runner (Linear)** | Execute Layer 0-1 workflows (sequential steps) |
| **Runner (Full)** | Execute Layer 0-3 workflows (branching, parallel, long-running) |
| **Auditor** | Emit conformant NDJSON audit events for all step executions |

---

## 12. Recommended Workflow Patterns

Agent Flow defines three recommended starting patterns. These serve as templates for new workflows and encode best practices for budgets, observability, and error handling.

### 12.1 Simple Pipeline

A linear sequence of steps with no agents or branching. Suitable for scheduled reports, data extraction, and single-task automation.

```
validate → process → output
```

**Characteristics:**
- Layer 1 (linear)
- All steps are `transform` or `tool` type
- No agent definitions needed
- Low budgets (few steps, few tokens)
- Best for deterministic, repeatable processes

### 12.2 Agentic Workflow

Multiple specialist agents with quality review. Suitable for analysis, content creation, and research tasks.

```
validate → specialist work → QA gate → assemble output
```

**Characteristics:**
- Layer 1-2 (linear or with conditional QA)
- Agent definitions with role, goal, and expected output
- A `gate` step with a critic agent for quality assurance
- Medium budgets
- The `expected_output` field on agents and steps is the key structural guardrail — it constrains LLM output shape without limiting reasoning

### 12.3 Parallel Fan-out

Split work across multiple specialists, merge results with conflict handling, and confirm before proceeding. Suitable for multi-perspective analysis and document processing.

```
validate → fan-out to workers → merge → confirmation gate → output
```

**Characteristics:**
- Layer 2 (parallel bundles)
- Bundle with multiple workers, each with per-worker budgets
- Explicit merge strategy and conflict handling
- Confirmation gate before final output (human or automated)
- Higher budgets to accommodate parallel execution
- The merge step makes conflict resolution auditable — not silent overwrite

---

## Appendix A: Standard Step Types Reference

| Type | Required Fields | Optional Fields |
|------|----------------|-----------------|
| `transform` | `id`, `type`, `description` | `reads`, `writes`, `when`, `on_error`, `expected_output`, `output_files`, `audit_output` |
| `skill` | `id`, `type`, `description`, `skill_ref` or `agent` | `reads`, `writes`, `when`, `on_error`, `expected_output`, `output_files`, `audit_output` |
| `tool` | `id`, `type`, `description`, `tool` | `reads`, `writes`, `when`, `retry`, `fallback`, `output_files`, `audit_output` |
| `decision` | `id`, `type`, `description`, `branches` | `reads` |
| `gate` | `id`, `type`, `description` | `reads`, `writes`, `agent`, `gate_method`, `when` |
| `parallel` | `id`, `type`, `description`, `bundle` | `reads`, `writes` |
| `subagent_bundle` | `id`, `type`, `description`, `bundle` | `reads`, `writes` (alias for `parallel`) |
| `end` | `id`, `type` | `description`, `reason_code` |

## Appendix B: Full Frontmatter JSON Schema

See `spec/schema.json` for the machine-readable JSON Schema that validates Agent Flow frontmatter.

## Appendix C: NDJSON Audit Format

Each line in an audit log file is a self-contained JSON object conforming to the event schema defined in §7.3. Lines are ordered chronologically. The file extension should be `.audit.ndjson`.

Example:

```
{"run_id":"a1b2","trace_id":"c3d4","event":"run_start","timestamp":"2026-02-28T14:30:00Z","data":{"workflow_name":"transcript-to-report","version":"1.0.0"}}
{"run_id":"a1b2","trace_id":"c3d4","step_id":"validate","event":"step_start","timestamp":"2026-02-28T14:30:01Z","data":{"type":"transform","reads":["input"]}}
{"run_id":"a1b2","trace_id":"c3d4","step_id":"validate","event":"step_complete","timestamp":"2026-02-28T14:30:02Z","data":{"status":"completed","duration_ms":1100,"reason_code":"INPUT_VALIDATED"}}
```
