---
name: document-creation-pipeline
kind: agent-flow/workflow
version: 1.0.0
description: >
  Create a polished business document from a brief — with research,
  drafting, editorial review, and stakeholder approval before publishing.

category: document-processing
icon: FileText
status: active
risk_profile: high

budgets:
  max_steps: 15
  max_tool_calls: 10
  max_tokens: 80000
  deadline_seconds: 1800

triggers:
  manual: true
  api: true

tools:
  allowlist: [drive.read, drive.write, web.search, pdf.extract]
  workers_can_call_tools: true

input:
  schema:
    type: object
    required: [brief, doc_type]
    properties:
      brief:
        type: string
        description: What the document should cover
      doc_type:
        type: string
        enum: [proposal, report, memo, presentation]
      tone:
        type: string
        default: professional
      audience:
        type: string
        description: Who will read this document
      reference_docs:
        type: array
        items: { type: string, format: uri }
        description: URLs or paths to reference materials

output:
  schema:
    type: object
    required: [document]
    properties:
      document:
        type: object
        properties:
          title: { type: string }
          sections: { type: array }
          word_count: { type: integer }
      metadata:
        type: object
        properties:
          sources_used: { type: integer }
          review_rounds: { type: integer }
---

# Purpose

End-to-end document creation: gather context, research, draft, review, revise, get stakeholder approval, and publish. Designed for business teams who need reliable, high-quality document production with human oversight at key stages.

The workflow pauses at stakeholder approval — the reviewer can approve, reject with feedback, or request revisions. This is a long-running workflow that may span hours or days.

## Agents

```agent
id: researcher
role: Gather context and reference material for document creation
goal: Find relevant data, precedents, and supporting material from available sources
tools: [web.search, drive.read, pdf.extract]
max_tokens: 8000
expected_output: |
  Object with fields:
  - sources: list of { title, url, key_points, relevance_score }
  - summary: overall research findings narrative
  - gaps: topics in the brief that need more information
```

```agent
id: drafter
role: Write document content based on brief and research
goal: Produce a complete, well-structured draft matching the requested type and tone
max_tokens: 15000
expected_output: |
  Object with fields:
  - title: document title
  - sections: list of { heading, content }
  - word_count: total words
```

```agent
id: editor
role: Review draft for clarity, accuracy, tone, and completeness
goal: Improve quality without changing meaning — flag any factual concerns
max_tokens: 8000
expected_output: |
  Object with fields:
  - revised_sections: list of { heading, content } (improved versions)
  - changes: list of { section, original, revised, reason }
  - quality_score: 0.0 to 1.0
  - concerns: list of flagged issues for stakeholder attention
```

## Steps

```step
id: validate_brief
type: transform
description: Check the brief has enough detail to produce the requested document type
reads: [input]
writes: [state.validated]
on_error: stop
reason_code: BRIEF_VALIDATED
reason_code_on_fail: BRIEF_INSUFFICIENT
```

```step
id: gather_references
type: tool
description: Fetch any reference documents provided in the brief
tool: drive.read
reads: [input.reference_docs]
writes: [state.references]
when: input.reference_docs.length > 0
retry:
  max_attempts: 2
  backoff_ms: [500, 2000]
reason_code: REFERENCES_GATHERED
reason_code_on_fail: REFERENCES_UNAVAILABLE
```

```step
id: research
type: skill
description: Research the topic using web search and reference materials
agent: researcher
reads: [state.validated, state.references]
writes: [state.research]
reason_code: RESEARCH_COMPLETE
```

```step
id: draft
type: skill
description: Write the first complete draft
agent: drafter
reads: [state.validated, state.research]
writes: [state.draft]
expected_output: |
  Complete document matching the doc_type, tone, and audience
  specified in the brief. Must address all points raised.
reason_code: DRAFT_COMPLETE
```

```step
id: edit
type: skill
description: Editor reviews and improves the draft
agent: editor
reads: [state.draft, state.validated]
writes: [state.edited]
reason_code: EDIT_COMPLETE
```

```step
id: stakeholder_approval
type: gate
description: Stakeholder reviews the edited document before publishing
reads: [state.edited]
writes: [state.approved]
gate_method: human_review
reason_code: APPROVED
reason_code_on_fail: REJECTED
```

```step
id: publish
type: tool
description: Write the approved document to the destination
tool: drive.write
reads: [state.approved]
writes: [output]
when: state.approved.gate_result == "approved"
reason_code: PUBLISHED
reason_code_on_fail: PUBLISH_FAILED
```

## Runtime

```runtime
checkpoint_after_each_step: true
resume_supported: true

waitpoints:
  - id: WP_APPROVAL
    after_step: stakeholder_approval
    event: human_approval
    timeout_seconds: 172800
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
```
