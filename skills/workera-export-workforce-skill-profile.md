---
name: workera-export-workforce-skill-profile
description: >-
  Pull a company's verified capability scores out of Workera and join them to the enterprise HRIS
  employee record, so a workforce skills profile can be built or refreshed in another system.
api: Workera API
generated: '2026-09-04'
method: generated
source: openapi/workera-api-openapi.json
operations:
  - WorkeraWebappsWeb.Rest.Controllers.PingController.ping
  - WorkeraWebappsWeb.Rest.Controllers.DomainController.all
  - WorkeraWebappsWeb.Rest.Controllers.V2.ScoresController.index
  - WorkeraWebappsWeb.Rest.Controllers.V2.ScoresController.show
  - WorkeraWebappsWeb.Rest.Controllers.V2.SelfRatingsController.index
---

# Export a workforce skill profile from Workera

Read-only. Nothing in this skill changes state in Workera.

## Before you start

- You need a company-scoped API key. Keys are issued by a Workera CSM to enterprise customers; there
  is no self-serve signup and no way to obtain one programmatically.
- Every request carries `authorization: Bearer <key>` over HTTPS.
- Base host `https://skills.workera.ai`. Capabilities live on `/api/v1/`, scores on `/api/v2/`.

## Steps

1. **Check the credential.** `GET /api/v1/ping`
   (`WorkeraWebappsWeb.Rest.Controllers.PingController.ping`). A 200 returns `{"pong": true}`. A 401
   means the key is missing or wrong; a 403 means the key is valid but lacks the scope the endpoint
   needs. Do not continue past a 401.

2. **Load the capability catalog.** `GET /api/v1/domains`
   (`WorkeraWebappsWeb.Rest.Controllers.DomainController.all`). Page with `limit` (default 10, max
   100) and `next_page_after`; keep going while `has_more` is true, following `next_page`. Index the
   results by `identifier`. `title` is the human name, `is_signature_domain` marks a Workera Signature
   Capability, and `is_shared` marks a capability another company shared with yours.

3. **Pull the scores.** `GET /api/v2/scores`
   (`WorkeraWebappsWeb.Rest.Controllers.V2.ScoresController.index`), paging the same way. Optionally
   filter with `source` (`baseline_assessment`, `mini_assessment`, `full_reassessment`).

4. **Join to the enterprise employee.** Each score carries
   `user.employee.identifier` — the customer's own HRIS employee ID. That is the join key back into
   Workday, SAP SuccessFactors or Oracle HCM. `user.identifier` is Workera's internal ID and is not
   meaningful outside Workera.

5. **Decide what counts as high-stakes.** Each score carries `initiative_type`
   (`skills_evaluation`, `skills_growth`, `limited_disclosure`, `benchmark`, or `null` for a
   standalone assessment). Workera's own guidance: `skills_evaluation` is the canonical high-stakes
   type, and the consumer decides which other types they treat as high-stakes. Do not silently
   collapse them.

6. **Do not mix assessed scores with self-ratings.** `GET /api/v2/self-ratings`
   (`WorkeraWebappsWeb.Rest.Controllers.V2.SelfRatingsController.index`) returns learner
   self-declared scores. They are a separate entity on purpose — the whole product premise is that
   verified scores are not self-reported ones. Keep them in separate fields downstream.

7. **Fetch detail only where you need it.** `GET /api/v2/scores/{score_identifier}`
   (`...V2.ScoresController.show`) returns the flat, topic-level shape with `skill_ratings[]`
   (rating 1-4) and their `behaviors[]`. A 404 means the identifier does not exist for your company.

## Reading the numbers

- `score` is 0-300.
- `proficiency_level` is one of `beginner`, `developing`, `accomplished`, `expert`.
- `skill_ratings[].rating` is 1-4 and is a different scale from `score`. Do not average them together.

## Rules the API enforces

- **Rate limits.** Per API key. Read `x-ratelimit-remaining` on every response; when it reaches 0,
  sleep for `x-ratelimit-reset` seconds before the next call. Exhaustion returns 429. No numeric limit
  is published, so the headers are the only source of truth.
- **Errors.** `{"code": ..., "message": ..., "type": ...}` as `application/json` — not RFC 9457
  problem+json. Branch on `code`; log `message`. See `errors/workera-problem-types.yml`.
- **Pagination.** Never assume one page. `has_more` plus `next_page` is the contract.
- **Idempotency and reversibility.** Not applicable — this whole flow is GETs. There is nothing to
  replay and nothing to undo.
