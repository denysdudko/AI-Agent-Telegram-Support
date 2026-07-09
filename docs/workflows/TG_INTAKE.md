# TG Intake

## Purpose

`TG Intake` receives Telegram group messages and turns normal user messages into waiting runtime queues. It is responsible for collecting messages and starting the 60-second escalation timer.

The workflow:

- receives Telegram group messages;
- detects whether the sender is an expert;
- creates or updates a waiting runtime queue for a normal user;
- publishes a RabbitMQ delay event for later escalation checks.

`TG Intake` does not call OpenAI and does not decide whether a message needs escalation. AI analysis is handled later by `TG Escalation`.

## Current Node Sequence

1. `Telegram Trigger`
2. `Normalize Message`
3. `Get a expert`
4. `IF Expert`
5. `answered_by_expert`
6. `Execute a SQL query`
7. `IF Runtime Exists`
8. `Build New Runtime State`
9. `Build Updated Runtime State`
10. `Create Runtime State`
11. `Update Runtime State`
12. `Build RabbitMQ Payload`
13. `Publish`

## Normalized Message Contract

`Normalize Message` converts the Telegram trigger payload into a compact object used by the rest of the workflow.

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

## Expert Handling

`Get a expert` checks `telegram_experts` to determine whether the sender is an expert.

If the sender is an expert:

- mark waiting queues in the same chat as `answered_by_expert`;
- set `expert_replied = true`;
- do not create a user queue;
- do not publish a RabbitMQ delay event.

## Normal User Handling

If the sender is not an expert, the workflow looks for an active waiting runtime queue using the normalized `chat_id`, `user_id`, and `status = waiting`.

If an active waiting runtime exists:

- append the message to `runtime_state.messages[]`;
- increment `queue_version`;
- update `last_message_at`;
- publish a new delay event with the updated `queue_version`.

If no active waiting runtime exists:

- create a new `telegram_runtime_state` record;
- set `status = waiting`;
- set `queue_version = 1`;
- set `expert_replied = false`;
- initialize `runtime_state.messages[]`, `questions[]`, `expert_reply`, and `escalation`;
- publish a delay event for the new runtime queue.

## RabbitMQ Payload

`Build RabbitMQ Payload` sends the minimum data needed by `TG Escalation` after the delay:

```json
{
  "runtime_id": 123,
  "queue_version": 1
}
```

`runtime_id` identifies the database row. `queue_version` protects `TG Escalation` from stale RabbitMQ TTL events when newer user messages have already updated the active queue.

## Design Rules

- `TG Intake` does not call OpenAI.
- `TG Intake` does not decide `needs_escalation`.
- `TG Intake` only collects messages and manages queue/timer state.
- AI analysis is handled by `TG Escalation`.

## Known Risks / Notes

- Current expert reply logic closes all waiting queues in the chat.
- This is acceptable for the MVP but may need refinement later if multiple users wait at the same time.
- Chat IDs and routing keys should be configurable long-term.
