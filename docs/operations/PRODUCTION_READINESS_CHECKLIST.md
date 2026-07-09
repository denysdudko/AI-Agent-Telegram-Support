# Production Readiness Checklist

## Purpose

This checklist tracks MVP production readiness for the Telegram Support Agent. Checked items are already documented or implemented in the current MVP. Open items are still missing, risky, or need explicit follow-up before production use.

## 1. Architecture

- [x] One orchestration path is documented.
- [x] Supabase/Postgres as the source of truth is documented.
- [x] RabbitMQ 60-second delay is documented.
- [x] `queue_version` stale timer protection is documented.
- [x] End-to-end message lifecycle is documented.
- [ ] Architecture decision records are not yet expanded beyond MVP documentation.

## 2. Infrastructure

- [x] Required MVP components are documented: n8n, Supabase/Postgres, CloudAMQP/RabbitMQ, Telegram bots, and OpenAI API.
- [x] Environment/configuration variables are documented without secrets.
- [ ] No deployment/runbook is documented yet.
- [ ] Expert group chat ID should be configurable.
- [ ] No new infrastructure should be added without an explicit task.

## 3. Database

- [x] `telegram_runtime_state` is documented.
- [x] `telegram_experts` is documented.
- [x] Status values are documented.
- [x] Data rules for active waiting queue lookup are documented.
- [ ] No SQL migration plan is implemented yet.
- [ ] No backup/restore procedure is documented yet.

## 4. RabbitMQ

- [x] RabbitMQ exchange and queue topology is documented.
- [x] TTL of `60000` ms is documented.
- [x] RabbitMQ payload contract is documented.
- [x] RabbitMQ events are documented as triggers, not the source of truth.
- [ ] No retry/error handling strategy is documented yet.
- [ ] No operational queue monitoring checklist is documented yet.

## 5. Telegram

- [x] Listener bot role is documented.
- [x] Broadcast/support bot role is documented.
- [x] TG Intake is documented.
- [x] TG Escalation is documented.
- [x] Ukrainian expert notification is documented.
- [ ] Telegram send failures are not transactionally tied to `Mark Escalated`.
- [ ] Expert reply currently closes all waiting queues in the chat.
- [ ] Telegram chat IDs should be configurable long-term.

## 6. OpenAI / AI Analysis

- [x] OpenAI is used only in `TG Escalation`.
- [x] AI structured output is documented.
- [x] Data contracts and JSON schema examples are documented.
- [x] User language detection is documented.
- [x] `needs_escalation` decision rules are documented.
- [ ] No prompt regression test checklist is documented yet.
- [ ] No fallback behavior for AI/API failure is documented yet.

## 7. Runtime State

- [x] Single `runtime_state` principle is documented.
- [x] `runtime_state` JSON structure is documented.
- [x] Runtime status transitions are documented.
- [x] `queue_version` protects against stale RabbitMQ events.
- [ ] Runtime cleanup/retention policy is not documented yet.
- [ ] Concurrent multi-user waiting behavior needs refinement after MVP.

## 8. Workflow Reliability

- [x] Main happy path is documented.
- [x] Expert reply path is documented.
- [x] Stale timer path is documented.
- [x] Ignored path is documented.
- [ ] No formal retry/error handling strategy is documented yet.
- [ ] No test checklist is documented yet.
- [ ] Telegram send success and database status updates are not transactionally coupled.

## 9. Security

- [x] Credentials must not be committed.
- [x] Workflow exports may contain credential references and must be reviewed before sharing.
- [x] Environment variables are documented without secrets.
- [ ] Workflow export review process is not formalized.
- [ ] Access policy for Telegram bots, database credentials, and RabbitMQ credentials is not documented yet.

## 10. Observability

- [x] Runtime state provides a durable source for debugging workflow decisions.
- [ ] No formal monitoring dashboard is documented yet.
- [ ] No alerting rules are documented yet.
- [ ] No operational log review procedure is documented yet.
- [ ] No RabbitMQ queue depth monitoring process is documented yet.

## 11. MVP Limitations

- [x] Known MVP workflow limitations are documented.
- [ ] Expert replies currently close all waiting queues in the chat.
- [ ] Telegram send failures are not transactionally tied to `Mark Escalated`.
- [ ] Expert chat ID should become configurable.
- [ ] Workflow exports may contain credential references.
- [ ] No formal monitoring dashboard is documented yet.
- [ ] No retry/error handling strategy is documented yet.
- [ ] No test checklist is documented yet.
- [ ] No deployment/runbook is documented yet.

## 12. Go-Live Decision

### Ready for MVP test

- [x] Architecture and main orchestration path are documented.
- [x] TG Intake and TG Escalation are documented.
- [x] Data contracts and status values are documented.
- [x] RabbitMQ delay and stale timer protection are documented.

### Required before production

- [ ] Make expert group chat ID configurable.
- [ ] Document deployment/runbook.
- [ ] Document retry/error handling strategy.
- [ ] Document monitoring and alerting.
- [ ] Document test checklist.
- [ ] Review workflow exports for credential references before sharing or deploying.

### Deferred after MVP

- [ ] Refine expert reply handling so it does not close unrelated waiting queues in the same chat.
- [ ] Transactionally align Telegram send results with `Mark Escalated`.
- [ ] Define runtime retention and cleanup policy.
- [ ] Expand architecture decision records.
