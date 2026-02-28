# Agent Flow

**A layered specification for agentic AI workflows.**

Structure tames LLM non-determinism. Agent Flow defines *what* happens, *when* it proceeds, and *who* does the work — while leaving the LLM free to reason creatively within each step's boundaries.

## The Problem

LLMs are non-deterministic by nature — the same prompt can produce different outputs every time. This is their strength (creativity, reasoning, adaptation) and their weakness (unpredictability, inconsistency, hallucination).

Current approaches either ignore this or try to force determinism, losing what makes LLMs valuable.

## The Solution

Agent Flow applies **structural guardrails** around non-deterministic steps:

| Layer | What's deterministic | What's non-deterministic |
|-------|---------------------|------------------------|
| **Steps** | Order, dependencies, data flow | LLM reasoning within each step |
| **Budgets** | Token limits, time limits, step caps | How the LLM uses its budget |
| **Gates** | When approval is required, who approves | What the reviewer decides |
| **Agents** | Role, tools, output format | How the agent fulfills its role |
| **Audit** | What gets logged, reason codes | — (fully deterministic) |

## Design Principles

1. **Business-friendly first** — A product manager can read and modify a workflow without engineering help
2. **Markdown-native** — Plain text files, version-controlled, diffable, no special tooling required
3. **Layered complexity** — Start simple, add sophistication only when needed
4. **Structure tames chaos** — Budgets, gates, validation, and stop conditions prevent runaway agents
5. **Observable by default** — Every run produces a verifiable audit trail
6. **Backward compatible** — Every valid skill file is a valid Agent Flow workflow (single-step)

## Layers

Agent Flow is a **strict superset** of the [Agent Skills Spec](https://github.com/anthropics/agent-skills-spec). Complexity is opt-in:

| Layer | Audience | What it adds |
|-------|----------|-------------|
| **0 — Skill** | Everyone | A valid SKILL.md with `name`/`description`. One implicit step. |
| **1 — Linear** | Skill authors | Ordered step blocks. Sequential pipeline. |
| **2 — Graph** | Builders | Conditions, parallel bundles, branching. DAG execution. |
| **3 — Long-running** | Platform teams | Checkpoints, waitpoints, signals. Resumable workflows. |

## Quick Example

```yaml
---
name: transcript-to-report
kind: agent-flow/workflow
version: 1.0.0
description: Convert meeting transcript into structured report with actions and risks

budgets:
  max_steps: 10
  max_tokens: 40000
  deadline_seconds: 300
---
```

# Purpose

Turn raw meeting transcripts into structured, actionable reports.

## Steps

```step
id: validate
type: transform
description: Check transcript has required fields
reads: [input]
writes: [state.validated]
on_error: stop
```

```step
id: extract
type: parallel
description: Three specialists extract actions, risks, and themes simultaneously
reads: [state.validated]
writes: [state.extracted]
bundle: extract_pack
```

```step
id: review
type: gate
description: Quality check before final assembly
reads: [state.extracted]
writes: [state.reviewed]
```

```step
id: assemble
type: skill
description: Combine into final report
reads: [state.reviewed]
writes: [output]
```

See the [full specification](SPEC.md) for complete details, step types, agent definitions, bundle configuration, and more.

## Standards We Build On

Agent Flow borrows proven concepts from established standards:

- **[BPMN 2.0](https://www.omg.org/spec/BPMN/2.0/)** — Gateway types for branching (exclusive, parallel, inclusive)
- **[Temporal.io](https://temporal.io/)** — Activity/retry semantics, signals for external events
- **[AWS Step Functions](https://docs.aws.amazon.com/step-functions/)** — Choice/Parallel/Map state types, Catch/Retry model
- **[LangGraph](https://github.com/langchain-ai/langgraph)** — Typed state with reads/writes, conditional edges, checkpoint-based pause/resume
- **[CrewAI](https://github.com/crewAIInc/crewAI)** — Agent persona model (role, goal, tools, expected output)
- **[OpenTelemetry](https://opentelemetry.io/)** — Span model, semantic conventions for trace attributes
- **[JSON Schema](https://json-schema.org/)** — Input/output schema definitions

## Repository Structure

```
sasha-workflow/
├── SPEC.md              # The full specification
├── spec/
│   ├── schema.json      # JSON Schema for validating workflow files
│   └── conventions.md   # Standard reason codes, event types
├── examples/
│   ├── transcript-to-report.md
│   ├── document-creation.md
│   └── simple-skill.md
└── README.md
```

## Status

This specification is a **draft proposal**. We welcome feedback via [issues](https://github.com/context-is-everything/sasha-workflow/issues) and [discussions](https://github.com/context-is-everything/sasha-workflow/discussions).

## License

MIT
