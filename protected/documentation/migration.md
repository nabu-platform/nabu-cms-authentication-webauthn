# WebAuthn database migration (PostgreSQL)

Do not run this migration until the verifier module and project are deployed together. Review existing table naming and schema prefix in each environment. The current model ID is `database.model`; it references `database.types.credential` and `database.types.ceremony`. Synchronize only after reviewing its non-destructive DDL and ensuring the composite credential uniqueness migration is ready. The `crud.credential` and `crud.ceremony` artifacts are configured but do not create tables or replace atomic conditional JDBC updates.

```sql
-- For an existing schema, add the new nullable binding column before deploying the updated services:
ALTER TABLE webauthn_ceremonies ADD COLUMN enrollment_identity_id uuid;
-- For a fresh schema generated from database.model, this column is already present.
-- After creating both tables from the model, and before accepting writes:
CREATE UNIQUE INDEX webauthn_credentials_rp_credential_unique
ON webauthn_credentials (rp_id, credential_id);
CREATE UNIQUE INDEX webauthn_ceremonies_challenge_unique
ON webauthn_ceremonies (challenge);
CREATE INDEX webauthn_credentials_owner_rp_active
ON webauthn_credentials (owner_id, rp_id, revoked);
CREATE INDEX webauthn_ceremonies_expiry
ON webauthn_ceremonies (expires, consumed);
```

Existing duplicates must be investigated and resolved manually, never automatically reassigned to a different owner. Credential `owner_id` and `(rp_id, credential_id)` must remain immutable. The `consumeCeremony` conditional update and the matching insert/counter update must share one transaction and each return exactly one affected row. Do not expose JDBC services directly over REST. This is a migration definition, not an executed database change.
