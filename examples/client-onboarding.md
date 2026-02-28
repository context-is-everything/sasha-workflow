---
name: client-onboarding
kind: agent-flow/workflow
version: 1.0.0
description: >
  End-to-end client onboarding workflow that validates intake data,
  runs parallel background checks, requires human approval at a gate,
  generates a welcome pack, and notifies the team.

category: operations
icon: UserPlus
status: active
owner: client-services-team
risk_profile: high

budgets:
  max_steps: 15
  max_tool_calls: 10
  max_tokens: 80000
  deadline_seconds: 600

triggers:
  manual: true
  api: true
  schedule: "0 9 * * 1"

tools:
  allowlist: [crm.read, crm.write, email.send, calendar.create, filesystem.read, filesystem.write]
  workers_can_call_tools: true

input:
  schema:
    type: object
    required: [client_name, contact_email, service_tier]
    properties:
      client_name: { type: string, description: "Full legal name of the client organisation" }
      contact_email: { type: string, format: email }
      service_tier: { type: string, enum: [starter, professional, enterprise] }
      industry: { type: string }
      notes: { type: string, description: "Any special requirements or context" }

output:
  schema:
    type: object
    required: [onboarding_pack, notifications_sent]
    properties:
      onboarding_pack:
        type: object
        properties:
          welcome_letter: { type: string }
          setup_checklist: { type: array }
          meeting_link: { type: string }
      notifications_sent: { type: boolean }
---

# Purpose

Automate the client onboarding journey from intake form through to welcome pack delivery. Three specialists run background research in parallel, a senior reviewer approves the engagement, and the final pack is assembled and dispatched automatically.

This workflow demonstrates every major feature of the Agent Flow v0.2.0 spec: subagent bundles (parallel execution), conditional logic, approval gates, error handling with fallback and retry, output files, audit output, runtime waitpoints with checkpoints, human-in-the-loop approval, PII redaction, and full audit observability.

## Agents

```agent
id: compliance_checker
role: Verify client meets regulatory and compliance requirements
goal: Flag any compliance risks based on industry, jurisdiction, and service tier
expected_output: |
  Object with fields:
  - compliant: true/false
  - flags: list of { area, severity, detail }
  - recommendation: approve / review_required / reject
```

```agent
id: market_researcher
role: Research the client's market position and recent activity
goal: Build a brief profile of the client's industry standing and any recent news
expected_output: |
  Object with fields:
  - company_summary: 2-3 sentence overview
  - recent_news: list of { headline, source, date }
  - competitors: list of company names
```

```agent
id: account_planner
role: Draft a tailored service plan based on client needs
goal: Match the client's service tier and industry to recommended service components
expected_output: |
  Object with fields:
  - recommended_services: list of { service, reason }
  - timeline: list of { milestone, target_date }
  - estimated_effort: string
```

```agent
id: senior_reviewer
role: Quality reviewer and approval authority for new client engagements
goal: Review all research and compliance findings, then approve or reject the onboarding
expected_output: |
  Object with fields:
  - approved: true/false
  - conditions: list of strings (any conditions attached to approval)
  - notes: string
```

## Steps

```step
id: validate_intake
type: transform
description: Check that all required client information is present and correctly formatted
reads: [input]
writes: [state.validated_intake]
on_error: stop
reason_code: INTAKE_VALIDATED
reason_code_on_fail: INTAKE_INVALID
```

```step
id: check_duplicates
type: tool
description: Search the CRM for existing records to prevent duplicate client entries
reads: [state.validated_intake]
writes: [state.duplicate_check]
tool: crm.read
on_error: retry
retry:
  max_attempts: 2
  backoff_ms: [500, 1000]
reason_code: DUPLICATE_CHECK_DONE
reason_code_on_fail: DUPLICATE_CHECK_FAILED
```

```step
id: run_background_research
type: subagent_bundle
description: Three specialists research compliance, market position, and service fit simultaneously
reads: [state.validated_intake, state.duplicate_check]
writes: [state.research_results]
bundle: research_pack
audit_output: audit-background-research.json
reason_code: RESEARCH_COMPLETE
```

```step
id: review_and_approve
type: gate
description: Senior reviewer examines all findings and decides whether to proceed
reads: [state.validated_intake, state.research_results]
writes: [state.approval]
agent: senior_reviewer
gate_method: critic_agent
when: risk_profile != "low"
on_error: fallback
fallback: auto_approve
reason_code: APPROVAL_GRANTED
reason_code_on_fail: APPROVAL_DENIED
```

```step
id: auto_approve
type: transform
description: Automatic approval for low-risk clients that skip the manual review
reads: [state.research_results]
writes: [state.approval]
reason_code: AUTO_APPROVED
```

```step
id: generate_welcome_pack
type: skill
description: Create a personalised welcome letter, setup checklist, and meeting invitation
reads: [state.validated_intake, state.approval, state.research_results]
writes: [state.welcome_pack]
agent: account_planner
output_files: [welcome-letter.pdf, setup-checklist.pdf, service-agreement.pdf]
expected_output: |
  A professional welcome pack containing:
  - Personalised welcome letter addressing the client by name
  - Step-by-step setup checklist tailored to their service tier
  - Draft service agreement with recommended components
  Tone should be warm, professional, and specific to their industry.
reason_code: PACK_GENERATED
reason_code_on_fail: PACK_GENERATION_FAILED
```

```step
id: send_notifications
type: tool
description: Email the welcome pack to the client and notify the internal account team
reads: [state.welcome_pack, state.validated_intake]
writes: [state.notifications]
tool: email.send
on_error: retry
reason_code: NOTIFICATIONS_SENT
reason_code_on_fail: NOTIFICATION_FAILED
```

```step
id: schedule_kickoff
type: tool
description: Create a kickoff meeting in the team calendar for the first week
reads: [state.validated_intake, state.approval]
writes: [state.meeting]
tool: calendar.create
on_error: skip
reason_code: MEETING_SCHEDULED
reason_code_on_fail: SCHEDULING_SKIPPED
```

```step
id: finalise_onboarding
type: end
description: Record the completed onboarding and assemble the final output
reads: [state.welcome_pack, state.notifications, state.meeting]
writes: [output]
output_files: [onboarding-summary.pdf]
stop_condition: output.onboarding_pack != null
reason_code: ONBOARDING_COMPLETE
```

## Bundles

```bundle
name: research_pack
version: 1.0

budgets:
  max_steps_per_worker: 3
  deadline_seconds_per_worker: 60
  max_tokens_per_worker: 8000

workers:
  - id: W1_COMPLIANCE
    agent: compliance_checker
  - id: W2_MARKET
    agent: market_researcher
  - id: W3_PLANNING
    agent: account_planner
    when: service_tier != "starter"

merge:
  strategy: combine_by_type
  dedupe_key: [type, label]
  conflict: send_to_critic

worker_output:
  format: agent-flow/worker.v1
  required_fields: [type, items, confidence]
```

## Runtime

```runtime
checkpoint_after_each_step: true
resume_supported: true
max_concurrency: 3
approval_required: true
human_in_the_loop: true

checkpoints:
  - every: 3_steps

waitpoints:
  - id: WP_CLIENT_APPROVAL
    after_step: review_and_approve
    event: human_approval
    timeout_seconds: 86400
    on_timeout: notify_then_stop
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
  patterns: ["email", "phone"]
```
