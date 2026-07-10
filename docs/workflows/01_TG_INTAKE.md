# 01 TG Intake

## Purpose

`TG Intake` receives Telegram group messages, detects experts, creates or updates waiting runtime state, and publishes the delayed RabbitMQ check.

## Configuration

`Load Production Config` contains the MVP configuration object, validates required fields and types, and keeps the config at `$json.config`.

This embedded workflow configuration is accepted for the MVP because the current n8n Cloud plan does not provide Custom Variables.

`Publish` reads:

- `rabbitmq.exchanges.support`
- `rabbitmq.routing_keys.delay`

## Payload Contract

RabbitMQ payload remains unchanged:

```json
{
  "runtime_id": 123,
  "queue_version": 1
}
```

## Scope

- No new AI nodes.
- No database schema changes.
- No `runtime_state` shape changes.
- No user-facing or expert-facing message changes.

