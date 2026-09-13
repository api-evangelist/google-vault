---
name: google-vault-open-matter-and-place-hold
description: Open a Google Vault matter and place a legal hold on the accounts or org unit it covers, using the Vault API v1.
api: Google Vault API
base_url: https://vault.googleapis.com
operations:
  - matters.create
  - matters.get
  - matters.addPermissions
  - matters.holds.create
  - matters.holds.list
  - matters.holds.addHeldAccounts
  - matters.holds.get
scopes:
  - https://www.googleapis.com/auth/ediscovery
generated: '2026-09-12'
method: generated
source: openapi/google-vault-matters-api-openapi.yml, openapi/google-vault-holds-api-openapi.yml, discovery/google-vault-discovery-v1.json
---

# Open a matter and place a hold

Placing a hold is a legal act. Everything below is a real write against a production Google
Workspace tenant — there is no sandbox and no dry-run flag.

## Before you start

- You need an OAuth 2.0 access token with `https://www.googleapis.com/auth/ediscovery`. The
  read-only scope cannot create anything.
- The acting user must hold Vault privileges in the Admin console. Scope alone is not enough.
- There is no idempotency key on this API. If a create times out, LIST before you retry.

## Steps

1. **Create the matter.** `POST /v1/matters` (`matters.create`) with `name` and `description`.
   Keep the returned `matterId`; every later call is addressed through it.

2. **Confirm it landed before retrying anything.** If step 1 returned an ambiguous error, call
   `GET /v1/matters` (`matters.list`) and match on `name` rather than firing `matters.create`
   again — a retry creates a second matter. Note `matters.list` costs 10 matter reads against a
   600-per-minute organization-wide budget, so do this once, not in a loop.

3. **Grant collaborators, if any.** `POST /v1/matters/{matterId}:addPermissions`
   (`matters.addPermissions`) with the `accountId` and `role`. Reversible with
   `matters.removePermissions`.

4. **Create the hold.** `POST /v1/matters/{matterId}/holds` (`matters.holds.create`). A hold
   carries exactly one `corpus` — `MAIL`, `DRIVE`, `GROUPS`, `HANGOUTS_CHAT`, `VOICE`,
   `CALENDAR` or `GEMINI` — and targets EITHER an `orgUnit` OR an explicit `accounts` list, not
   both. Add a `query` to narrow what is preserved.

5. **Add accounts later if needed.** `POST /v1/matters/{matterId}/holds/{holdId}:addHeldAccounts`
   (`matters.holds.addHeldAccounts`). Reversible with `matters.holds.removeHeldAccounts`.

6. **Verify.** `GET /v1/matters/{matterId}/holds/{holdId}` (`matters.holds.get`) and confirm the
   corpus, the query and the account list are what you intended.

## Rules

- **Never call `matters.holds.delete` to undo a mistake in this flow.** Deleting a hold is
  irreversible — there is no restore method in the API — and it releases the preserved data back
  to normal retention. To narrow a hold, remove accounts or update the query instead.
- Hold writes are capped at 60 per minute per project; each `holds.create` also spends a matter
  read, a matter write and a hold read.
- Errors arrive as `{"error": {"code", "message", "status"}}` (google.rpc.Status), not RFC 9457.
  409 means the resource already exists — usually your earlier attempt succeeded.
- On 429, back off with `min(((2^n)+random_ms), 32–64s)`. No `Retry-After` header is sent.
