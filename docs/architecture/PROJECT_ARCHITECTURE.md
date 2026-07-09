# Project Architecture

## MVP Goal

The MVP automates first-line support inside Telegram groups while keeping human experts in control of final answers. The agent listens to user messages, waits briefly for an expert response, escalates unanswered messages, and notifies the user when a response is available.

The system is intentionally small: n8n orchestrates the workflows, Supabase/Postgres stores runtime state, RabbitMQ provides the delay timer, and Telegram is the interaction channel.

## Main Pipeline

```text
Telegram Group
  -> TG Intake
  -> Supabase runtime_state
  -> RabbitMQ delay
  -> TG Escalation
  -> Experts notification
  -> User notification
```

1. A user sends a message in a Telegram support group.
2. `TG Intake` records the message and creates or updates one runtime state row.
3. RabbitMQ holds a delayed event for the 60-second waiting window.
4. `TG Escalation` runs after the delay and checks the current runtime state.
5. If no expert has answered, experts are notified.
6. When an expert answer is available, the user is notified in Telegram.

## Architectural Layers

### 1. Orchestration Layer

The orchestration layer is implemented in n8n. It receives Telegram events, coordinates state changes, schedules delayed checks, and sends notifications. The current workflows in this layer are `TG Intake` and `TG Escalation`.

### 2. Retrieval Layer

The retrieval layer is responsible for reading the current state and context required to decide what happens next. For the MVP, this primarily means reading `runtime_state` from Supabase/Postgres before continuing a workflow step.

### 3. Answer Layer

The answer layer handles the final response path. In the MVP, answers come from experts or future AI-assisted logic, then flow back through the orchestration layer to notify the user.

## Single runtime_state Principle

Each support interaction should have one authoritative `runtime_state` record. Workflows must read and update that record instead of passing long-lived state only through transient workflow executions.

This keeps retries, delayed checks, and expert responses aligned around one source of current truth.

## JSON Schema Principle

Structured data exchanged between workflow nodes, database records, and AI steps should follow explicit JSON schemas. Schemas make payloads predictable, easier to validate, and safer to evolve as the agent grows.

## RabbitMQ Delay Timer

RabbitMQ is used for the 60-second timer because the delay must survive outside a single n8n execution. A queued delayed message gives the system a simple handoff point: `TG Intake` can schedule the check, and `TG Escalation` can process it later without keeping an active workflow waiting.

## Supabase/Postgres Source of Truth

Supabase/Postgres is the source of truth because runtime decisions need durable, queryable state. It stores the current status of each support interaction so workflows can recover from retries, delayed execution, or duplicate events without relying on memory or workflow-local data.

## Current Workflows

### TG Intake

`TG Intake` receives Telegram group messages, normalizes the incoming event, writes the interaction into `runtime_state`, and schedules the delayed escalation check through RabbitMQ.

### TG Escalation

`TG Escalation` consumes the delayed check, reads the latest `runtime_state`, decides whether the message still needs attention, and triggers expert or user notifications based on the current status.

## Current Statuses

- `waiting`: the user message is recorded and the system is waiting for an expert answer or the delay timer.
- `answered_by_expert`: an expert has answered and the user notification path can proceed.
- `escalated`: the waiting window expired and the message was escalated to experts.
- `ignored`: the message does not require support handling or should not continue through the pipeline.

## Change Scope

This document describes the current project direction only. It does not introduce runtime, workflow, database, service, or implementation changes.
