# 02 TG Escalation

## Purpose

`TG Escalation` consumes delayed RabbitMQ events, reloads runtime state, ignores stale events, runs AI analysis, and sends expert/user notifications when escalation is required.

## Configuration

`Load Production Config` contains the MVP configuration object, validates required fields and types, and keeps the config at `$json.config` before runtime loading and AI processing.

This embedded workflow configuration is accepted for the MVP because the current n8n Cloud plan does not provide Custom Variables.

Config-driven values:

- `openai.default_model`
- `telegram.expert_group.chat_id`
- `telegram.parse_mode`
- `languages.default_fallback`

## Expert Message

The expert alert preserves the visible Ukrainian structure with runtime ID, group, user, Telegram ID, topic, questions, original-message link, and message count.

After extracted questions and before the original-message link, it includes a Telegram HTML expandable blockquote:

```html
<blockquote expandable>...</blockquote>
```

The quote is collapsed by default in Telegram clients that support expandable blockquotes. It contains `Повний текст звернення:` and every stored message from `runtime_state.messages[]` in chronological array order. Dynamic Telegram HTML content is escaped locally in `Build Escalation Message`.

## RabbitMQ Trigger Exception

`RabbitMQ Trigger` cannot read config from a previous node because it starts the workflow. For MVP compatibility, its queue is set directly to `tg.escalation`.

This is the only accepted operational literal in the workflow export for Task 013.

## Payload Contract

RabbitMQ payload remains unchanged:

```json
{
  "runtime_id": 123,
  "queue_version": 1
}
```

## Deferred MVP Scope

- Retry strategy is deferred.
- Partial failure protection is deferred.
- Structured logging, monitoring, and alerts are deferred.
