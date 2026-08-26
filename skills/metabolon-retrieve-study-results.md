---
name: metabolon-retrieve-study-results
description: Locate a Metabolon metabolomics study in the MyMetabolon portal and download its delivered result files, individually or as a zip bundle.
api: Metabolon Portal API
base_url: https://portal-api.prod.metabolon.com
operations:
  - Projects_Retrieve_AllDetails2
  - Projects_Retrieve_DetailsByCode2
  - Projects_Retrieve_IsEmpty2
  - Files_GetFilesByProjectId2
  - Files_SearchFilesByProjectId2
  - Files_DownloadFile2
  - Files_DownloadFileBundle2
  - Files_DownloadProjectFileBundleGET2
generated: '2026-08-25'
method: generated
source: openapi/metabolon-portal-api-openapi.yml
---

# Retrieve Metabolon study results

Metabolon delivers metabolomics results as files attached to a **project** (one customer study). This skill
gets you from "which study?" to the files on disk.

## Before you start

- **Authentication.** Every operation requires an Auth0 bearer token from `https://auth0.metabolon.com/`
  and a MyMetabolon account with access to the specific project. The published OpenAPI declares no
  `securitySchemes`; send `Authorization: Bearer <token>` regardless. See
  `authentication/metabolon-authentication.yml`.
- **Entitlement is per project, not per scope.** Token scopes are stock OIDC identity scopes and grant no
  API access. Your projects come from `UserProject` membership. If a project id 404s, it is far more
  likely that you are not entitled to it than that it does not exist.
- **Use `/api/v2`.** Every operation is published twice, under `/api/v1` and `/api/v2`, schema-identical.
  Prefer v2; the operationIds above are the v2 forms.

## Steps

1. **Find the project.**
   - If you know the human-facing code, call `Projects_Retrieve_DetailsByCode2`
     (`GET /api/v2/projects/projectCode/{projectCode}`).
   - Otherwise call `Projects_Retrieve_AllDetails2` (`GET /api/v2/projects`) and match on the returned
     `ProjectDescription`. This operation takes **no pagination parameters** and returns your full
     entitled set — expect the whole list in one response.
   - Keep the `projectId` (uuid). It is the path parameter for everything downstream.

2. **Check the project actually has deliverables.**
   Call `Projects_Retrieve_IsEmpty2` (`GET /api/v2/projects/empty/{projectId}`). An empty project means the
   study has not been delivered yet — stop here rather than reporting "no results found".

3. **List the files.**
   `Files_GetFilesByProjectId2` (`GET /api/v2/files/project/{projectId}`) returns `ProjectFile` records
   carrying `fileId`, `projectId` and file metadata. To narrow, use `Files_SearchFilesByProjectId2`
   (`GET /api/v2/files/project/{projectId}/search/{searchText}`) — note the search term is a **path
   segment**, so URL-encode it.

4. **Download.**
   - One file: `Files_DownloadFile2` (`GET /api/v2/files/download/{projectId}/{fileId}`).
   - A chosen set: `Files_DownloadFileBundle2` (`POST /api/v2/files/download/{projectId}/fileIds`) with the
     file ids in the body. Returns a zip.
   - Everything: `Files_DownloadProjectFileBundleGET2` (`GET /api/v2/files/download/{projectId}`). Returns a
     zip of the whole project — for a large study this is a big response, so prefer the id list when you
     know what you want.

## Rules

- **Read-only skill.** Nothing here mutates. Do not reach for the `/api/v2/admin/files/*` operations to
  "clean up" — `Files_RemoveFile2` has no reversal (see `conventions/metabolon-conventions.yml`).
- **Errors are RFC 7807 `ProblemDetails`,** returned as `application/json` (not `application/problem+json`).
  Read `title` and `detail`. `404` is declared on the file and report operations; `500` and `502` on the
  bundle paths. See `errors/metabolon-problem-types.yml`.
- **No retry signal.** No `Retry-After` or `RateLimit-*` header is declared anywhere. The download
  operations do not declare `429`, but back off exponentially on any `5xx` rather than hammering.
- **No pagination on the list operations.** Do not invent `page`/`limit` parameters for
  `Files_GetFilesByProjectId2` — they do not exist and will be ignored.
