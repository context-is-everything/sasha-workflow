---
name: report-publisher-pipeline
kind: agent-flow/workflow
version: 1.0.0
description: >
  Generate branded business documents from structured page specifications.
  Validates input, plans layouts, renders charts, solves text fitting,
  and exports to multiple formats (PPTX, PDF, HTML, Web).

category: document-processing
icon: FileText
status: active
risk_profile: low

triggers:
  manual: true
  api: true

budgets:
  max_steps: 10
  max_tool_calls: 15
  deadline_seconds: 120

tools:
  allowlist: [filesystem.read, filesystem.write, browser.measure]

input:
  schema:
    type: object
    required: [pagespec_path, design_pack_path]
    properties:
      pagespec_path:
        type: string
        description: Path to PageSpec JSON file
      design_pack_path:
        type: string
        description: Path to design pack directory
      output_dir:
        type: string
        description: Output directory for exported files
        default: ./output
      formats:
        type: array
        items:
          type: string
          enum: [pptx, pdf, html, web]
        default: [pptx, pdf]
      pages:
        type: string
        description: Page range filter (e.g., "1-5")

output:
  schema:
    type: object
    required: [exports]
    properties:
      exports:
        type: array
        items:
          type: object
          properties:
            format: { type: string }
            success: { type: boolean }
            output_path: { type: string }
            error: { type: string }
      timing:
        type: object
        properties:
          planning_ms: { type: integer }
          solving_ms: { type: integer }
          charts_ms: { type: integer }
          export_ms: { type: integer }
          total_ms: { type: integer }
---

# Purpose

Generate polished, branded business documents from structured data. This is a
deterministic pipeline — no LLM agents involved. It demonstrates that Agent Flow
works for traditional processing pipelines, not just agentic AI workflows.

The pipeline takes a PageSpec (structured content) and a Design Pack (brand
configuration) and produces publication-ready documents in multiple formats.

## Steps

```step
id: validate_input
type: transform
description: Validate PageSpec JSON and design pack structure
reads: [input]
writes: [state.validated]
on_error: stop
reason_code: INPUT_VALIDATED
reason_code_on_fail: INVALID_INPUT
```

```step
id: load_design_pack
type: tool
description: Load and parse the design pack configuration (palette, tokens, layouts, grammar)
tool: filesystem.read
reads: [input.design_pack_path]
writes: [state.design_pack]
on_error: stop
reason_code: DESIGN_PACK_LOADED
reason_code_on_fail: DESIGN_PACK_ERROR
```

```step
id: filter_pages
type: transform
description: Apply page range filter if specified
reads: [state.validated, input.pages]
writes: [state.pages]
when: input.pages != null
reason_code: PAGES_FILTERED
```

```step
id: plan_layouts
type: transform
description: Match page content to layouts using grammar rules, resolve slot regions to absolute coordinates
reads: [state.pages, state.design_pack]
writes: [state.planned]
on_error: stop
reason_code: LAYOUTS_PLANNED
reason_code_on_fail: PLANNING_ERROR
```

```step
id: render_charts
type: tool
description: Render Chart.js configurations to PNG images
tool: browser.measure
reads: [state.planned]
writes: [state.with_charts]
reason_code: CHARTS_RENDERED
reason_code_on_fail: CHART_RENDER_ERROR
```

```step
id: solve_fit
type: transform
description: Find optimal font sizes and spacing using beam search with headless browser measurement
reads: [state.with_charts, state.design_pack]
writes: [state.solved]
on_error: fallback
fallback: solve_fit_simple
reason_code: FIT_SOLVED
reason_code_on_fail: FIT_FAILED
```

```step
id: solve_fit_simple
type: transform
description: Fallback fitting using default font sizes without measurement
reads: [state.with_charts, state.design_pack]
writes: [state.solved]
reason_code: FIT_SOLVED_SIMPLE
```

```step
id: repair_overflow
type: transform
description: Handle any remaining text overflow by tightening spacing or creating continuation pages
reads: [state.solved]
writes: [state.repaired]
reason_code: OVERFLOW_REPAIRED
```

```step
id: export
type: parallel
description: Export to all requested formats simultaneously
reads: [state.repaired, input.formats, input.output_dir]
writes: [output]
bundle: export_pack
reason_code: EXPORT_COMPLETE
reason_code_on_fail: EXPORT_FAILED
```

## Bundles

```bundle
name: export_pack
version: 1.0

budgets:
  deadline_seconds_per_worker: 30

workers:
  - id: W_PPTX
    role: export_pptx
    when: input.formats contains "pptx"
  - id: W_PDF
    role: export_pdf
    when: input.formats contains "pdf"
  - id: W_HTML
    role: export_html
    when: input.formats contains "html"
  - id: W_WEB
    role: export_web
    when: input.formats contains "web"

merge:
  strategy: combine_by_type
  conflict: fail

worker_output:
  format: agent-flow/worker.v1
  required_fields: [format, success, output_path]
```

## Observability

```observability
audit_level: full
log_format: ndjson
required_events:
  - run_start
  - step_start
  - step_complete
  - step_skipped
  - budget_check
  - run_complete
  - run_failed
```
