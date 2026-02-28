# Agent Flow Conventions

Standard codes, event types, and naming conventions for Agent Flow implementations.

## Reason Codes

Every step completion emits a `reason_code`. These are the standard codes defined by the spec:

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

Workflows may extend this set with domain-specific reason codes declared in their step blocks (e.g. `BRIEF_VALIDATED`, `INPUT_VALIDATED`). Custom codes should use `UPPER_SNAKE_CASE`.

## Event Types

Required audit events emitted during workflow execution:

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

## Semantic Attributes

Following the OpenTelemetry pattern, Agent Flow defines standard attribute names for interoperability across implementations:

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

## State Namespaces

Workflows use a typed state dictionary with three namespaces:

| Namespace | Description | Example |
|-----------|-------------|---------|
| `input` | Read-only workflow input | `input.transcript_text` |
| `state` | Mutable intermediate state | `state.validated`, `state.extracted` |
| `output` | Workflow output (written by final step) | `output.report` |

## Naming Conventions

| Element | Convention | Example |
|---------|-----------|---------|
| Workflow names | kebab-case | `transcript-to-report` |
| Step IDs | snake_case | `validate_input` |
| Agent IDs | snake_case | `action_extractor` |
| Bundle names | snake_case | `extract_pack` |
| Reason codes | UPPER_SNAKE_CASE | `BUDGET_EXCEEDED` |
| Event types | snake_case | `step_complete` |
| State keys | dot-separated snake_case | `state.validated` |
| Semantic attributes | dot-separated lowercase | `agent_flow.step.id` |
