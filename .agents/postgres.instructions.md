---
description: 'General PostgreSQL and C safety for backend extension work.'
applyTo: 'pg_documentdb/**,pg_documentdb_core/**,pg_documentdb_extended_rum/**'
---

# PostgreSQL and C extension safety

These are the existing general safety requirements extracted from the backend guidance. Read the trusted [operating policy](instructions.md) and select repository/component contracts through [DocumentDB backend selection](documentdb.instructions.md#instruction-selection). Use the [PostgreSQL testing strategy](postgres.TESTING.md) for validation and load gateway references when an SQL-to-wire contract changes.

## C Extension Safety

- All header files must have the `#ifndef` header macro where the header macro must have a unique name that describes the intent of the header file.
- Use `palloc`/`palloc0` rather than `malloc` so allocations are tied to the surrounding memory context.
- Never assume SQL-level inputs to C UDFs are non-NULL unless the function is declared `STRICT` in the SQL definition. Even array elements inside a non-NULL array can be NULL.
- Do not rely on `Assert()` for critical invariants that can be triggered by user input - `Assert` is a no-op in release builds. Use `ereport(ERROR, ...)` for any NULL check or invariant whose violation would cause a crash.
- foreach_delete_current must assign the output of the macro back to the list that it's deleting from. No code shall reference the list after the foreach_delete_current.
- Call `CHECK_FOR_INTERRUPTS()` inside long-running loops and recursive walks so a query stays cancellable, and `check_stack_depth()` in recursive functions so deep input cannot overflow the stack.
- Allocate results in a memory context that remains alive until their last use. Switch into per-row, per-batch, or per-attempt contexts before allocating the data they should reclaim, including setup strings. Do not use a longer-lived context such as `PortalContext` unless the data must survive that long.
- Keep resource release and ownership bookkeeping together. Prefer a small helper for work between relation/index open and close, so a new early return cannot bypass cleanup. Do not expose a lower-level unlock helper that permits skipping the associated state reset. Preserve intentional transaction-scoped lock lifetimes rather than releasing those locks early.
- Keep the timeout enabled when using a lock helper that promises a maximum wait. If an operation must wait indefinitely, explain why that cannot prevent other required work from progressing.
- Give a new PostgreSQL varlena base type its own representation and type-specific argument/Datum macros rather than treating it as `bytea`. Check text input/output and binary send/receive separately, including COPY and coordinator/worker parameters. Validate decoded sizes in backend C code; direct SQL and binary input can bypass wire-protocol size limits.
- Release no-longer-needed detoasted argument copies with `PG_FREE_IF_COPY` after their last use on normal returns. Do not free borrowed inputs or storage backing returned/retained values. Pass native varlena values through compatible helpers rather than making avoidable C-string conversions.
- For externally supplied identifiers that must fit PostgreSQL `NameData`, reject byte lengths at or above `NAMEDATALEN` before truncating conversions or catalog calls. Do not let distinct names alias by truncation, or impose this limit on public names that use a different backing representation.
- Do not broaden access to a sensitive catalog base table merely to support a metadata query. Prefer a suitable catalog view or narrowly scoped C UDF with the required fields and execution identity; establish why direct access is necessary before granting it.

## SQL Construction

- SQL expression and clause builders must not append `;`; only the function building the complete statement should do that. Keep externally supplied values parameterized. If a query calls a function that writes data or changes state, verify how often it executes rather than assuming a CTE makes it run exactly once. Prefer separate small SPI statements in the same transaction when combining them adds no required behavior.
- Quote dynamic SQL identifiers with the established identifier-formatting helper; value parameters do not quote identifiers. Always schema-qualify SQL operators, for example `OPERATOR(pg_catalog.=)`, or use a schema-qualified function with equivalent behavior. Use schema-qualified function calls as well. Never rely on the caller's `search_path` to resolve functions or operators.
- When old and new SQL signatures share a C entry point, use `PG_NARGS()` (or the established argument-count check) before accessing newly added argument slots and retain the old-arity behavior. This protects calls through an existing signature; it does not make a new SQL object/signature available.

## Error Handling and Temporary Session State

- Do not add `PG_TRY`/`PG_CATCH` to previously straight-line code solely to restore session state when an escaping error tears down the dedicated session. Verify that lifecycle first. For background index-build sessions, prefer save/set/call/reset on the normal-return path instead of another exception wrapper.
- Avoid `volatile` locals, error-state bookkeeping, and catch-and-rethrow scaffolding introduced only to support an unnecessary wrapper. Reuse a small helper for symmetric set/reset operations when appropriate.
- Preserve required error handling. Add exception cleanup only when the error path needs it, such as restoring state in a surviving session or releasing resources not reclaimed by session teardown. Do not assume reusable client sessions have the dedicated-worker lifecycle.
