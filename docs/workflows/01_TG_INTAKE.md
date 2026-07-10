# 01 TG Intake

## Purpose

`TG Intake` receives Telegram group messages, detects experts, creates or updates waiting runtime state, and publishes the delayed RabbitMQ check.

## Configuration

`Load Production Config` reads the n8n Variable `TG_SUPPORT_CONFIG_JSON`, parses it with `JSON.parse`, validates required fields and types, and keeps the parsed object at `$json.config`.

`Publish` reads:

- `rabbitmq.exchanges.support`
- `rabbitmq.routing_keys.delay`

The workflow export does not contain duplicated embedded runtime config values. Changing `TG_SUPPORT_CONFIG_JSON` updates this workflow without changing the export.

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

