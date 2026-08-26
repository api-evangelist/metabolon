---
name: metabolon-build-study-estimate
description: Use the Metabolon Study Builder to produce an indicative price estimate for a metabolomics study, save a draft, and email a quote summary to Metabolon sales.
api: Metabolon Portal API
base_url: https://portal-api.prod.metabolon.com
operations:
  - StudyBuilder_PostEstimate2
  - StudyBuilder_PostDraft2
  - StudyBuilder_GetDraft2
  - StudyBuilder_PostQuoteSummaryEmail2
generated: '2026-08-25'
method: generated
source: openapi/metabolon-portal-api-openapi.yml
---

# Build a Metabolon study estimate

Metabolon publishes no price list — `https://www.metabolon.com/pricing/` is a 404 and studies are quoted
per project (`plans/metabolon-plans-pricing.yml`). The Study Builder is the only pricing surface, and it
is inside the portal.

## Steps

1. **Request an estimate.** `StudyBuilder_PostEstimate2` (`POST /api/v2/study-builder/estimate`) with a
   `StudyBuilderEstimateRequestDto` body. The contract states it will "calculate a fresh indicative
   estimate from the pricing matrix, persist the wizard inputs and result to
   StudyBuilderEstimateRequestRecords, then return the estimate."

2. **Save a draft** (pre-registration flow only). `StudyBuilder_PostDraft2`
   (`POST /api/v2/study-builder/draft`) saves selections **without pricing** and returns an opaque
   `draftToken` used to link the draft to an Auth0 signup. Retrieve it later with `StudyBuilder_GetDraft2`
   (`GET /api/v2/study-builder/draft`).

3. **Send it to sales.** `StudyBuilder_PostQuoteSummaryEmail2` (`POST /api/v2/study-builder`) emails the
   quote summary to the submitter **and to Metabolon's internal sales inbox**.

## Rules — read these before calling anything

- **`StudyBuilder_PostEstimate2` IS NOT IDEMPOTENT, and the contract says so explicitly:** "Each call
  creates a new stored request; this route does not load a previously saved estimate." There is no
  `Idempotency-Key` header on this API. A retried or duplicated call creates a duplicate persisted record.
  Never retry it blindly on a timeout — you cannot tell whether the first call landed.
- **`StudyBuilder_PostQuoteSummaryEmail2` sends real email to a human sales team.** It has no dry-run, no
  preview and no recall. Call it once, only on explicit user instruction, and never as part of a retry loop.
- **These four operations are the rate-limited ones.** `429` is declared on the estimate, draft and
  quote-summary operations. No `Retry-After` or `RateLimit-*` header is returned, so on a `429` back off
  with exponential jitter and do not retry immediately.
- `503` is declared on the quote-summary operation. `500` means the estimate failed to persist and **no
  estimate was returned** — the contract states this; do not report a partial result.
- The estimate is **indicative**, not a quote. Present it as such. Business Line resolves from Salesforce
  for authenticated users; guests are referred to sales.
