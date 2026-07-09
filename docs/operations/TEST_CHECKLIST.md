# MVP Test Checklist

## Purpose

This checklist supports manual MVP testing for the Telegram Support Agent. It verifies the documented behavior of `TG Intake`, `TG Escalation`, Supabase/Postgres runtime state, RabbitMQ delay events, and Telegram notifications.

Do not mark these checks as automated. Execute them manually in a controlled Telegram test group with test users, test experts, test RabbitMQ queues, and test database records.

## Shared Verification Points

- [ ] Confirm expected status values are used only as documented: `waiting`, `answered_by_expert`, `escalated`, `ignored`.
- [ ] Confirm `runtime_state.messages[]` contains collected user messages.
- [ ] Confirm `runtime_state.analysis.language` is set after `TG Escalation`.
- [ ] Confirm `runtime_state.analysis.topic` is set after `TG Escalation`.
- [ ] Confirm `runtime_state.analysis.questions` is set after `TG Escalation`.
- [ ] Confirm `runtime_state.analysis.needs_escalation` is set to `true` or `false`.
- [ ] Confirm `runtime_state.escalation.sent_to_experts` is `true` only when escalation is required.
- [ ] Confirm `runtime_state.escalation.sent_to_user` is `true` only when escalation is required.
- [ ] Confirm `TG Intake` publishes a RabbitMQ delay event with `runtime_id` and `queue_version`.
- [ ] Confirm an old RabbitMQ event with an outdated `queue_version` is ignored.
- [ ] Confirm the latest RabbitMQ event with the current `queue_version` can proceed when status is still `waiting`.
- [ ] Confirm expert Telegram notification text uses Ukrainian.
- [ ] Confirm expert Telegram notification includes group, user, Telegram ID, topic, questions, link, and message count.
- [ ] Confirm user Telegram notification uses the detected language template.
- [ ] Confirm user Telegram notification replies to the latest user message.

## 1. Normal User Support Request

### Purpose

Verify the main happy path: a user support request waits for the delay, is analyzed, escalates to experts, notifies the user, and ends as `escalated`.

### Steps

- [ ] Send a clear support request from a non-expert user in the test Telegram group.
- [ ] Verify `TG Intake` creates a runtime record with status `waiting`.
- [ ] Verify `TG Intake` publishes a RabbitMQ delay event for the created runtime record.
- [ ] Do not send any expert reply during the 60-second wait.
- [ ] Allow the RabbitMQ delay event to trigger `TG Escalation`.
- [ ] Verify `TG Escalation` runs AI analysis and determines escalation is required.

### Expected Result

- [ ] Runtime status changes from `waiting` to `escalated`.
- [ ] `runtime_state.analysis.needs_escalation` is `true`.
- [ ] Expert notification is sent.
- [ ] User notification is sent.

### Data to Verify in Supabase/Postgres

- [ ] Runtime record exists for the user and chat.
- [ ] `status = escalated`.
- [ ] `queue_version` matches the processed RabbitMQ event.
- [ ] `runtime_state.messages[]` contains the user support request.
- [ ] `runtime_state.analysis.language` is populated.
- [ ] `runtime_state.analysis.topic` is populated.
- [ ] `runtime_state.analysis.questions` contains at least one relevant question or support item.
- [ ] `runtime_state.analysis.needs_escalation = true`.
- [ ] `runtime_state.escalation.sent_to_experts = true`.
- [ ] `runtime_state.escalation.sent_to_user = true`.

### Telegram Messages to Verify

- [ ] Expert message is in Ukrainian.
- [ ] Expert message includes group, user, Telegram ID, topic, questions, link, and message count.
- [ ] User message uses the detected language template.
- [ ] User message replies to the latest user message.

## 2. Multiple Messages from the Same User Within 60 Seconds

### Purpose

Verify message aggregation, `queue_version` incrementing, stale RabbitMQ event handling, and escalation based on the latest collected messages.

### Steps

