# Production Configuration Contract

## Purpose

This document defines the single production configuration contract for the Telegram Support Agent MVP. The contract centralizes operational values that are currently spread across n8n workflow JSON exports and infrastructure setup.

The contract centralizes configuration without changing database schema, runtime state shape, AI node count, services, or infrastructure.

## Contract Rules

- The runtime configuration file must be JSON.
- The example config must not contain secrets, tokens, passwords, connection URLs, or credential names.
- Workflows must continue to use one orchestration layer, one retrieval layer, one answer layer, a single `runtime_state`, and JSON schema everywhere.
- The config must not introduce Redis, a new service, or a second runtime state store.
- n8n workflow exports read operational values from this contract instead of relying on scattered hardcoded workflow literals.
- Secrets remain in n8n credentials or environment-specific secret management, not in this config.

## Config File

Example path:

```text
config/telegram-support-agent.config.example.json
```

Production deployments should provide an environment-specific JSON file with the same shape. The repository example documents defaults and non-secret operational values only.

## Runtime Configuration Shape

The top-level object contains:

- `environment`: environment name and fallback behavior.
- `telegram`: source group rules, expert group target, and Telegram formatting.
- `rabbitmq`: exchange, queue, routing key, and delay TTL settings.
- `openai`: default model used by the escalation analysis path.
- `languages`: supported language keys and fallback language.

## Required Fields

### environment

- `name`: environment label, such as `development`, `staging`, or `production`.

### telegram.source_groups

- `enabled`: whether source group filtering should be enforced by workflow logic.
- `groups[]`: list of allowed Telegram source groups.
- `groups[].chat_id`: source Telegram group chat id.
- `groups[].title`: human-readable group name for operators.

The example keeps `groups[]` empty because no source group id is currently documented as a repository default. Production config should fill this list before source group filtering is enabled.

### telegram.expert_group

- `chat_id`: expert Telegram group chat id.
- `title`: human-readable expert group name.

Current default expert chat id:

```text
-1003811712654
```

### telegram.parse_mode

- `parse_mode`: Telegram message parse mode used by expert and user notifications.

Current default:

```text
HTML
```

### rabbitmq.exchanges

- `support`: primary exchange used by `TG Intake`.
- `dead_letter`: dead-letter exchange used after delay expiry.

Current defaults:

```text
tg.support
tg.dlx
```

### rabbitmq.queues

- `delay`: delay queue.
- `escalation`: queue consumed by `TG Escalation`.

Current defaults:

```text
tg.delay
tg.escalation
```

### rabbitmq.routing_keys

- `delay`: routing key used when publishing delayed support events.
- `escalation`: routing key used when dead-lettered delayed events are routed to escalation.

Current default routing key:

```text
delay
```

### rabbitmq.delay_ttl_ms

- Delay window in milliseconds before an unanswered support request can be escalated.

Current default:

```text
60000
```

### openai.default_model

- Default model used by the existing escalation analysis path.

Current default:

```text
gpt-5-mini
```

### languages.supported

- Supported language keys used by AI structured output and user notification templates.

Current defaults:

```text
ru, uk, pl, en, other
```

### languages.default_fallback

- Fallback user notification language when the detected language is unsupported.

Current default:

```text
other
```

## Example JSON Contract

```json
{
  "environment": {
    "name": "production"
  },
  "telegram": {
    "source_groups": {
      "enabled": false,
      "groups": []
    },
    "expert_group": {
      "chat_id": -1003811712654,
      "title": "Telegram Support Experts"
    },
    "parse_mode": "HTML"
  },
  "rabbitmq": {
    "exchanges": {
      "support": "tg.support",
      "dead_letter": "tg.dlx"
    },
    "queues": {
      "delay": "tg.delay",
      "escalation": "tg.escalation"
    },
    "routing_keys": {
      "delay": "delay",
      "escalation": "tg.escalation"
    },
    "delay_ttl_ms": 60000
  },
  "openai": {
    "default_model": "gpt-5-mini"
  },
  "languages": {
    "supported": ["ru", "uk", "pl", "en", "other"],
    "default_fallback": "other"
  }
}
```

