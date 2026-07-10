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
3. `Load Production Config`
4. `Load Runtime State`
5. `IF Status Waiting`
6. `Basic LLM Chain`
7. `OpenAI Chat Model`
8. `Structured Output Parser`
9. `Save Analysis`
10. `IF needs escalation`
11. `Build Escalation Message`
12. `Build User Message`
13. `Build Final Runtime State`
14. `Send to experts`
15. `Send to user`
16. `Mark Escalated`
17. `Mark Ignored`

## Configuration Loading

`RabbitMQ Trigger` uses an expression based on the production config contract default for `rabbitmq.queues.escalation`. Because trigger subscription happens before Code nodes can run, `Load Production Config` runs immediately after `Parse Payload`.

`Load Production Config` validates required config paths before runtime loading, stale-event checks, AI analysis, message building, and Telegram sends.

Config-driven values:

- `openai.default_model`
- `telegram.expert_group.chat_id`
- `telegram.parse_mode`
- `languages.default_fallback`
- `rabbitmq.queues.escalation`

The RabbitMQ payload remains unchanged and still contains only `runtime_id` and `queue_version`.

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
- Operational values, including the expert group chat ID, are now loaded from the production config contract in the workflow export.
