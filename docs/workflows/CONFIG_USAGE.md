# Workflow Config Usage

## Purpose

This document explains how `TG Intake` and `TG Escalation` use the production configuration contract in the n8n workflow exports.

The configuration contract is defined in `docs/configuration/PRODUCTION_CONFIG.md`, with a JSON-only example at `config/telegram-support-agent.config.example.json`.

## MVP Implementation

The current n8n Cloud plan does not provide Custom Variables. Custom Variables require Pro Cloud or Self-hosted Enterprise, so workflows cannot rely on runtime variable reads for configuration in the current deployment.

For MVP compatibility, each workflow export keeps the production configuration object embedded in its existing `Load Production Config` Code node. This is the accepted MVP implementation.

`Load Production Config`:

- contains a JSON object matching `config/telegram-support-agent.config.example.json`;
- validates required fields and types;
- exposes the parsed config at `$json.config`;
- stops the workflow with a clear `Production config validation failed` error if validation fails.

The workflow exports do not contain environment-specific secrets. API keys, bot tokens, database passwords, RabbitMQ URLs, and credential names must remain in n8n credentials or secret management.

## Validation Checks

Validation checks:

- `environment.name` is a non-empty string;
- `telegram.expert_group.chat_id` is an integer;
- `telegram.parse_mode` is a non-empty string;
- RabbitMQ exchange, queue, and routing key values are non-empty strings;
- `rabbitmq.delay_ttl_ms` is a positive integer;
- `openai.default_model` is a non-empty string;
- `languages.supported` is a non-empty array;
- `languages.default_fallback` is included in `languages.supported`.

## TG Intake Usage

`TG Intake` reads config immediately after `Normalize Message` and before expert lookup, runtime lookup, runtime creation/update, and RabbitMQ publishing.

Required config values:

- `rabbitmq.exchanges.support`
- `rabbitmq.routing_keys.delay`
- `rabbitmq.delay_ttl_ms`
- `environment.name`

Implemented replacements in `TG Intake`:

- RabbitMQ publish exchange reads from `$json.config.rabbitmq.exchanges.support`.
- RabbitMQ publish routing key reads from `$json.config.rabbitmq.routing_keys.delay`.
- RabbitMQ payload remains unchanged: `runtime_id` and `queue_version`.

## TG Escalation Usage

`TG Escalation` reads config after `Parse Payload` and before `Load Runtime State`, stale-event checks, AI analysis, message building, or Telegram sends.

Required config values:

- `rabbitmq.queues.escalation`
- `openai.default_model`
- `languages.supported`
- `languages.default_fallback`
- `telegram.expert_group.chat_id`
- `telegram.parse_mode`
- `environment.name`

Implemented replacements in `TG Escalation`:

- OpenAI model reads from `$json.config.openai.default_model`.
- Expert Telegram chat id reads from `$json.config.telegram.expert_group.chat_id`.
- Expert and user Telegram parse mode reads from `$json.config.telegram.parse_mode`.
- User notification fallback language reads from `$json.config.languages.default_fallback`.
- RabbitMQ payload remains unchanged: `runtime_id` and `queue_version`.

## RabbitMQ Trigger Exception

`RabbitMQ Trigger` cannot read config from a previous node because it starts the workflow. For MVP compatibility, `TG Escalation` sets the trigger queue directly to `tg.escalation`.

This is the only accepted operational literal in the workflow exports for Task 013. It matches `rabbitmq.queues.escalation` in `config/telegram-support-agent.config.example.json`.

## RabbitMQ Topology Defaults

RabbitMQ topology follows the same config contract:

- Support exchange: `rabbitmq.exchanges.support`, default `tg.support`.
- Dead-letter exchange: `rabbitmq.exchanges.dead_letter`, default `tg.dlx`.
- Delay queue: `rabbitmq.queues.delay`, default `tg.delay`.
- Escalation queue: `rabbitmq.queues.escalation`, default `tg.escalation`.
- Delay routing key: `rabbitmq.routing_keys.delay`, default `delay`.
- Escalation routing key: `rabbitmq.routing_keys.escalation`, default `tg.escalation`.
- Delay TTL: `rabbitmq.delay_ttl_ms`, default `60000`.

## Deferred Scope

The following items are accepted MVP deferrals and are not part of Task 013:

- retry strategy;
- partial failure protection;
- structured logging;
- monitoring;
- alerts.

These items belong to the next version.
