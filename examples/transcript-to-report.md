---
name: transcript-to-report
kind: agent-flow/workflow
version: 1.0.0
description: >
  Process a meeting transcript into a structured report with
  action items, risks, key decisions, and follow-ups.

category: document-processing
icon: FileText
status: active
risk_profile: medium

budgets:
  max_steps: 10
  max_tool_calls: 5
  max_tokens: 40000
  deadline_seconds: 300

triggers:
  manual: true
  api: true

tools:
  allowlist: [drive.read, calendar.lookup]
  workers_can_call_tools: false

input:
  schema:
    type: object
    required: [transcript_text]
    properties:
      transcript_text: { type: string }
      meeting_date: { type: string, format: date }
      attendees: { type: array, items: { type: string } }

output:
  schema:
    type: object
    required: [report]
    properties:
      report:
        type: object
        properties:
          summary: { type: string }
          actions: { type: array }
          risks: { type: array }
          decisions: { type: array }
---

# Purpose

Turn raw meeting transcripts into structured, actionable reports. Three specialists extract different insight types in parallel, a critic reviews quality, and the final report is assembled with a full audit trail.

A product manager can modify the agents' roles and expected outputs to change what gets extracted — no code changes needed.

## Agents

```agent
id: action_extractor
role: Extract action items from meeting transcript
goal: Identify every commitment made, who owns it, and any stated deadline
expected_output: |
  List of objects with fields:
  - owner: person name
  - action: what they committed to do
  - deadline: stated deadline or "not specified"
  - verbatim_quote: exact words from transcript
```

```agent
id: risk_extractor
role: Identify risks, concerns, and blockers mentioned in the meeting
goal: Surface anything that could delay or derail discussed initiatives
expected_output: |
  List of objects with fields:
  - risk: description of the risk
  - severity: low | medium | high
  - raised_by: who mentioned it
  - verbatim_quote: exact words from transcript
```

```agent
id: theme_extractor
role: Identify key decisions, themes, and strategic topics
goal: Capture what was decided and the major discussion threads
expected_output: |
  List of objects with fields:
  - type: decision | theme
  - description: summary
  - verbatim_quote: exact words from transcript
```

```agent
id: critic
role: Quality reviewer for extracted content
goal: Verify extractions are accurate, complete, and not hallucinated against the source transcript
expected_output: |
  Object with fields:
  - approved: true/false
  - issues: list of { step, problem, suggestion }
  - confidence: 0.0 to 1.0
```

## Steps

```step
id: validate_input
type: transform
description: Verify transcript is present and has enough content to process
reads: [input]
writes: [state.validated]
on_error: stop
reason_code: INPUT_VALIDATED
reason_code_on_fail: FAILED_VALIDATION
```

```step
id: extract_insights
type: parallel
description: Three specialists extract actions, risks, and themes simultaneously
reads: [state.validated]
writes: [state.extracted]
bundle: extract_pack
reason_code: INSIGHTS_EXTRACTED
```

```step
id: qa_review
type: gate
description: Critic checks all extractions against the source transcript
reads: [state.validated, state.extracted]
writes: [state.reviewed]
agent: critic
gate_method: critic_agent
when: risk_profile != "low"
on_error: fallback
fallback: qa_simple
reason_code: QA_PASSED
reason_code_on_fail: QA_FAILED
```

```step
id: qa_simple
type: transform
description: Basic validation — check that outputs are non-empty
reads: [state.extracted]
writes: [state.reviewed]
reason_code: QA_SIMPLE_PASSED
```

```step
id: assemble_report
type: skill
description: Combine reviewed extractions into a polished report
reads: [state.reviewed]
writes: [output]
expected_output: |
  A structured report with:
  - Executive summary (2-3 paragraphs)
  - Action items table (owner, action, deadline)
  - Risk register (risk, severity, owner)
  - Key decisions log
  Professional tone, suitable for distribution to stakeholders.
stop_condition: output.report != null
reason_code: REPORT_ASSEMBLED
```

## Bundles

```bundle
name: extract_pack
version: 1.0

budgets:
  max_steps_per_worker: 3
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

## Observability

```observability
audit_level: full
log_format: ndjson
required_events:
  - run_start
  - step_start
  - step_output
  - step_complete
  - gate_decision
  - budget_check
  - run_complete
  - run_failed
```