## JSON Schema

The configuration should be validated with this schema before a workflow uses it:

```json
{
  "type": "object",
  "required": ["environment", "telegram", "rabbitmq", "openai", "languages"],
  "additionalProperties": false,
  "properties": {
    "environment": {
      "type": "object",
      "required": ["name"],
      "additionalProperties": false,
      "properties": {
        "name": {
          "type": "string",
          "minLength": 1
        }
      }
    },
    "telegram": {
      "type": "object",
      "required": ["source_groups", "expert_group", "parse_mode"],
      "additionalProperties": false,
      "properties": {
        "source_groups": {
          "type": "object",
          "required": ["enabled", "groups"],
          "additionalProperties": false,
          "properties": {
            "enabled": {
              "type": "boolean"
            },
            "groups": {
              "type": "array",
              "items": {
                "type": "object",
                "required": ["chat_id", "title"],
                "additionalProperties": false,
                "properties": {
                  "chat_id": {
                    "type": "integer"
                  },
                  "title": {
                    "type": "string"
                  }
                }
              }
            }
          }
        },
        "expert_group": {
          "type": "object",
          "required": ["chat_id", "title"],
          "additionalProperties": false,
          "properties": {
            "chat_id": {
              "type": "integer"
            },
            "title": {
              "type": "string"
            }
          }
        },
        "parse_mode": {
          "type": "string",
          "enum": ["HTML"]
        }
      }
    },
    "rabbitmq": {
      "type": "object",
      "required": ["exchanges", "queues", "routing_keys", "delay_ttl_ms"],
      "additionalProperties": false,
      "properties": {
        "exchanges": {
          "type": "object",
          "required": ["support", "dead_letter"],
          "additionalProperties": false,
          "properties": {
            "support": {
              "type": "string"
            },
            "dead_letter": {
              "type": "string"
            }
          }
        },
        "queues": {
          "type": "object",
          "required": ["delay", "escalation"],
          "additionalProperties": false,
          "properties": {
            "delay": {
              "type": "string"
            },
            "escalation": {
              "type": "string"
            }
          }
        },
        "routing_keys": {
          "type": "object",
          "required": ["delay", "escalation"],
          "additionalProperties": false,
          "properties": {
            "delay": {
              "type": "string"
            },
            "escalation": {
              "type": "string"
            }
          }
        },
        "delay_ttl_ms": {
          "type": "integer",
          "minimum": 1
        }
      }
    },
    "openai": {
      "type": "object",
      "required": ["default_model"],
      "additionalProperties": false,
      "properties": {
        "default_model": {
          "type": "string",
          "minLength": 1
        }
      }
    },
    "languages": {
      "type": "object",
      "required": ["supported", "default_fallback"],
      "additionalProperties": false,
      "properties": {
        "supported": {
          "type": "array",
          "minItems": 1,
          "items": {
            "type": "string"
          }
        },
        "default_fallback": {
          "type": "string"
        }
      }
    }
  }
}
```

## Config-Driven Workflow Values

The n8n workflow exports should read these operational values from the production config contract:

- `TG Intake` RabbitMQ publish exchange: `tg.support`.
- `TG Intake` RabbitMQ publish routing key: `delay`.
- `TG Escalation` RabbitMQ trigger queue: `tg.escalation`.
- `TG Escalation` OpenAI model: `gpt-5-mini`.
- `TG Escalation` expert Telegram chat id: `-1003811712654`.
- `TG Escalation` Telegram parse mode: `HTML`.
- RabbitMQ topology defaults: `tg.support`, `tg.dlx`, `tg.delay`, `tg.escalation`, `delay`, and `60000`.

Task 011 implemented config loading in the workflow exports while keeping the same defaults and business behavior.

## Non-Goals

- Workflow exports may read from this contract, but the contract must not change business behavior by itself.
- No database schema changes are introduced.
- No implementation code is added.
- No new AI nodes are added.
- No Redis or new service is introduced.
- No secrets are stored in the example config.
