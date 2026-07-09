# Infrastructure

## Purpose

This document describes the infrastructure required for the Telegram Support Agent MVP. It documents the current project direction only and does not introduce new services, runtime behavior, workflow changes, or database changes.

## Required Components

- `n8n`: runs the `TG Intake` and `TG Escalation` workflows and coordinates Telegram, RabbitMQ, Supabase/Postgres, and OpenAI calls.
- `Supabase/Postgres`: stores durable application data and acts as the source of truth for support interactions.
- `CloudAMQP/RabbitMQ`: provides the delayed escalation timer between intake and escalation.
- `Telegram bots`: receive group messages and send support notifications.
- `OpenAI API`: provides AI capabilities for future prompt-based processing and answer generation.

## RabbitMQ Configuration

RabbitMQ is used for the 60-second escalation delay. The MVP expects the following topology:

- Exchange `tg.support`: primary exchange used by workflows to publish support events.
- Exchange `tg.dlx`: dead-letter exchange used after the delay expires.
- Queue `tg.delay`: delay queue with a TTL of `60000` ms.
- Queue `tg.escalation`: queue consumed by the escalation workflow.
- TTL: messages in `tg.delay` expire after `60000` ms.
- DLX routing: expired messages from `tg.delay` are routed through `tg.dlx` to `tg.escalation`.

This keeps the timer outside long-running n8n executions and lets `TG Escalation` process the message after the waiting window.

## Supabase/Postgres Role

Supabase/Postgres stores runtime state, experts, and any durable records needed by the workflows. It is the source of truth for deciding whether a message is still waiting, already answered, escalated, or ignored.

Workflows should read the latest state from Supabase/Postgres before making escalation or notification decisions.

## Telegram Bot Roles

- Listener bot: receives Telegram group messages and triggers the intake path.
- Broadcast/support bot: sends notifications to experts and users.

Telegram chat IDs must be configurable so workflows can be moved between development, test, and production groups without editing workflow logic.

## Environment and Configuration Variables

The following configuration values must be documented and supplied through environment or credential management. Do not commit secrets.

- `OPENAI_API_KEY`
- `POSTGRES_HOST`
- `POSTGRES_PORT`
- `POSTGRES_DATABASE`
- `POSTGRES_USER`
- `POSTGRES_PASSWORD`
- `RABBITMQ_URL`
- `TELEGRAM_LISTENER_TOKEN`
- `TELEGRAM_BROADCAST_TOKEN`
- Telegram support group chat ID
- Telegram expert group chat ID
- RabbitMQ exchange, queue, routing key, and TTL settings

## Production Notes

- Credentials must not be committed to the repository.
- n8n workflow exports may contain credential references; review exports before committing.
- Telegram chat IDs must be configurable.
- The expert group chat ID must not be hardcoded long-term.