- [ ] Send a support request from a non-expert user.
- [ ] Within 60 seconds, send one or more follow-up messages from the same user in the same chat.
- [ ] Verify each follow-up is appended to the existing runtime state instead of creating an unrelated active waiting record.
- [ ] Verify the first RabbitMQ delay event has an older `queue_version`.
- [ ] Allow the old delayed event to arrive.
- [ ] Allow the latest delayed event to arrive.

### Expected Result

- [ ] Old `queue_version` event is ignored.
- [ ] Latest `queue_version` event can proceed if status remains `waiting`.
- [ ] Runtime status eventually becomes `escalated` when analysis requires expert help.
- [ ] User notification replies to the latest user message, not the first message.

### Data to Verify in Supabase/Postgres

- [ ] One active runtime record tracks the collected messages for the user/chat.
- [ ] `runtime_state.messages[]` contains all user messages sent during the 60-second window.
- [ ] `queue_version` increases after the follow-up message.
- [ ] The stale event does not change status away from `waiting`.
- [ ] After the latest event proceeds, `runtime_state.analysis.language`, `analysis.topic`, `analysis.questions`, and `analysis.needs_escalation` are set.
- [ ] If escalated, `runtime_state.escalation.sent_to_experts = true` and `runtime_state.escalation.sent_to_user = true`.

### Telegram Messages to Verify

- [ ] No expert or user notification is sent by the old stale event.
- [ ] Expert message uses Ukrainian.
- [ ] Expert message shows the correct message count.
- [ ] User notification replies to the latest user message.

## 3. Expert Reply Before Timer Expires

### Purpose

Verify an expert reply closes waiting runtime records before escalation and prevents delayed escalation from proceeding.

### Steps

- [ ] Send a support request from a non-expert user.
- [ ] Confirm runtime status is `waiting`.
- [ ] Before 60 seconds expires, send a reply from a configured expert in the same Telegram chat.
- [ ] Allow the original RabbitMQ delay event to arrive after the waiting period.

### Expected Result

- [ ] Runtime status changes to `answered_by_expert`.
- [ ] Delayed `TG Escalation` does not notify experts or the user.
- [ ] The delayed event is ignored because status is no longer `waiting`.

### Data to Verify in Supabase/Postgres

- [ ] `status = answered_by_expert`.
- [ ] `runtime_state.messages[]` still contains the collected user message or messages.
- [ ] `runtime_state.expert_reply.detected` reflects that an expert reply was detected, if available in the current runtime state.
- [ ] `runtime_state.analysis.needs_escalation` is absent or not changed by the stale delayed event.
- [ ] `runtime_state.escalation.sent_to_experts` is not `true`.
- [ ] `runtime_state.escalation.sent_to_user` is not `true`.

### Telegram Messages to Verify

- [ ] No expert escalation notification is sent.
- [ ] No user escalation notification is sent.
- [ ] Expert reply remains visible in the original Telegram group thread.

## 4. Stale RabbitMQ Event Ignored

### Purpose

Verify RabbitMQ events are treated as triggers only, and stale events with old `queue_version` values cannot mutate current runtime state.

### Steps

- [ ] Send message A from a non-expert user.
- [ ] Record the runtime id and initial `queue_version`.
- [ ] Send message B from the same user within 60 seconds.
- [ ] Record the updated `queue_version`.
- [ ] Observe or replay the old RabbitMQ event with the initial `queue_version`.
- [ ] Observe the latest RabbitMQ event with the current `queue_version`.

### Expected Result

- [ ] Old `queue_version` event is ignored.
- [ ] Latest `queue_version` event can proceed if `status = waiting`.
- [ ] Runtime state is loaded from Supabase/Postgres before proceeding.

### Data to Verify in Supabase/Postgres

- [ ] Runtime record keeps the latest `queue_version`.
- [ ] `runtime_state.messages[]` includes message A and message B.
- [ ] Status remains `waiting` after the old event is ignored.
- [ ] After the latest event, status becomes `escalated` or `ignored` depending on AI analysis.
- [ ] `runtime_state.analysis.needs_escalation` is set to `true` or `false` only after the latest event proceeds.

