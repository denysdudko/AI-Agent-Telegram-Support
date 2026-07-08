# Prompts

## Purpose

The `prompts` directory stores prompt assets and JSON schemas used by the AI agent. It keeps AI behavior, structured output contracts, and user-facing message templates reviewable outside exported workflow JSON.

## Directory Layout

- `user_messages/` stores plain text prompt files and reusable user-facing message templates.
- `schemas/` stores JSON schemas used for structured model output, validation, and integration contracts.

## Storage Rules

- Prompts are stored as plain text files.
- JSON schemas are stored separately from prompts in `schemas/`.
- Prompts are version-controlled in Git.
- Workflow nodes should reference prompt files whenever technically possible instead of embedding long prompt text.
- Every prompt change must be committed together with related documentation updates.

## Versioning

Git history is the primary record of prompt and schema changes. Use commits and diffs to review how AI behavior and structured output contracts evolve over time.
