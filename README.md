# Event Property Context Agent

`event-property-context-agent` is Project 1 of a broader **Tacit to Explicit Knowledge** effort.

The goal is to turn implicit analytics context around platform event properties into structured, reviewable knowledge.

## Problem

Event repositories and Copilot-assisted code lookup can often explain what an event or property is. What is harder to discover is how that property is actually used in analytics:

- which metrics reference it,
- which analysis topics depend on it,
- whether it is used as a filter, segment, join key, CASE condition, or aggregation input,
- what business context can be inferred from SQL usage,
- and what still needs human review.

This project focuses on formalizing that missing context.

## MVP

Given:

1. access to a specific event definition repository, and
2. one BigQuery SQL file or SQL text,

produce property-centered candidate knowledge as:

- canonical YAML knowledge cards,
- human-readable Markdown review documents,
- source/evidence references,
- confidence levels,
- review status,
- and open human-review questions.

SQL-derived interpretation is always treated as candidate knowledge until reviewed.

## Core flow

```text
connect specific event repository
→ understand event
→ understand event properties
→ identify missing analytics context
→ inspect BigQuery SQL / analytics code usage
→ generate property context cards
→ human review
→ approved knowledge only after review
```

## Current repository contents

- `agent-prd.md` — product requirements
- `knowledge-schema.md` — YAML knowledge schema
- `eval-spec.md` — evaluation criteria
- `tool-contracts.md` — internal tool/function contracts
- `failure-cases.md` — expected failure modes and handling
- `cost-and-caching.md` — token/caching strategy
- `implementation-plan.md` — phased implementation plan

## Scope guard

This repository should not contain real company-internal event definitions, SQL, dashboards, or customer data. Use sample or anonymized fixtures only.

## Status

Design/specification phase. Implementation has not started yet.
