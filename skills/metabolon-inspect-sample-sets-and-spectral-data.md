---
name: metabolon-inspect-sample-sets-and-spectral-data
description: Inspect the sample sets belonging to a Metabolon project and obtain a download link for the project's raw spectral data.
api: Metabolon Portal API
base_url: https://portal-api.prod.metabolon.com
operations:
  - SampleSetsInfo_GetSampleSetsInfos2
  - SampleSetsInfo_GetSampleSetsInfo2
  - SampleSetsInfo_GetSampleSetsCount2
  - SpectralData_Get2
  - SpectralData_GetDownloadLink2
  - Reports_GetReport
generated: '2026-08-25'
method: generated
source: openapi/metabolon-portal-api-openapi.yml
---

# Inspect Metabolon sample sets and raw spectral data

A Metabolon project is analysed as one or more **sample sets**. Raw mass-spectrometry data is delivered
separately from the processed result files.

## Steps

1. **Size the study.** `SampleSetsInfo_GetSampleSetsCount2`
   (`GET /api/v2/samplesetsinfo/project/{projectId}/count`) returns how many sample sets the project has.
   Use it before listing so you know what you are about to pull.

2. **List the sample sets.** `SampleSetsInfo_GetSampleSetsInfos2`
   (`GET /api/v2/samplesetsinfo/project/{projectId}`) returns `SampleSetInfo` records keyed on
   `projectId` + `sampleSetId`.

3. **Drill into one.** `SampleSetsInfo_GetSampleSetsInfo2`
   (`GET /api/v2/samplesetsinfo/{projectId}/{sampleSetId}`).

4. **Raw spectral data.**
   - `SpectralData_Get2` (`GET /api/v2/spectraldata/{projectId}`) returns
     `SpectralDataDownloadRecord` entries describing what is available.
   - `SpectralData_GetDownloadLink2` (`GET /api/v2/spectraldata/{projectId}/download`) returns a download
     link. Treat the link as short-lived: fetch it immediately, and re-request rather than caching it.

5. **QC report (optional).** `Reports_GetReport`
   (`GET /api/v1/reports/{reportId}/{projectId}`) returns a `PlatformQcReport`. This is the one operation
   in the whole contract published **only under `/api/v1`** — there is no v2 form. It declares `404`.

## Rules

- **Read-only skill.** No operation here writes.
- Raw spectral data files are large. Confirm the caller wants them before requesting a link — Metabolon
  states that unprocessed data is archived in the cloud and stored indefinitely, so there is no urgency
  and no reason to bulk-pull speculatively.
- Errors are RFC 7807 `ProblemDetails` over `application/json`.
- `sampleSetId` is scoped to its `projectId`; never carry one across projects.
