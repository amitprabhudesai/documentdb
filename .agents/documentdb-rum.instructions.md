---
description: 'Extended RUM component and its generic SQL API boundary.'
applyTo: 'pg_documentdb_extended_rum/**'
---

# DocumentDB extended RUM

Owns requirements for `pg_documentdb_extended_rum/` and callers that expose its implementation details. Load [PostgreSQL safety](postgres.instructions.md), [shared backend contracts](documentdb-common.instructions.md), and the applicable references selected by [DocumentDB backend selection](documentdb.instructions.md#instruction-selection).

## Extended-RUM Semantics in OSS SQL Surface

The OSS `pg_documentdb` SQL surface must not expose UDFs whose purpose is specific to extended RUM internals.

Flag as ❌ **error** any new OSS SQL UDF that:
- Requires a consuming extension to wait for a generic SQL schema release before using an internal helper that could be a direct C API.
- Bakes consumer-specific or extended-RUM-specific decoded semantics into the generic OSS SQL API.
