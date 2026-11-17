# WebAuthn database migration (PostgreSQL)

This is application-owned PostgreSQL DDL guidance, not a Nabu migration to run automatically. Do not run it until the verifier module and application are deployed together. The application owns installing and evolving `database.model` (`database.types.credential` and `database.types.ceremony`), reviewing schema prefixes and generated non-destructive DDL, and adding the composite unique constraint before accepting credential writes. Nabu field metadata only requests single-field automatic indexes; it cannot enforce `(rp_id, credential_id)` uniqueness. `crud.credential` and `crud.ceremony` do not create tables or replace atomic conditional JDBC updates.

```sql
-- For an existing schema, add the new nullable binding column before deploying the updated services:
ALTER TABLE webauthn_ceremonies ADD COLUMN enrollment_identity_id uuid;
-- For a fresh schema generated from database.model, this column is already present.
-- After creating both tables from the model, before accepting credential writes:
-- Check for existing duplicate (rp_id, credential_id) pairs first.
CREATE UNIQUE INDEX webauthn_credentials_rp_credential_unique
ON webauthn_credentials (rp_id, credential_id);
-- The credential model confirms an automatic single-column owner_id index.
-- The ceremony model declares challenge unique; this uniqueness remains in place.
-- Finish reads use id (primary key).
-- A single-column expires index is recommended for expiry lookups and cleanup,
-- but is not verified in the current model.
```

Existing duplicates must be investigated and resolved manually, never automatically reassigned to a different owner. Credential `owner_id` and `(rp_id, credential_id)` must remain immutable. If owner/RP/active-credential reads require a composite `(owner_id, rp_id, revoked)` index after measuring query plans, the application may install it instead of the redundant automatic `owner_id` single-column index; do not blindly install both. The current model does not automatically index `rp_id` or `credential_id`; their index edits were not persisted. The application-managed unique index on `(rp_id, credential_id)` is required to enforce credential uniqueness. Add separate single-column indexes only if application query patterns need them. The `consumeCeremony` conditional update and the matching insert/counter update must share one transaction and each return exactly one affected row. Do not expose JDBC services directly over REST. This is a migration definition, not an executed database change.
