# Workflows

## Purpose

The `workflows` directory stores exported n8n workflow JSON files that are part of the Telegram support automation runtime. These files are the source-controlled copies used to review, deploy, and restore workflow definitions.

## Workflow Files

- `TG Intake.json` contains the Telegram intake workflow responsible for receiving and normalizing incoming support messages.
- `TG Escalation.json` contains the escalation workflow responsible for routing cases that require handoff or follow-up.
- `archive/` stores previous workflow exports only when an archived version is explicitly requested.

## Naming Convention

Workflow filenames use stable, human-readable names that match the workflow name in n8n:

```text
<Domain> <Purpose>.json
```

Examples:

- `TG Intake.json`
- `TG Escalation.json`

Workflow filenames must remain stable. Do not rename workflow files unless the workflow itself is intentionally renamed and the related documentation is updated in the same change.

## Maintenance Rules

- Every workflow change must be committed together with related documentation changes.
- Exported workflows must come directly from n8n without manual JSON editing.
- Do not move, rename, or reformat workflow JSON files as part of unrelated changes.
- Store previous workflow versions in `workflows/archive/` only when explicitly requested.

## Versioning

Git history is the primary version control mechanism for workflows. Use commits, diffs, and tags to inspect workflow changes over time. The `archive/` directory is not a replacement for Git history; it is reserved for explicitly requested snapshots.
