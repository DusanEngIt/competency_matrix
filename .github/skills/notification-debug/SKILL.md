---
name: notification-debug
description: "Debug stuck, missing, or undelivered notifications. Use when a user didn't receive an in-app alert after a manager edit, review reminders were not sent, email notifications bounced, or the notification badge count is wrong."
argument-hint: "Describe the issue: 'manager edit notification not received', 'review reminders not sent', or provide employee email"
---

# Notification Debug

Traces notifications end-to-end from trigger event → Celery task → DB → email delivery.

## Notification Types

| Trigger                         | Type           | Celery Task                                    |
| ------------------------------- | -------------- | ---------------------------------------------- |
| Manager edits subordinate skill | In-app + Email | `notifications.send_skill_change_notification` |
| Review cycle reminder (T-14)    | In-app + Email | `review_reminders.send_reminders`              |
| New hire welcome                | Email only     | `workday_sync.send_welcome_email`              |

## Procedure

### 1. Check Notifications Table

```sql
-- All notifications for a user
SELECT id, type, message, is_read, created_at
FROM notifications
WHERE employee_id = (SELECT id FROM employees WHERE email = '<user_email>')
ORDER BY created_at DESC
LIMIT 20;

-- Undelivered email notifications
SELECT id, type, notification_sent, created_at
FROM skill_audit_log
WHERE employee_id = (SELECT id FROM employees WHERE email = '<user_email>')
  AND notification_sent = FALSE
ORDER BY created_at DESC;
```

### 2. Check Celery Task Was Queued

```bash
docker compose logs celery --tail=200 | grep -E "notification|skill_change|reminder|ERROR"
```

If no log entry for the expected event → task was never queued. Check the backend route that should have called `.delay()`.

### 3. Check Email Delivery (SMTP)

```bash
# Verify SMTP env vars are set
docker compose exec backend env | grep SMTP

# Test SMTP connectivity
docker compose exec backend python -c "
import smtplib
s = smtplib.SMTP(host='<SMTP_HOST>', port=587)
s.starttls()
s.login('<user>', '<pass>')
print('SMTP OK')
"
```

### 4. Re-queue a Failed Notification

```bash
# For a specific audit log entry where notification_sent = FALSE
docker compose exec backend python -c "
from app.tasks.notifications import send_skill_change_notification
send_skill_change_notification.delay('<audit_log_id>')
"
```

### 5. Check In-App Badge Count

The unread count is cached in Redis:

```bash
docker compose exec redis redis-cli get "notifications:unread:<employee_id>"
# If stale or NULL, cache will rebuild on next GET /api/notifications
docker compose exec redis redis-cli del "notifications:unread:<employee_id>"
```

## Common Issues

| Symptom                   | Cause                                          | Fix                                            |
| ------------------------- | ---------------------------------------------- | ---------------------------------------------- |
| In-app alert not shown    | `notification_sent = FALSE` in audit_log       | Re-queue notification task                     |
| Email not received        | SMTP not configured (Section 17 open question) | Set SMTP env vars + restart backend            |
| Review reminders not sent | Celery beat not running                        | Start `celery beat` alongside worker           |
| Badge count wrong         | Stale Redis cache                              | `DEL notifications:unread:<employee_id>`       |
| Duplicate notifications   | Task retried and succeeded twice               | Add idempotency check in task before inserting |

## Output

Report:

1. Notification row found in DB (yes/no)
2. `skill_audit_log.notification_sent` status
3. Celery task evidence in logs
4. SMTP connectivity result
5. Root cause + fix
