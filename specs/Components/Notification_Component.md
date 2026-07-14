# Notification Component

**Version:** 1.0
**Last Modified:** 2026-07-13
**Depends On:** Config_Framework.md (v1.0), Error_Handling_Framework.md (v1.0), Audit_Framework.md (v1.0), Logging_Framework.md (v1.0)
**Category:** Components

## Purpose
Implements the notification/alerting delivery referenced throughout the framework — whenever `Error_Handling_Framework.md` says "alert," `Audit_Framework.md` flags a `Mismatch`, or `Schema_Management_Framework.md` detects a breaking change, this is the component that actually sends the notification.

## Scope
Covers notification delivery only. Does NOT decide *when* to notify (that's determined by the calling Framework/Component, which passes a trigger event) and does NOT define notification *content templates* (those are embedded in the calling context, not centrally managed here — this component just delivers whatever message it's given).

## Inputs
| Input | Type | Description |
|---|---|---|
| `notification_type` | enum(Error, Warning, Info) | Urgency classification |
| `title` | string | Short summary (e.g., "Pipeline Halt: Orders") |
| `message` | string | Detail body — what happened, which table, what action is expected |
| `pipeline_id` | string | For traceability back to logs/audit |
| `table_name` | string, optional | If notification is table-specific |
| `blocked_dependents` | array[string], optional | If a failure cascaded, lists the downstream tables affected (per `Error_Handling_Framework.md`'s requirement) |

## Outputs
Delivery confirmation: `{delivered: boolean, channel: string, timestamp: timestamp}`. Does not return content — fire-and-forget from the caller's perspective.

## Configuration
Notification delivery targets are configured at the framework level (not per-table), since alert routing is an operational concern, not a data-pipeline concern:

| Setting | Type | Description |
|---|---|---|
| `notification_channel` | enum(email, webhook, databricks_alert) | Where notifications go |
| `notification_target` | string | Email address, webhook URL, or Databricks alert name |
| `enabled` | boolean | Global kill-switch for notifications (e.g., disable during development/testing) |

These settings should be stored as Databricks widgets, workspace-level configs, or a simple `notification_config` table — NOT in `Pipeline_Config`, since they're operational, not per-table.

## Implementation Rules
- Notification delivery must never block the pipeline — if the notification channel is unreachable, log the delivery failure and continue. A pipeline that completed its data work but failed to send an email is far better than a pipeline halted because an email server was down.
- Notifications must never contain sensitive data values (actual row content, PII) — only table names, counts, rule names, and pipeline identifiers. Consistent with the sensitivity principle in `Logging_Framework.md`.
- `blocked_dependents` should be included in the notification body whenever present — this saves the operator from having to manually look up what else was affected by a failure.

## Error Handling
| Scenario | Behavior |
|---|---|
| Notification channel unreachable | Retry once, then log "notification delivery failed" to the standard log table and return `{delivered: false}` — never halt |
| `notification_channel` is misconfigured (e.g., invalid webhook URL) | Log config error, return `{delivered: false}` — this is a setup issue, not a pipeline issue |
| `enabled = false` | Return immediately with `{delivered: false}`, no retry, no error — explicitly intended for dev/test suppression |

## Logging
Logs every notification attempt: `notification_type`, `title`, `delivered` (true/false), `channel`, and (on failure) the reason delivery failed. Per `Logging_Framework.md`'s Notebook Log type.

## Return Values
Returns `{delivered: boolean, channel: string, timestamp: timestamp}`. Never raises exceptions.

## Pseudo Logic
```
FUNCTION send_notification(notification_type, title, message, pipeline_id, table_name=None, blocked_dependents=None):
    config = READ_NOTIFICATION_CONFIG()
    IF NOT config.enabled:
        RETURN {delivered: false, channel: "disabled", timestamp: NOW()}

    body = FORMAT_BODY(title, message, pipeline_id, table_name, blocked_dependents)

    TRY:
        DELIVER(config.notification_channel, config.notification_target, body)
        LOG("notification delivered", notification_type, title)
        RETURN {delivered: true, channel: config.notification_channel, timestamp: NOW()}
    CATCH:
        RETRY_ONCE()
        IF still_failed:
            LOG_WARNING("notification delivery failed", notification_type, title, reason)
            RETURN {delivered: false, channel: config.notification_channel, timestamp: NOW()}
```

## Acceptance Criteria
- [ ] Notification delivery never halts the pipeline, regardless of channel availability.
- [ ] No sensitive data values appear in notification content — only identifiers, counts, and rule names.
- [ ] `blocked_dependents` are included when present, giving operators full cascading-failure visibility.
- [ ] `enabled = false` cleanly suppresses all notifications without error.

## Example (Illustrative Only)
```
notification_type: Error
title: "Pipeline Halt: Orders (Silver)"
message: "DQ blocking rule 'positive_amount_check' failed on 12 rows. Table processing halted."
pipeline_id: pipe_20260713_001
blocked_dependents: [fact_sales]
channel: webhook
delivered: true
```

## Dependencies
- `Error_Handling_Framework.md` (v1.0) — primary trigger source for Error/Warning notifications.
- `Audit_Framework.md` (v1.0) — triggers Info/Warning notifications on reconciliation `Mismatch`.
- `Logging_Framework.md` (v1.0) — logs delivery attempts/failures.
- `Config_Framework.md` (v1.0) — `pipeline_id`, `table_name` trace to standard config.

## Future Extension Points
- Could add per-severity channel routing (e.g., `Error` → PagerDuty, `Warning` → Slack, `Info` → email) if operational needs grow beyond a single channel.
- Could add notification batching/digest mode (collect all warnings from a workflow run and send one summary notification at the end, rather than one per event) to reduce alert fatigue.

## AI Generation Notes
Any agent generating error-handling or audit-checking code that references "alert" must call this component — never inline ad hoc notification logic (e.g., a raw HTTP call to a webhook) per notebook, since that bypasses the global enable/disable switch and sensitivity safeguards.