# Database Schema

## Purpose

This document describes the current database tables used by the Telegram Support Agent MVP. It documents existing workflow usage only and does not introduce schema, migration, workflow, or runtime changes.

Column types are marked as inferred where they are not explicitly defined in repository SQL.

## Tables

### telegram_runtime_state

`telegram_runtime_state` stores one support interaction, also treated as a queue session. It is the durable source for chat, user, queue, and support status while Telegram messages move through `TG Intake`, RabbitMQ delay, and `TG Escalation`.

Known fields from workflow usage:

- `id`: runtime record identifier, used by RabbitMQ payloads and escalation lookup. Type inferred.
- `chat_id`: Telegram chat identifier. Type inferred.
- `chat_title`: Telegram chat title. Type inferred.
- `user_id`: Telegram message author identifier. Type inferred.
- `user_name`: display name built from Telegram user fields. Type inferred.
- `user_language`: optional language value. Type inferred.
- `status`: queue status. Type inferred.
- `queue_version`: numeric version for the active queue. Type inferred.
- `expert_replied`: boolean flag showing whether an expert replied.
- `queue_started_at`: timestamp for when the queue session started. Type inferred.
- `last_message_at`: timestamp for the latest user message. Type inferred.
- `runtime_state`: JSONB payload used for message history, analysis, expert reply state, and escalation state.
- `updated_at`: timestamp updated by workflow SQL statements. Type inferred.

The table keeps history of user interactions. Active queues are found by filtering for the current waiting state, not by assuming each user has only one lifetime row.

### telegram_experts

`telegram_experts` stores Telegram user IDs for experts. `TG Intake` uses it to decide whether the message author is an expert before treating a message as a user support request.

Known fields from workflow usage:

- Telegram user identifier for expert lookup. Exact column name and type are inferred from workflow usage.

## Known Statuses

- `waiting`: the user interaction is active and waiting for expert response or escalation.
- `answered_by_expert`: an expert replied while the queue was waiting.
- `escalated`: the delayed escalation check found that the interaction still needed expert attention.
- `ignored`: the interaction was evaluated and does not continue through the escalation path.

## queue_version

`queue_version` increments on every new user message in an active waiting queue. `TG Intake` sends the current `queue_version` with the RabbitMQ delay payload.

`TG Escalation` compares the delayed payload version with the latest database value. If they differ, the delayed event is stale and must not escalate the old state.

## runtime_state JSONB

`runtime_state` stores mutable per-interaction context as JSONB. The current workflows use these top-level areas:

- `messages[]`: Telegram messages collected in the queue session.
- `questions[]`: extracted or tracked questions.
- `analysis`: AI analysis output used by escalation decisions.
- `expert_reply`: expert reply detection and related state.
- `escalation`: notification state for experts and users.

Example:

```json
{
  "messages": [
    {
      "message_id": 12345,
      "text": "I need help with my account.",
      "timestamp": "2026-07-09T10:00:00.000Z"
    }
  ],
  "questions": [],
  "analysis": {
    "language": "en",
    "topic": "Account support",
    "questions": ["I need help with my account."],
    "needs_escalation": true,
    "analyzed_at": "2026-07-09T10:01:00.000Z"
  },
  "expert_reply": {
    "detected": false
  },
  "escalation": {
    "sent_to_experts": false,
    "sent_to_user": false
  }
}
```

## Data Rules

- Keep one active waiting queue per `chat_id` + `user_id`.
- Historical interactions must remain stored.
- Do not rely on `UNIQUE(chat_id, user_id)`.
- Active queue lookup must use `chat_id` + `user_id` + `status = waiting`.

## Migration Notes

- This task documents the current schema only.
- Do not create migrations yet.
- Future schema changes must be explicit tasks.
