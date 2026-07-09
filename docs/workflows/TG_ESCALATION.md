# TG Escalation

## Purpose

`TG Escalation` processes delayed RabbitMQ events created by `TG Intake`. It reloads the latest runtime state, ignores stale timer events, analyzes the collected user messages with AI, and decides whether the support interaction should be escalated.

The workflow:

- consumes delayed RabbitMQ events from `tg.escalation`;
- loads the latest `telegram_runtime_state` row;
- continues only for current waiting queues;
- runs AI analysis with structured output;
- decides whether escalation is required;
- notifies experts and the user when needed;
- marks the runtime as `escalated` or `ignored`.

## Current Node Sequence

1. `RabbitMQ Trigger`
2. `Parse Payload`
3. `Load Runtime State`
4. `IF Status Waiting`
5. `Basic LLM Chain`
6. `OpenAI Chat Model`
7. `Structured Output Parser`
8. `Save Analysis`
9. `IF needs escalation`
10. `Build Escalation Message`
11. `Build User Message`
12. `Build Final Runtime State`
13. `Send to experts`
14. `Send to user`
15. `Mark Escalated`
16. `Mark Ignored`

## RabbitMQ Input Payload

`TG Escalation` expects a minimal delayed payload:

```json
{
  "runtime_id": 123,
  "queue_version": 1
}
```

- `runtime_id` identifies the `telegram_runtime_state` row to reload.
- `queue_version` identifies the queue version that existed when the delay event was published.

## Stale Event Protection

The workflow protects against old RabbitMQ TTL events before running AI analysis:

- compare the RabbitMQ `queue_version` with the database `queue_version`;
- continue only if both versions match;
- continue only if the database `status = waiting`.

If a user sends a newer message while the queue is still waiting, `TG Intake` increments `queue_version`. Older delayed events then become stale and should not trigger escalation.

## AI Structured Output Schema

The AI step returns structured JSON through the output parser:

```json
{
  "language": "en",
  "topic": "Account access issue",
  "questions": ["How can I restore account access?"],
  "needs_escalation": true
}
```

Fields:

- `language`: detected user language, currently expected as `ru`, `uk`, `pl`, `en`, or `other`.
- `topic`: short business topic for the request.
- `questions[]`: concise extracted questions or converted problem/request statements.
- `needs_escalation`: boolean decision for whether expert escalation is required.

## needs_escalation Logic

`needs_escalation = false` for:

- greetings;
- thanks;
- acknowledgements;
- small talk;
- messages without a request, problem, or question.

`needs_escalation = true` for:

- questions;
- reported problems;
- help requests;
- information requests;
- support requests.

## runtime_state Update

`Save Analysis` stores AI output in `runtime_state.analysis`:

- `analysis.language`
- `analysis.topic`
- `analysis.questions`
- `analysis.needs_escalation`
- `analysis.analyzed_at`

When escalation is required, `Build Final Runtime State` stores notification state in `runtime_state.escalation`:

- `escalation.sent_to_experts`
- `escalation.sent_to_user`
- `escalation.sent_at`

`Mark Escalated` sets the database status to `escalated`. `Mark Ignored` sets the database status to `ignored` when escalation is not needed.

## Expert Notification

The expert notification is built in Ukrainian and uses Telegram HTML parse mode. It includes:

- ticket id;
- group title;
- user display name;
- Telegram user ID;
- topic;
- questions;
- message link;
- message count.

The message is sent to the expert group chat.

## User Notification

The user notification is selected from predefined templates based on detected user language. Supported template keys are `uk`, `ru`, `pl`, `en`, and `other`.

The notification:

- explains that the request was forwarded to support;
- includes the detected topic;
- replies to the latest user message;
- uses Telegram HTML parse mode.

## Known Risks / Notes

- `Mark Escalated` currently can run in parallel with `Send to experts` and `Send to user` because all three branches start from `IF needs escalation`.
- This may mark runtime as `escalated` even if one Telegram send fails.
- This is acceptable for MVP documentation, but should be refined later.
- The expert group chat ID is currently hardcoded and should become configurable later.
