---
description: 'Instruction entry point for the foundational DocumentDB core extension.'
applyTo: 'pg_documentdb_core/**'
---

# DocumentDB core extension

For `pg_documentdb_core/`, load [PostgreSQL safety](postgres.instructions.md) and [shared backend contracts](documentdb-common.instructions.md).

Use [DocumentDB backend selection](documentdb.instructions.md#instruction-selection) for affected consumers and compatibility. A change to shared BSON representations, SQL interfaces, or hooks can require another component's guidance even when only core files change.
