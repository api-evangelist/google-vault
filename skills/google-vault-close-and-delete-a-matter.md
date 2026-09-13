---
name: google-vault-close-and-delete-a-matter
description: Wind down a Google Vault matter safely — close it, delete it, and restore it within the 30-day window if the deletion was wrong.
api: Google Vault API
base_url: https://vault.googleapis.com
operations:
  - matters.get
  - matters.holds.list
  - matters.close
  - matters.delete
  - matters.undelete
  - matters.reopen
  - matters.list
scopes:
  - https://www.googleapis.com/auth/ediscovery
generated: '2026-09-12'
method: generated
source: openapi/google-vault-matters-api-openapi.yml, https://developers.google.com/workspace/vault/guides/matters
---

# Close and delete a matter, reversibly

This is the one flow on the Vault API where the undo path is fully documented. Use it, and know
exactly where it stops.

## Steps

1. **Inspect what the matter still holds.**
   `GET /v1/matters/{matterId}` (`matters.get`) with `view=FULL`, then
   `GET /v1/matters/{matterId}/holds` (`matters.holds.list`). Closing a matter affects the holds
   inside it — know what they are before you act.

2. **Close it.** `POST /v1/matters/{matterId}:close` (`matters.close`). Fully reversible:
   `POST /v1/matters/{matterId}:reopen` (`matters.reopen`) puts it back, with no stated deadline.

3. **Delete it, if that is really the intent.** `POST` is not the verb —
   `DELETE /v1/matters/{matterId}` (`matters.delete`). **Only a closed matter can be deleted**,
   which is why step 2 is not optional.

4. **Know the restore window.** Google's own guide states: *"A deleted matter remains in Trash
   for approximately 30 days, during which time it can be restored. After that period, the matter
   is permanently purged."* Within that window,
   `POST /v1/matters/{matterId}:undelete` (`matters.undelete`) brings it back with its data.

5. **Find a deleted matter you need to restore.** `GET /v1/matters?state=DELETED`
   (`matters.list`) lists matters in the `DELETED` state. Remember each call costs ten matter
   reads.

## Rules

- Deletion is soft: the matter moves to state `DELETED`, it is not removed. The four states are
  `STATE_UNSPECIFIED`, `OPEN`, `CLOSED`, `DELETED`.
- After roughly 30 days the purge is permanent and no API call recovers it.
- Deleting the matter does not give you back a hold you deleted earlier. `matters.holds.delete`
  has no inverse anywhere in this API.
- 404 on any of these means the matter id is wrong or not visible to this caller; 409 means the
  target is already in the state you asked for.
