---
name: google-vault-size-and-run-an-export
description: Size a Google Vault search with a count operation, then run and collect an export of the matching data.
api: Google Vault API
base_url: https://vault.googleapis.com
operations:
  - matters.count
  - operations.get
  - matters.savedQueries.create
  - matters.exports.create
  - matters.exports.get
  - matters.exports.list
scopes:
  - https://www.googleapis.com/auth/ediscovery
generated: '2026-09-12'
method: generated
source: openapi/google-vault-exports-api-openapi.yml, openapi/google-vault-matters-api-openapi.yml, openapi/google-vault-operations-api-openapi.yml, discovery/google-vault-discovery-v1.json
---

# Size a search, then export

Exports are the expensive operation on this API: an organization may only have **20 in progress
at once**, and export writes are capped at 20 per minute per project with `exports.create` alone
costing ten of them. Size first.

## Steps

1. **Rehearse with a count.** `POST /v1/matters/{matterId}:count` (`matters.count`) with the
   `Query` you intend to export. This is the closest thing to a dry run on this API: it reports
   what the query would match without creating a hold or an export.

2. **Poll the operation.** `matters.count` returns a long-running `Operation`. Call
   `GET /v1/operations/{operationsId}` (`operations.get`) until `done` is true, then read
   `response` — or `error`, which is where a count failure surfaces. A caller that only checks
   the HTTP status of `matters.count` will not see the failure.

3. **Save the query if it will be reused.**
   `POST /v1/matters/{matterId}/savedQueries` (`matters.savedQueries.create`). The same `Query`
   object drives holds, saved queries and exports, so a query you have counted can be reused
   verbatim.

4. **Create the export.** `POST /v1/matters/{matterId}/exports` (`matters.exports.create`) with
   the `query` and `exportOptions`. Check `exports.list` first if there is any chance a previous
   attempt already started — there is no idempotency key, and a duplicate export burns one of
   the 20 organization-wide slots.

5. **Poll for completion.** `GET /v1/matters/{matterId}/exports/{exportId}`
   (`matters.exports.get`) until `status` leaves `IN_PROGRESS` and becomes `COMPLETED` or
   `FAILED`. Exports do NOT use the `operations` resource.

6. **Collect the output.** A completed export exposes `cloudStorageSink.files` — the Google Cloud
   Storage objects holding the exported data. This is the only point at which the Vault API hands
   back content rather than metadata; treat those files as the sensitive artifact they are.

## Rules

- `matters.exports.list` costs 5 export reads per call; `exports.get` costs 1. Poll with `get`
  on a known id, never by re-listing.
- `matters.exports.delete` removes the export and its results and cannot be undone. It does not
  roll back anything the export did; it destroys the artifact.
- Both the count and the export require the full `ediscovery` scope. The read-only scope cannot
  call `matters.count` even though it reads.
