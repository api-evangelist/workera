---
name: workera-stream-assessment-events
description: >-
  Receive and verify Workera assessment, program and appeal webhooks, deduplicate them, and backfill
  anything the retry ladder gave up on.
api: Workera API
generated: '2026-09-04'
method: generated
source:
  - asyncapi/workera-events-asyncapi.yml
  - openapi/workera-api-openapi.json
operations:
  - WorkeraWebappsWeb.Rest.Controllers.V2.ScoresController.show
---

# Consume Workera webhooks

## Before you start

Webhook endpoints, the event types you receive, and the shared signing secret are all configured per
company by your Workera CSM. There is no self-serve webhook management API, so you cannot register,
list, rotate or replay a webhook programmatically.

## Steps

1. **Verify the signature before you parse the body.** Every request carries
   `X-Workera-Signature: sha256=<hex>`, an HMAC-SHA256 digest of the RAW request body keyed with your
   shared secret. Compute over the raw bytes — not a re-serialised object — and compare in constant
   time. Reject anything that does not match, and return a non-2xx only for genuine failures.

2. **Acknowledge fast, process later.** Return 2xx as soon as the signature verifies and the payload
   is durably queued. Workera's retry ladder is immediate, then 1 minute, 5 minutes, 15 minutes and
   1 hour; after 5 failures delivery stops permanently and an email goes to your configured contact.
   A slow handler burns retries.

3. **Deduplicate on `identifier`.** Delivery is at-least-once and Workera says so explicitly. Treat
   `identifier` as the idempotency key for score events. Note the one trap: on `assessment_started`
   the field is `assessment_identifier`, which is a DIFFERENT ID space from the `identifier` on
   `assessment_completed` (a domain score). The two do not correlate — do not join them.

4. **Route by event type.**
   - `assessment_started` — a learner began a baseline, mini or full reassessment. Does not fire on
     resume. Use it for started-but-not-completed reporting.
   - `assessment_completed` — the current flat score event. Carries `score`, `proficiency_level`,
     `initiative_type`, `skill_ratings[]` with `behaviors[]`.
   - `score_updated` — the older nested shape covering baseline and skill-boost. Workera describes
     `assessment_completed` as its v2 replacement but publishes no retirement date, so handle both if
     both are enabled for you.
   - `program_completed` — fires once per learner per program when required (non-elective)
     capabilities hit their target scores.
   - `self_score_completed` — a self-declared score. Keep it out of your verified-score field.
   - `appeal_approved` — a human reviewer changed a score. Carries `score_before_appeal` and
     `score_after_appeal`; the docs warn these are the delta from this appeal and are not necessarily
     the learner's current best score. Re-read the score rather than trusting the pair as current.

5. **Expect fields you do not know.** Workera states that the `assessment_started` field set is
   intentionally open and may gain fields non-breakingly. Parse permissively.

6. **Know why an event never arrived.** Delivery is silently skipped for a user with no resolvable
   enterprise employee association, and for any address on the company's webhook exclusion list. A
   missing event is not necessarily a missing assessment.

7. **Backfill what you lost.** There is no delivery log and no replay endpoint. Recover with
   `GET /api/v2/scores/{score_identifier}`
   (`WorkeraWebappsWeb.Rest.Controllers.V2.ScoresController.show`), which returns the same flat
   topic-level shape as `assessment_completed`. For a wider gap, page `GET /api/v2/scores` and
   reconcile against what you have stored.

## Rules

- These payloads carry `user.email` and the enterprise employee ID. Treat them as employee personal
  data on receipt.
- Errors on the REST backfill calls use `{code, message, type}`; rate limits are signalled with
  `x-ratelimit-*` and exhaustion is 429.
