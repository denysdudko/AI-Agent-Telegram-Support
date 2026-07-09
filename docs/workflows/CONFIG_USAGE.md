# Workflow Config Usage

## Purpose

This document explains how `TG Intake` and `TG Escalation` must use the production configuration contract in the next implementation step. It documents the required direction only; current n8n workflow JSON files are not changed by this task.

The configuration contract is defined in `docs/configuration/PRODUCTION_CONFIG.md`, with a JSON-only example at `config/telegram-support-agent.config.example.json`.

## Shared Rules

- Workflows must read one JSON configuration object for operational values.
- Secrets must stay in n8n credentials or environment-specific secret management.
- Config reads must not change the documented architecture: one orchestration layer, one retrieval layer, one answer layer, one `runtime_state`, and JSON schema everywhere.
- Config reads must not introduce Redis, a new service, or a second runtime store.
- Config validation should happen before operational values are used.
- Workflow behavior should remain equivalent to the current defaults unless an explicit future task changes behavior.

## TG Intake Usage

`TG Intake` must read config before it publishes RabbitMQ delay events.

Required config values:

- `telegram.source_groups.enabled`
- `telegram.source_groups.groups[]`
- `rabbitmq.exchanges.support`
- `rabbitmq.queues.delay`
- `rabbitmq.routing_keys.delay`
- `rabbitmq.delay_ttl_ms`
- `environment.name`

Usage requirements:

- If source group filtering is enabled, accept messages only from configured `telegram.source_groups.groups[].chat_id` values.
- Continue to normalize Telegram input into the existing message contract.
- Continue to use Supabase/Postgres as the source of truth for `runtime_state`.
- Continue to publish the same minimal RabbitMQ payload: `runtime_id` and `queue_version`.
- Publish delayed support events through the configured support exchange and delay routing key.
- Keep the 60-second MVP behavior by using the configured default `rabbitmq.delay_ttl_ms = 60000` until a future task changes the value.

Current hardcoded values to replace in `TG Intake`:

- RabbitMQ publish exchange `tg.support` should read from `rabbitmq.exchanges.support`.
- RabbitMQ publish routing key `delay` should read from `rabbitmq.routing_keys.delay`.
- RabbitMQ delay queue/topology value `tg.delay` should be managed from `rabbitmq.queues.delay` and RabbitMQ infrastructure config.
- RabbitMQ delay TTL `60000` should read from `rabbitmq.delay_ttl_ms`.

## TG Escalation Usage

`TG Escalation` must read config before it consumes delayed events, runs AI analysis, builds Telegram messages, or sends notifications.

Required config values:

- `rabbitmq.queues.escalation`
- `rabbitmq.routing_keys.escalation`
- `openai.default_model`
- `languages.supported`
- `languages.default_fallback`
- `telegram.expert_group.chat_id`
- `telegram.parse_mode`
- `environment.name`

Usage requirements:

- Consume delayed events from the configured escalation queue.
- Continue to parse the minimal RabbitMQ payload: `runtime_id` and `queue_version`.
- Continue to reload the latest row from Supabase/Postgres before analysis.
- Continue to ignore stale events where the RabbitMQ `queue_version` does not match the database `queue_version`.
- Continue to analyze only current `waiting` runtime records.
- Use the configured OpenAI model for the existing analysis path.
- Keep structured AI output aligned with configured supported language keys.
- Use `languages.default_fallback` when AI returns an unsupported language.
- Send expert notifications to `telegram.expert_group.chat_id`.
- Use `telegram.parse_mode` for expert and user Telegram messages.

Current hardcoded values to replace in `TG Escalation`:

- RabbitMQ trigger queue `tg.escalation` should read from `rabbitmq.queues.escalation`.
- OpenAI model `gpt-5-mini` should read from `openai.default_model`.
- Expert Telegram chat id `-1003811712654` should read from `telegram.expert_group.chat_id`.
- Telegram parse mode `HTML` should read from `telegram.parse_mode`.
- Supported user notification language keys `uk`, `ru`, `pl`, `en`, and `other` should align with `languages.supported`.
- Fallback language `other` should read from `languages.default_fallback`.

## RabbitMQ Topology Usage

RabbitMQ topology must come from the same config contract:

- Support exchange: `rabbitmq.exchanges.support`, default `tg.support`.
- Dead-letter exchange: `rabbitmq.exchanges.dead_letter`, default `tg.dlx`.
- Delay queue: `rabbitmq.queues.delay`, default `tg.delay`.
- Escalation queue: `rabbitmq.queues.escalation`, default `tg.escalation`.
- Delay routing key: `rabbitmq.routing_keys.delay`, default `delay`.
- Escalation routing key: `rabbitmq.routing_keys.escalation`, default `tg.escalation`.
- Delay TTL: `rabbitmq.delay_ttl_ms`, default `60000`.

## Implementation Notes for Next Task

- Add config loading in n8n using existing workflow capabilities or environment-specific JSON injection.
- Validate the JSON shape before using values in operational nodes.
- Replace literals one node at a time and keep payload contracts unchanged.
- Do not add new AI nodes.
- Do not change database schema.
- Do not change `runtime_state` shape unless a future explicit task requires it.
- Do not store secrets in the JSON config.

