---
inclusion: always
---

# EARS-Based Task Breakdown

This steering file controls how tasks are broken down into specs for spec-driven development.

## Process

Tasks follow this model:

**Requirements → Design Document → Task Breakdown**

## Task Breakdown Format: EARS Notation

All task breakdowns MUST be written using EARS (Easy Approach to Requirements Syntax) — precise, unambiguous requirements.

### Why EARS?

- Eliminates ambiguity
- Machine-parseable by the agent
- Maps directly to test cases
- Reviewable by non-technical stakeholders

### EARS Patterns

| Pattern | Template |
|---------|----------|
| Ubiquitous | "The system shall..." |
| Event-driven | "When [event], the system shall..." |
| State-driven | "While [state], the system shall..." |
| Optional | "Where [feature], the system shall..." |
| Unwanted | "If [condition], then the system shall..." |

## Ambiguity Handling Rules

If any task contains ambiguities or long-winded, difficult tasks:

- **NEVER** guess or make assumptions
- **NEVER** skip tasks or leave for later
- **ALWAYS** grill the user with options and pros/cons of each one so the user can make a decision
