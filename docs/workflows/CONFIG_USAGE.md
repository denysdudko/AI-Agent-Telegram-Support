# Workflow Config Usage

## Purpose

This document explains how `TG Intake` and `TG Escalation` use the production configuration contract in the n8n workflow exports.

The configuration contract is defined in `docs/configuration/PRODUCTION_CONFIG.md`, with a JSON-only example at `config/telegram-support-agent.config.example.json`.

## Implemented Approach

Both workflow exports include a `Load Production Config` Code node. The node reads the n8n Variable `TG_SUPPORT_CONFIG_JSON`, parses it with `JSON.parse`, validates required fields and types, and exposes the parsed object as `config` for downstream nodes.

`TG_SUPPORT_CONFIG_JSON` is the single runtime configuration source of truth for both workflows. The workflow exports do not contain the runtime config values.

## Creating TG_SUPPORT_CONFIG_JSON in n8n

1. Open n8n.
2. Go to the Variables section.
3. Create a variable named `TG_SUPPORT_CONFIG_JSON`.
4. Copy the full JSON value from `config/telegram-support-agent.config.example.json`.
5. Adjust non-secret operational values for the target environment.
6. Save the variable.

Do not store secrets in `TG_SUPPORT_CONFIG_JSON`. API keys, bot tokens, database passwords, RabbitMQ URLs, and credential names must remain in n8n credentials or secret management.

Changing `TG_SUPPORT_CONFIG_JSON` updates the runtime configuration used by both `TG Intake` and `TG Escalation`. If a trigger-level setting such as the RabbitMQ trigger queue does not apply automatically after a variable change, reactivate the affected workflow so n8n recreates the trigger subscription.

Validation checks:

- missing `TG_SUPPORT_CONFIG_JSON` variable;
- invalid JSON;
- `environment.name` is a non-empty string;
- `telegram.expert_group.chat_id` is an integer;
- `telegram.parse_mode` is a non-empty string;
- RabbitMQ exchange, queue, and routing key values are non-empty strings;
- `rabbitmq.delay_ttl_ms` is a positive integer;
- `openai.default_model` is a non-empty string;
- `languages.supported` is a non-empty array;
- `languages.default_fallback` is included in `languages.supported`.

If validation fails, the Code node throws a clear `Production config validation failed` error and stops the workflow before downstream operational steps run.

`TG Escalation` has one trigger-specific detail: the RabbitMQ Trigger queue is evaluated before any Code node can run. Its queue field reads directly from `TG_SUPPORT_CONFIG_JSON` with `JSON.parse($vars.TG_SUPPORT_CONFIG_JSON).rabbitmq.queues.escalation`. The validation node then runs immediately after payload parsing and before runtime loading, AI analysis, or Telegram sends.

## Shared Rules

- Workflows must read one JSON configuration object from `TG_SUPPORT_CONFIG_JSON` for operational values.
- Secrets must stay in n8n credentials or environment-specific secret management.
- Config reads must not change the documented architecture: one orchestration layer, one retrieval layer, one answer layer, one `runtime_state`, and JSON schema everywhere.
- Config reads must not introduce Redis, a new service, or a second runtime store.
- Config validation happens in the `Load Production Config` Code node.
- Workflow behavior should remain equivalent to the current defaults unless an explicit future task changes behavior.

## TG Intake Usage

`TG Intake` reads `TG_SUPPORT_CONFIG_JSON` immediately after `Normalize Message` and before expert lookup, runtime lookup, runtime creation/update, and RabbitMQ publishing.

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

`TG Escalation` reads `TG_SUPPORT_CONFIG_JSON` after `Parse Payload` and before `Load Runtime State`, stale-event checks, AI analysis, message building, or Telegram sends.

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

- RabbitMQ trigger queue reads directly from `JSON.parse($vars.TG_SUPPORT_CONFIG_JSON).rabbitmq.queues.escalation`.
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

- The current implementation does not embed duplicated JSON config objects in workflow exports.
- `TG_SUPPORT_CONFIG_JSON` must follow the same shape as `config/telegram-support-agent.config.example.json`.
- Do not add new AI nodes.
- Do not change database schema.
- Do not change `runtime_state` shape unless a future explicit task requires it.
- Do not store secrets in the JSON config.

## Deferred Scope

The following items are accepted MVP deferrals and are not part of Task 012:

- retry strategy;
- partial failure protection;
- structured logging;
- monitoring;
- alerts.

These items belong to the next version.
