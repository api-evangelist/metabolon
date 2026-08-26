---
name: metabolon-check-service-health
description: Poll the unauthenticated health tier of the four Metabolon services to determine whether the portal and its bioinformatics services are up, before or during a support investigation.
api: Metabolon Portal API
base_url: https://portal-api.prod.metabolon.com
operations:
  - Health_HealthCheck2
  - Health_ShallowPing2
  - Health_DeepPing2
  - Health_DatadogHttpCheck2
  - Health_DatadogServiceCheck2
generated: '2026-08-25'
method: generated
source: openapi/metabolon-portal-api-openapi.yml
---

# Check Metabolon service health

Metabolon operates **no public status page** — `status.metabolon.com` does not resolve. The health tier on
each service is the only availability signal available, and unlike everything else in this contract it
answers without authentication.

## The four services

| Service | Host | Contract |
|---|---|---|
| Portal API | `https://portal-api.prod.metabolon.com` | `openapi/metabolon-portal-api-openapi.yml` |
| Discovery Panels | `https://discovery.prod.metabolon.com` | `openapi/metabolon-discovery-panels-api-openapi.yml` |
| Pathway Explorer | `https://pathway.prod.metabolon.com` | `openapi/metabolon-pathway-explorer-api-openapi.yml` |
| Heatmap | `https://heatmap.prod.metabolon.com` | `openapi/metabolon-heatmap-api-openapi.yml` |

All four expose the identical five health operations under both `/api/v1` and `/api/v2`.

## Steps

1. **Shallow ping first.** `Health_ShallowPing2` (`GET /api/v2/health/ping`) returns a
   `ShallowPingResponse` with a `HealthEnum`. Cheap; use it for a liveness sweep across all four hosts.
2. **Deep ping when shallow is healthy but behaviour is wrong.** `Health_DeepPing2`
   (`GET /api/v2/health/deepping`) returns `DeepPingStatus`, which includes `PostgresDeepPingStatus` —
   this is what distinguishes "service up, database unreachable" from "service down".
3. **Overall check.** `Health_HealthCheck2` (`GET /api/v2/health`).
4. **Datadog forms.** `Health_DatadogHttpCheck2` and `Health_DatadogServiceCheck2` return `DatadogResponse`
   shapes intended for a monitor. Prefer them only if you are wiring a monitor, not diagnosing.

## Rules

- **Poll gently.** These endpoints are unauthenticated, which makes them easy to abuse. A sweep every
  30–60 seconds is ample; there is no published limit and no `RateLimit-*` header to tell you when you
  have crossed one.
- `Health_ShallowPing2` declares `500` with the description "The request generated an error on the server".
  A `500` here is itself the signal — report it, do not retry it away.
- A healthy Portal API says nothing about the Discovery, Pathway or Heatmap services. Check each host you
  actually depend on.
- These are the only Metabolon operations that do **not** require an Auth0 token. Do not assume the same of
  anything else in the contract.
