# Data Contracts

## Purpose

This document defines MVP v1 data contracts used across the Telegram Support Agent pipeline. Contracts are aligned with current `TG Intake` and `TG Escalation` workflow behavior and do not introduce workflow, database, runtime, or service changes.

## Shared Values

Allowed `language` values:

- `ru`
- `uk`
- `pl`
- `en`
- `other`

Known runtime status values:

- `waiting`
- `answered_by_expert`
- `escalated`
- `ignored`

## 1. Normalized Telegram Message

Purpose: compact Telegram message shape used by `TG Intake` after the trigger payload is normalized.

Producer: `TG Intake` / `Normalize Message`

Consumer: `TG Intake` expert detection, runtime lookup, runtime creation, runtime update, and RabbitMQ publishing steps.

Required fields:

- `chat_id`
- `chat_title`
- `chat_type`
- `message_id`
- `user_id`
- `first_name`
- `last_name`
- `text`
- `telegram_date`
- `received_at`

Example:

```json
{
  "chat_id": -1001234567890,
  "chat_title": "Support Group",
  "chat_type": "supergroup",
  "message_id": 12345,
  "user_id": 987654321,
  "first_name": "Alex",
  "last_name": "User",
  "text": "I need help with my order.",
  "telegram_date": 1720000000,
  "received_at": "2026-07-09T10:00:00.000Z"
}
```

## 2. Runtime State

Purpose: shared state object stored in Supabase/Postgres as `telegram_runtime_state.runtime_state`. It carries message history, AI analysis, expert reply state, and escalation notification state.

Producer: `TG Intake` creates and updates the initial runtime state; `TG Escalation` adds analysis and escalation state.

Consumer: `TG Escalation`, database updates, expert notification builder, and user notification builder.

Required fields:

- `messages[]`
- `questions[]`
- `analysis`
- `expert_reply`
- `escalation`

Example:

```json
{
  "messages": [
    {
      "message_id": 12345,
      "text": "I need help with my order.",
      "timestamp": "2026-07-09T10:00:00.000Z"
    }
  ],
  "questions": [],
  "analysis": {
    "language": "en",
    "topic": "Order support",
    "questions": ["I need help with my order."],
    "needs_escalation": true,
    "analyzed_at": "2026-07-09T10:01:00.000Z"
  },
  "expert_reply": {
    "detected": false
  },
  "escalation": {
    "sent_to_experts": true,
    "sent_to_user": true,
    "sent_at": "2026-07-09T10:01:10.000Z"
  }
}
```

## 3. RabbitMQ Delay Payload

Purpose: minimal delayed trigger payload passed from `TG Intake` to `TG Escalation` after the waiting window.

Producer: `TG Intake` / `Build RabbitMQ Payload`

Consumer: `TG Escalation` / `Parse Payload` and `Load Runtime State`

Required fields:

- `runtime_id`
- `queue_version`

Example:

```json
{
  "runtime_id": 123,
  "queue_version": 2
}
```

## 4. AI Analysis Output

Purpose: structured OpenAI output used by `TG Escalation` to decide whether a support request needs expert escalation.

Producer: `TG Escalation` / `Basic LLM Chain` with `Structured Output Parser`

Consumer: `TG Escalation` / `Save Analysis`, `IF needs escalation`, expert notification builder, and user notification builder.

Required fields:

- `language`
- `topic`
- `questions[]`
- `needs_escalation`

Example:

```json
{
  "language": "en",
  "topic": "Order support",
  "questions": ["How can I check my order status?"],
  "needs_escalation": true
}
```

## 5. Expert Notification Context

Purpose: data shape used to build the Ukrainian expert-facing Telegram notification.

Producer: `TG Escalation` / `Build Escalation Message`

Consumer: `TG Escalation` / `Send to experts`

Required fields:

- `id`
- `chat_title`
- `user_name`
- `user_id`
- `topic`
- `questions[]`
- `message_link`
- `message_count`
- `expert_message`

Example:

```json
{
  "id": 123,
  "chat_title": "Support Group",
  "user_name": "Alex User",
  "user_id": 987654321,
  "topic": "Order support",
  "questions": ["How can I check my order status?"],
  "message_link": "https://t.me/c/1234567890/12345",
  "message_count": 2,
  "expert_message": "<b>Group:</b>\nSupport Group\n\n<blockquote expandable><b>Повний текст звернення:</b>\n\n1. I need help with my order.</blockquote>"
}
```

## 6. User Notification Context

Purpose: data shape used to build the user-facing Telegram notification after escalation.

Producer: `TG Escalation` / `Build User Message`

Consumer: `TG Escalation` / `Send to user`

Required fields:

- `chat_id`
- `language`
- `topic`
- `user_message`
- `reply_to_message_id`

Example:

```json
{
  "chat_id": -1001234567890,
  "language": "en",
  "topic": "Order support",
  "user_message": "Thank you for contacting us. Your inquiry regarding <b>Order support</b> has been forwarded to our support team.",
  "reply_to_message_id": 12346
}
```

## JSON Schema Examples

### AI Analysis Output

```json
{
  "type": "object",
  "required": ["language", "topic", "questions", "needs_escalation"],
  "additionalProperties": false,
  "properties": {
    "language": {
      "type": "string",
      "enum": ["ru", "uk", "pl", "en", "other"]
    },
    "topic": {
      "type": "string"
    },
    "questions": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "needs_escalation": {
      "type": "boolean"
    }
  }
}
```

### RabbitMQ Delay Payload

```json
{
  "type": "object",
  "required": ["runtime_id", "queue_version"],
  "additionalProperties": false,
  "properties": {
    "runtime_id": {
      "type": "integer"
    },
    "queue_version": {
      "type": "integer",
      "minimum": 1
    }
  }
}
```

### runtime_state.analysis

```json
{
  "type": "object",
  "required": ["language", "topic", "questions", "needs_escalation", "analyzed_at"],
  "additionalProperties": false,
  "properties": {
    "language": {
      "type": "string",
      "enum": ["ru", "uk", "pl", "en", "other"]
    },
    "topic": {
      "type": "string"
    },
    "questions": {
      "type": "array",
      "items": {
        "type": "string"
      }
    },
    "needs_escalation": {
      "type": "boolean"
    },
    "analyzed_at": {
      "type": "string",
      "format": "date-time"
    }
  }
}
```

## Contract Rules

- All AI outputs must be structured JSON.
- `runtime_state` is the shared state object across the pipeline.
- RabbitMQ payload must stay minimal.
- Downstream nodes must reload state from Supabase/Postgres instead of trusting RabbitMQ payload as full state.
- User-facing text must not be generated in `TG Intake`.
- Expert-facing text must be built only after AI analysis.

## Versioning Notes

- Contracts are MVP v1.
- Future breaking changes must be explicit tasks.
- Avoid silent changes to field names.
