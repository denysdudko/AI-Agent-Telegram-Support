# Workflow Config Usage

## Purpose

This document explains how `TG Intake` and `TG Escalation` use the production configuration contract in the n8n workflow exports.

The configuration contract is defined in `docs/configuration/PRODUCTION_CONFIG.md`, with a JSON-only example at `config/telegram-support-agent.config.example.json`.

## Implemented Approach

Both workflow exports include a `Load Production Config` Code node. The node loads the JSON contract defaults from `config/telegram-support-agent.config.example.json`, validates required paths, and exposes the config as `config` for downstream nodes.

Validation checks:

- required top-level keys: `environment`, `telegram`, `rabbitmq`, `openai`, and `languages`;
- required RabbitMQ keys for exchanges, queues, routing keys, and delay TTL;
- required expert Telegram group chat id;
- required Telegram parse mode;
- required OpenAI default model;
- required fallback language.

If validation fails, the Code node throws a clear `Production config validation failed` error and stops the workflow before downstream operational steps run.

`TG Escalation` has one trigger-specific exception: the RabbitMQ Trigger queue is evaluated before any Code node can run. Its queue field is still expression-based and uses the same config contract default for `rabbitmq.queues.escalation`; the validation node runs immediately after payload parsing and before runtime loading, AI analysis, or Telegram sends.

## Shared Rules

- Workflows must read one JSON configuration object for operational values.
- Secrets must stay in n8n credentials or environment-specific secret management.
- Config reads must not change the documented architecture: one orchestration layer, one retrieval layer, one answer layer, one `runtime_state`, and JSON schema everywhere.
- Config reads must not introduce Redis, a new service, or a second runtime store.
- Config validation happens in the `Load Production Config` Code node.
- Workflow behavior should remain equivalent to the current defaults unless an explicit future task changes behavior.

## TG Intake Usage

`TG Intake` reads config immediately after `Normalize Message` and before expert lookup, runtime lookup, runtime creation/update, and RabbitMQ publishing.

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

Implemented replacements in `TG Intake`:

- RabbitMQ publish exchange now reads from `rabbitmq.exchanges.support`.
- RabbitMQ publish routing key now reads from `rabbitmq.routing_keys.delay`.
- RabbitMQ payload remains unchanged: `runtime_id` and `queue_version`.

## TG Escalation Usage

`TG Escalation` reads config after `Parse Payload` and before `Load Runtime State`, stale-event checks, AI analysis, message building, or Telegram sends.

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

Implemented replacements in `TG Escalation`:

- RabbitMQ trigger queue uses an expression based on the config contract default for `rabbitmq.queues.escalation`.
- OpenAI model now reads from `openai.default_model`.
- Expert Telegram chat id now reads from `telegram.expert_group.chat_id`.
- Expert and user Telegram parse mode now reads from `telegram.parse_mode`.
- User notification fallback language now reads from `languages.default_fallback`.
- RabbitMQ payload remains unchanged: `runtime_id` and `queue_version`.

## RabbitMQ Topology Usage

RabbitMQ topology must come from the same config contract:

- Support exchange: `rabbitmq.exchanges.support`, default `tg.support`.
- Dead-letter exchange: `rabbitmq.exchanges.dead_letter`, default `tg.dlx`.
- Delay queue: `rabbitmq.queues.delay`, default `tg.delay`.
- Escalation queue: `rabbitmq.queues.escalation`, default `tg.escalation`.
- Delay routing key: `rabbitmq.routing_keys.delay`, default `delay`.
- Escalation routing key: `rabbitmq.routing_keys.escalation`, default `tg.escalation`.
- Delay TTL: `rabbitmq.delay_ttl_ms`, default `60000`.

## Remaining Notes

- The current implementation embeds the JSON example contract defaults in each workflow export through the `Load Production Config` node.
- A later deployment task may replace the embedded defaults with environment-specific JSON injection while keeping the same contract shape.
- Do not add new AI nodes.
- Do not change database schema.
- Do not change `runtime_state` shape unless a future explicit task requires it.
- Do not store secrets in the JSON config.