### Telegram Messages to Verify

- [ ] Old event does not send expert or user notifications.
- [ ] If latest event escalates, expert message uses Ukrainian and includes group, user, Telegram ID, topic, questions, link, and message count.
- [ ] If latest event escalates, user message uses the detected language template and replies to the latest user message.

## 5. Greeting / Thanks / Small Talk Ignored

### Purpose

Verify non-support messages are analyzed and marked `ignored` without notifying experts or users.

### Steps

- [ ] Send a greeting, thanks, acknowledgement, or small-talk-only message from a non-expert user.
- [ ] Confirm runtime status starts as `waiting`.
- [ ] Let the 60-second RabbitMQ delay event trigger `TG Escalation`.
- [ ] Verify AI analysis identifies that escalation is not needed.

### Expected Result

- [ ] Runtime status changes from `waiting` to `ignored`.
- [ ] `runtime_state.analysis.needs_escalation` is `false`.
- [ ] No expert notification is sent.
- [ ] No user notification is sent.

### Data to Verify in Supabase/Postgres

- [ ] `status = ignored`.
- [ ] `runtime_state.messages[]` contains the small-talk message.
- [ ] `runtime_state.analysis.language` is set.
- [ ] `runtime_state.analysis.topic` is set to a non-support or small-talk topic.
- [ ] `runtime_state.analysis.questions` is empty or contains no actionable support question.
- [ ] `runtime_state.analysis.needs_escalation = false`.
- [ ] `runtime_state.escalation.sent_to_experts` is not `true`.
- [ ] `runtime_state.escalation.sent_to_user` is not `true`.

### Telegram Messages to Verify

- [ ] No expert notification appears.
- [ ] No user escalation confirmation appears.

## 6. User Notification Language Selection

### Purpose

Verify user notification language follows detected language and unsupported language falls back to the `other`/`en` template.

### Steps

- [ ] Send a support request in a supported language, such as Ukrainian, Polish, English, or Russian.
- [ ] Let the request escalate.
- [ ] Record the detected `runtime_state.analysis.language`.
- [ ] Repeat with an unsupported language or mixed-language message.
- [ ] Let the unsupported-language request escalate.

### Expected Result

- [ ] Supported-language notification uses the matching detected language template.
- [ ] Unsupported-language notification falls back to the `other`/`en` template.
- [ ] User notification replies to the latest user message.

### Data to Verify in Supabase/Postgres

- [ ] `status = escalated` for each escalated request.
- [ ] `runtime_state.analysis.language` matches a supported value or `other`.
- [ ] `runtime_state.analysis.topic` is set.
- [ ] `runtime_state.analysis.questions` is set.
- [ ] `runtime_state.analysis.needs_escalation = true`.
- [ ] `runtime_state.escalation.sent_to_user = true`.
- [ ] `runtime_state.escalation.sent_to_experts = true`.

### Telegram Messages to Verify

- [ ] User message uses the detected language template for supported languages.
- [ ] User message uses the `other`/`en` fallback template for unsupported language.
- [ ] User message replies to the latest user message.
- [ ] Expert message still uses Ukrainian regardless of user language.

## 7. Expert Notification Format

### Purpose

Verify the expert-facing notification has the required Ukrainian format and includes all required support context.

### Steps

- [ ] Send a support request that clearly requires escalation.
- [ ] Allow `TG Escalation` to notify experts.
- [ ] Inspect the expert group notification.

### Expected Result

- [ ] Expert notification is sent only when `runtime_state.analysis.needs_escalation = true`.
- [ ] Expert notification uses Ukrainian.
- [ ] Expert notification includes all required context for a human expert.

### Data to Verify in Supabase/Postgres

