---
name: workera-pull-audit-events-to-siem
description: >-
  Poll Workera's enterprise audit log into a SIEM, with correct cursor handling, scope diagnosis and
  correlation keys.
api: Workera API
generated: '2026-09-04'
method: generated
source: openapi/workera-api-openapi.json
operations:
  - WorkeraWebappsWeb.Rest.Controllers.AuditEventsController.index
  - WorkeraWebappsWeb.Rest.Controllers.PingController.ping
---

# Pull the Workera audit log into a SIEM

## Before you start

`GET /api/v1/audit_events` (`WorkeraWebappsWeb.Rest.Controllers.AuditEventsController.index`) requires
the `audit_events` scope on your API key. A key without it authenticates fine and then returns **403**
— not 401. Ask your Workera CSM to grant the scope; you cannot grant it yourself.

## Steps

1. **Confirm the key works at all.** `GET /api/v1/ping`. If that is 200 and `/api/v1/audit_events` is
   403, the problem is the scope, not the key.

2. **Poll on an interval.** Workera's own guidance is roughly every minute.

3. **Page correctly.** `limit` defaults to 50 and maxes at 100 on this endpoint (the rest of the API
   defaults to 10). Sort with `order` (`asc` or `desc`, default `desc`). Continue while `has_more` is
   true.

4. **Track the cursor as a timestamp.** `next_page_after` on this endpoint is an ISO 8601 datetime,
   because that is the field the sort runs on — the general rule is that the cursor must be in the
   same format as the sorted field. Persist the `created_at` of the last processed event as your
   cursor. For a catch-up window, bound the pull with `from` (inclusive) and `to` (exclusive) instead.

5. **Filter server-side when you can.** `action`, `actor_id`, `target_type` and `target_id` are all
   query parameters. Pulling everything and filtering locally wastes your rate-limit budget.

6. **Map the correlation keys.**
   - `actor_enterprise_id` is the customer's HRIS employee ID — join on this to your identity data.
   - `request_id` correlates an event with Workera application logs when you open a support case.
   - `session_id` groups a user's activity.
   - `id` is a UUIDv7, so it sorts roughly by time and is safe as a deduplication key.
   - `targets[]` is polymorphic: each entry carries its own `{type, id}` (`user` and `program` are the
     documented types).

7. **Know the action vocabulary.** Actions are `{domain}.{action}`. Workera documents:
   `auth.login`, `auth.login_failed`, `auth.logout`, `auth.password_reset_requested`,
   `auth.password_reset_completed`, `auth.mfa_setup`, `auth.mfa_verified`, `auth.account_locked`,
   `user.created`, `user.deactivated`, `user.role_changed`, `admin.impersonation_started`,
   `admin.impersonation_ended`, `data.exported`, `data.sensitive_read`, `data.bulk_updated`,
   `api.auth_failed`, `api.rate_limited`, `api.key_created`, `api.key_revoked`, `program.created`,
   `program.launched`, `program.updated`, `program.deleted`, `program.learner_added`,
   `program.learner_removed`, `config.updated`. The list is documented, not declared as an enum in the
   schema — treat unknown actions as valid and alert rather than drop.

8. **Watch the security-relevant ones.** `admin.impersonation_started` / `..._ended`,
   `data.exported`, `data.sensitive_read`, `api.key_created` and `api.key_revoked` are the events a
   detection rule usually wants first.

## Rules

- **Rate limits.** Polling every minute against an unpublished limit means you must read
  `x-ratelimit-remaining` and back off on `x-ratelimit-reset`. 429 on exhaustion.
- **400 is real here.** This is the only operation in the API that declares a 400 — a malformed `from`,
  `to` or `limit` will get one. Validate before you send.
- `location` is an IP address and `metadata` is documented as PII-safe, but `actor_enterprise_id` and
  `user_agent` are still employee-identifying. Apply your normal handling.
