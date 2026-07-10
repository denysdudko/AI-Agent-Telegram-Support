# 02 TG Escalation

## Purpose

`TG Escalation` consumes delayed RabbitMQ events, reloads runtime state, ignores stale events, runs AI analysis, and sends expert/user notifications when escalation is required.

## Configuration

`RabbitMQ Trigger` reads the escalation queue directly from:

```javascript
JSON.parse($vars.TG_SUPPORT_CONFIG_JSON).rabbitmq.queues.escalation
```

`Load Production Config` then reads the same n8n Variable `TG_SUPPORT_CONFIG_JSON`, parses it with `JSON.parse`, validates required fields and types, and keeps the parsed object at `$json.config` before runtime loading and AI processing.

Config-driven values:

- `openai.default_model`
- `telegram.expert_group.chat_id`
- `telegram.parse_mode`
- `languages.default_fallback`
- `rabbitmq.queues.escalation`

The workflow export does not contain duplicated embedded runtime config values. Changing `TG_SUPPORT_CONFIG_JSON` updates both workflows. If n8n does not apply a RabbitMQ trigger queue change automatically, reactivate `TG Escalation`.

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