- [ ] `status = escalated`.
- [ ] `runtime_state.analysis.language` is set.
- [ ] `runtime_state.analysis.topic` is set and matches the notification topic.
- [ ] `runtime_state.analysis.questions` is set and appears in the notification.
- [ ] `runtime_state.analysis.needs_escalation = true`.
- [ ] `runtime_state.escalation.sent_to_experts = true`.

### Telegram Messages to Verify

- [ ] Expert message includes group name.
- [ ] Expert message includes user name.
- [ ] Expert message includes Telegram ID.
- [ ] Expert message includes topic.
- [ ] Expert message includes questions.
- [ ] Expert message includes a link to the Telegram message.
- [ ] Expert message includes message count.
- [ ] Expert message is in Ukrainian.

## 8. Runtime Status Transitions

### Purpose

Verify each documented runtime status transition can be observed manually and no undocumented status is introduced.

### Steps

- [ ] Trigger a normal support request and verify initial status `waiting`.
- [ ] Let one request escalate and verify final status `escalated`.
- [ ] Reply as an expert before the timer expires and verify final status `answered_by_expert`.
- [ ] Send small talk and verify final status `ignored`.
- [ ] Review runtime records created during the test run.

### Expected Result

- [ ] New user support runtime starts as `waiting`.
- [ ] Escalated support request ends as `escalated`.
- [ ] Expert-handled request ends as `answered_by_expert`.
- [ ] Non-support message ends as `ignored`.
- [ ] No other status value is used.

### Data to Verify in Supabase/Postgres

- [ ] `waiting` appears only while the request is inside the active delay window.
- [ ] `answered_by_expert` records do not send escalation notifications.
- [ ] `escalated` records have `runtime_state.escalation.sent_to_experts = true` and `runtime_state.escalation.sent_to_user = true`.
- [ ] `ignored` records have `runtime_state.analysis.needs_escalation = false`.
- [ ] `runtime_state.messages[]` is present for every tested record.

### Telegram Messages to Verify

- [ ] `waiting` alone does not send a user or expert notification.
- [ ] `answered_by_expert` does not send escalation notifications.
- [ ] `escalated` sends expert and user notifications.
- [ ] `ignored` sends no expert or user notification.

## 9. Failure / Edge Cases

### Purpose

Verify known MVP edge cases are understood and manually checked without introducing workflow, database, runtime, service, or implementation changes.

### Steps

- [ ] Send an empty text message or message with only whitespace, if Telegram and the workflow trigger allow it.
- [ ] Send a message without a question mark but with a clear support request, such as "My payment failed and I need help".
- [ ] Send an unsupported-language support request.
- [ ] Simulate or observe a Telegram send failure in a controlled test environment, if practical.
- [ ] Record the behavior and compare it to known MVP limitations.

### Expected Result

- [ ] Empty text message does not create a broken escalation path.
- [ ] Message without a question mark but with a support request can still be escalated when AI analysis determines help is needed.
- [ ] Unsupported language falls back to the `other`/`en` user notification template when escalation is required.
- [ ] Telegram send failure is recorded as a known MVP risk, not fixed in this task.

### Data to Verify in Supabase/Postgres

- [ ] Empty text behavior is recorded with the observed status: `ignored`, `waiting`, or another documented outcome if already implemented.
- [ ] Support request without a question has `runtime_state.analysis.needs_escalation = true` when escalation is required.
- [ ] Unsupported language has `runtime_state.analysis.language = other` or another documented fallback value.
- [ ] For escalated edge cases, `runtime_state.analysis.topic` and `runtime_state.analysis.questions` are set after `TG Escalation`.
- [ ] For Telegram send failure, record whether database state shows attempted escalation and note that send failures are not transactionally tied to runtime status in the MVP.

### Telegram Messages to Verify

- [ ] Empty text message does not produce malformed expert or user Telegram output.
- [ ] Support request without a question mark produces normal expert and user notifications when escalated.
- [ ] Unsupported-language request uses the fallback user template when escalated.
- [ ] Telegram send failure is documented in test notes as a known MVP risk.

