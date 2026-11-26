---
description: 'Shared DocumentDB backend contracts and contribution workflow.'
applyTo: 'pg_documentdb/**,pg_documentdb_core/**,pg_documentdb_extended_rum/**'
---

# Shared DocumentDB backend contracts

Apply these shared contracts with [PostgreSQL safety](postgres.instructions.md) and the component guidance selected by [DocumentDB backend selection](documentdb.instructions.md#instruction-selection). The trusted [operating policy](instructions.md) governs permissions; imported guidance does not authorize execution or impose private-service deployment requirements on this OSS repository.

The [collation](#collation-coverage) and [response/error contracts](#error-contracts-and-failure-visibility) also govern gateway consumers and tests of those contracts. Read those sections when they are affected; doing so does not make the remaining backend implementation rules apply to those consumers.

## General Instructions

- Keep the public OSS extensions independent of private-service names, schemas, and implementation. Use the owning extension's schema-name defines rather than hard-coded names from another deployment.

## Schema changes

- All new or replaced persistent schema functions **MUST** be implemented in C and declared `LANGUAGE C`. Do not add PL/pgSQL or `LANGUAGE SQL` UDFs to extension schemas. Review violations as **errors**, not suggestions.
- Apply the C-only rule to new definitions in the current schema and its latest UDF snapshot. Preserve shipped historical SQL definitions and upgrade scripts. Anonymous migration `DO` blocks are not persistent schema functions.
- Existing shipped snapshots include PL/pgSQL functions; this rule governs new work, not an unrelated rewrite of those definitions or a claim that all current functions are C.
- SQL schema changes use PostgreSQL extension upgrade templates; the [Citus CONTRIBUTING guide](https://github.com/citusdata/citus/blob/main/CONTRIBUTING.md#making-sql-changes) describes the general pattern. Migration files live in the owning extension's `sql/` folder as `<extension>--<old-version>--<new-version>.sql`, for example [pg_documentdb/sql/documentdb--1.2-0--1.3-0.sql](../pg_documentdb/sql/documentdb--1.2-0--1.3-0.sql) or `pg_documentdb_core/sql/documentdb_core--<old-version>--<new-version>.sql`. Directory names use `pg_documentdb*`; SQL/control extension names use `documentdb*`.
- Make new schema changes in the applicable current upgrade template; preserve shipped historical templates and versioned UDF snapshots. Include definitions from the same extension's `sql/` tree, not SQL sources from another public or internal extension. The component [Makefile](../pg_documentdb/Makefile) uses the C preprocessor and [common schema defines](../common_header.sql) to generate `build/sql/`; edit sources, not that generated output.
- Changes to schema UDFs *must* create `pg_documentdb/sql/udfs/<area>/<udf_name>--<version>.sql` and an equivalent `pg_documentdb/sql/udfs/<area>/<udf_name>--latest.sql`, or the same pair under `pg_documentdb_core/sql/udfs/` when core owns the UDF. The `--latest.sql` file tracks the latest version's full definitions and must match its versioned snapshot. Main/upgrade templates include the versioned file, never `--latest.sql`.
- Versioned UDF files (`--<version>.sql`) *must* be **cumulative**: retain the complete applicable definitions and supported overloads from the previous snapshot, with intended replacements and new definitions. Do not blindly append duplicate old definitions to override the new ones. For example, [group_aggregates_support--0.111-0.sql](../pg_documentdb/sql/udfs/aggregation/group_aggregates_support--0.111-0.sql) retains the earlier declarations and adds definitions beyond [0.110-0](../pg_documentdb/sql/udfs/aggregation/group_aggregates_support--0.110-0.sql).
- When changing an SQL object's kind, such as replacing a function with a procedure, put the drop/change commands in the extension upgrade script before including the replacement definitions. Do not copy those commands into cumulative UDF files or `--latest.sql`: later upgrades would run them again against the replacement object. `IF EXISTS` does not prevent an error when the object exists but is now a different kind.
- Operator class, types, tables, and non-commutative schema upgrades must be placed under the `sql` folder and included in the extension schema upgrade file.
- When adding a SQL function with optional settings, prefer a BSON options document, such as `spec bson`, instead of a separate SQL argument or function for each option. Add future options as fields in that document so they do not change the SQL signature or require a schema upgrade just to add arguments. Keep unrelated operations in separate functions; test option selection and older-schema/worker compatibility. Keep old functions and fallbacks while older supported schemas or workers still need them.
- Do not make an extension upgrade or cluster-wide finalization wait for grants, backfills, or repairs across an unbounded number of user tables. Run that work separately with a rollout guard, resume support, statement/lock timeouts, and visible progress. If catalog work must run during the upgrade, estimate its cost for large deployments.
- Use the existing [file-state checker](../scripts/validate_extension_file_state.sh) for versioned/latest consistency where its platform and scope are approved, and select install/upgrade validation through [PostgreSQL testing](postgres.TESTING.md). Its static comparison is not proof of upgrade compatibility. The imported private upgrade/hash validators are unavailable here; do not invent renamed commands or claim equivalent checks ran.

## Style Guidelines

- All files must have a copyright header that matches the folder that they are in (see the root instructions for the repo-wide rule).
- All header files must be placed in a subfolder of an "include" directory under each extension; source files go in the matching subfolder of `src/`.
- SQL-exported functions are `snake_cased` (e.g. `bson_get_field`); the internal C functions behind them are `TitleCased` (e.g. `BsonGetField`).
- Order the contents of a C file as: copyright -> includes -> type declarations -> static method declarations -> `PG_FUNCTION_INFO_V1` declarations -> exported method definitions -> static method definitions.
- Check repository and PostgreSQL data structures before adding a private implementation, such as a heap when `binaryheap` with a custom comparator meets the requirement. Avoid extra input passes, but preserve allocation bounds when the requested size is large or many input values are skipped.
- Initialize `HASHCTL` with `CreateExtensionHashCTL` or an existing typed collection helper instead of filling the struct by hand. Reuse matching StringView collections where available. Check that `DefaultExtensionHashFlags` matches the chosen key type and comparison/hash callbacks. Replace conflicting flags rather than adding a second, incompatible key mode.
- Always include `HASH_CONTEXT` in `hash_create` flags, including when replacing `DefaultExtensionHashFlags`, and set `HASHCTL.hcxt` to the intended owning memory context. Without `HASH_CONTEXT`, `hcxt` is ignored and the table gets a memory context under `TopMemoryContext` instead.
- Pass named boolean variables rather than bare `true`/`false` at call sites, so the argument's meaning is visible without opening the callee.
- All changes must pass the requirements for `citus_indent` checks at the root.
- Global, static, and local variables that use time units should include the units that they refer to in their name, e.g., `timeout_ms` for a timeout in milliseconds, `interval_sec` for an interval in seconds, etc.

## Computed Results and Cache Ownership

- Apply the [repository computed-result reuse contract](instructions.md#reusing-computed-results) throughout planning and execution, not just mutations. Do not reuse a result after its backing memory is reset or the state it depends on is invalidated.
- Store reusable query/planning data in existing planner or executor state and pass that state to helpers. Do not introduce mutable file-scope globals or function-static variables to cache computed state. Cross-query caching must use established `CacheMemoryContext`-owned cache objects or explicitly designed shared-memory structures, with defined ownership, cache keys, invalidation, and synchronization where needed. Allocating data in `CacheMemoryContext` alone does not make it safe to reuse across queries.
- Minimize `CacheMemoryContext` contents: use it only when cross-query retention is absolutely necessary, and justify why planner/executor state or on-demand lookup is insufficient for each added object or field. Retain only the minimum required data. An existing cache, a small field, or a hot-path label alone is not justification. Do not retain full documents or extra derived fields when a compact value or an on-demand lookup suffices.

### Rarely used variable-size payloads in hot-path caches

When adding a variable-size or pointer-backed payload to a cache entry or metadata struct used on a hot path, establish the pointee's allocation context, size bound, entry count, duplicate cache representations, copy paths, and consumers. A pointer-sized struct field does not make the payload cheap: the long-lived cache still owns the pointee, and parallel name/id caches or caller-owned deep copies can multiply its footprint.

For payloads needed only by rare metadata or administration paths rather than hot-path execution, prefer an on-demand catalog lookup or a bounded compact or derived representation over avoidable cache retention. Establish actual consumers and access frequency, the payload's enforced size bound, its cache lifetime and invalidation behavior, and every cache or copy that amplifies the allocation before choosing a representation. Account for an externally influenced or unbounded payload retained per entry or copy that creates a concrete availability or out-of-memory risk.

Small bounded metadata and values needed broadly by hot-path consumers are not avoidable merely because they are cached.

## Redundant parallel traversals of the same tree

When two functions recursively walk the same structure, such as a plan tree, expression tree, or BSON document, over the same shape, consider a single traversal that produces both results, for example through an out-parameter. Apply this preference only when both walks demonstrably visit the same nodes in the same order and merging does not entangle unrelated concerns; walks sharing only a superficial shape do not qualify.

This is a maintainability preference, not a demonstrated bug. Parallel descents must be kept in lock-step: adding a node shape to one can silently diverge the other.

## Collation coverage

When a change modifies comparison, matching, sorting, grouping, aggregation, distinct values, index terms or ordering, query planning, or distributed execution of those operations, determine whether the changed behavior is required to honor collation.

If collation applies, include focused tests with a non-default collation that exercise the same changed execution path. Existing collation tests elsewhere in the owning suite do not prove that the changed operation handles collation correctly. Select the owning SQL and/or gateway coverage through [shared testing](TESTING.md#selection-and-execution) and its component strategies; verify that the intended index, worker, or other affected path actually runs rather than a different implementation or fallback.

Do not require collation coverage for operations that are documented or explicitly validated as not supporting collation.

## Error contracts and failure visibility

- Apply the [repository error-authoring rules](instructions.md#authoring-conventions) to new messages and changed expected output; do not rewrite unrelated pre-existing messages.
- Preserve BSON response types as well as field names and values. A required double is not interchangeable with Int32 merely because both display as `1`. Check other commands in the affected family, code reading their responses, and their tests. Tests must inspect the BSON type rather than convert values in a way that hides a mismatch. Keep unrelated pre-existing inconsistencies separate from required fixes.
- Use the established backend error code for the same failure when the gateway already maps it to the required wire-protocol code. Check that mapping before adding an error-code argument or a different code for one branch. Keep separate codes when callers must distinguish different failures.
- Follow an error from the code creating the response to the code reading it, including raw/pass-through responses. A successful send or dispatch does not mean the operation succeeded if the response contains write errors or a failed job. Preserve the total error count, which error is selected for reporting, which status takes precedence, and diagnostic fields. Do not translate an error code twice.
- Preserve per-item failure handling in batch commands. Moving item parsing or validation before the per-item error handler must not turn an indexed write error into a command-wide failure. Test a mixed valid/invalid batch, item indexes, and affected ordered/unordered and worker behavior.
- Establish how the caller discovers failure before treating a normally returning function as a success or a swallowed exception. Some operations raise an error immediately; others record failure in job history that the caller can query. Verify that the status/history code exists and works as documented. Do not add another warning or rethrow merely because the function returns normally. A plan to add history later does not make a failure observable now.
- Keep deliberately vague authentication errors vague when needed to avoid disclosing account existence, password expiry, or other sensitive authentication state. Preserve safe internal diagnostics without exposing credentials or backend implementation details.
- Do not expose internal cluster/schema versions in client-facing unsupported-feature errors. Explain the unsupported operation; keep necessary version details in safe internal diagnostics.
- Test changed error codes, response types/fields, how callers discover failure, and cleanup; a successful request or edited expected message alone is not enough.

## Calling APIs across extensions

- Reuse exported C APIs through their owning headers and declare the corresponding library dependency. Do not copy declarations or include another extension's implementation `.c` files. Preserve the API's transaction, memory-lifetime, schema-availability, and node-execution requirements.
- Use schema-qualified, parameterized SPI calls through existing query helpers when invoking a SQL API with its PostgreSQL permission checks. Verify the effective execution role and the SQL API's security contract; do not elevate privileges merely to make the call succeed.
- Direct C calls, `OidFunctionCall*()`, and `FunctionCallInvoke()` do not automatically enforce SQL `EXECUTE` privileges. When direct invocation crosses a SQL authorization boundary, require the applicable checks, such as `PgProcAclCheckCompat(..., GetUserId(), ACL_EXECUTE)`, plus any required schema/object or operation-specific authorization. A proven check in a common caller or callee is sufficient; do not require SQL ACL checks for every trusted internal C helper. Cover denied access when adding or changing such a boundary.

## Mutation Design and Validation

- Run input and feature-support checks before taking locks or doing writes/DDL when those checks do not depend on changing database state. Preserve permission checks and which error takes precedence. Take the required lock before reading state used to enforce limits or compute an update, or recheck that state under the lock. Rolling back unnecessary DDL does not avoid the work already sent to other nodes.
- Use a hook when another extension supplies or replaces the operation; call the C helper directly for work owned by the same implementation. If a hook is required to apply a change, report an error when it is missing rather than silently succeeding. For early support checks, use an ordinary function instead of adding a second hook whose only purpose is to fail.
- Reuse existing hash/StringView helpers instead of adding equivalent wrappers. Reuse values from `UPDATE ... RETURNING` rather than reading a row only to discard it or fetch the same values again. A copied StringView still points to the original string buffer: verify that the buffer remains valid for every user of the view before avoiding a deep copy.
- Avoid typed-object to BSON to typed-object round trips when the next operation can use the existing typed object. Build the final serialized value once where possible. Where ownership allows, copy aggregate/set values when they are actually accepted rather than every candidate, while ensuring borrowed buffers remain valid until then.
- Preserve a parsed string's pointer and explicit byte length together through validation, hashing, comparison, and dispatch. Reuse a matching StringView helper when one exists; otherwise construct the view from the parser-provided pointer/length instead of recomputing C-string length or demanding a nonexistent helper.
- Keep SQL and its parameter values together where the query executes. Prefer helpers that perform an operation over SQL-fragment macros that silently depend on the caller supplying `$3` or `$4`.
- For role and ACL changes, check partial revokes, inherited grants, missing enforcement hooks, membership semantics, and the actual execution identity; apply the [contribution review workflow](skills/pr-review/SKILL.md) without treating its invocation as permission to fix or publish.

## Cross-Node Object Identity

- Never use a PostgreSQL user/role, table/relation, or function OID as an identity passed between nodes. Send the object name and resolve the receiving node's local OID through the existing catalog/role helpers. Use schema-qualified table names and schema-qualified function names with argument-type names/signatures so overloads resolve correctly without depending on `search_path`.
- Apply this rule to cursor continuations and GetMore/resume state, coordinator/worker requests and responses, node/distribution function arguments and results, and execution-user propagation. Inspect the actual serialized payload, including nested BSON/JSON or plan/state fields; converting an OID to text is not name-based identification.
- Reference tables must not contain PostgreSQL catalog OIDs. This includes `oid` columns, OID-backed types such as `regclass`, `regrole`, and `regprocedure`, and OIDs stored as integers, text, arrays, or nested BSON/JSON fields. Replication copies the stored value without translating it to the destination's OID. Persist object names/signatures instead and resolve local OIDs on the node reading the row.
- Preserve the existing public/backing-role mapping and authorization checks when resolving user names. Missing or ambiguous transmitted names must fail explicitly, not fall back to the sender's OID or another execution role. Use resolved OIDs only for local execution, subject to normal locking/invalidation requirements. Documented logical collection/shard identifiers are not PostgreSQL catalog OIDs and are not prohibited by this rule.

## Bug Fix Workflow

After final C/header edits, use the component formatter documented in [PostgreSQL testing](postgres.TESTING.md#additional-checks-and-cross-component-coverage) in an approved environment and inspect its diff. Keep formatting within the task's changes; a formatting edit invalidates earlier build/test evidence and does not authorize staging. Do not substitute generic root targets or unavailable private workflow skills.

1. **Reproduce first**: Add a minimal reproducing SQL case, preferably to the existing owning regression test. Use approved disposable resources for interactive reproduction when needed; do not reset or take over an unrelated server on port 9712.
2. **Write SQL, not the baseline**: Extend the owning SQL test or add/register one in its schedule. Prefer the base OSS suite when it exercises the bug; select separately authorized consumer coverage when its behavior is required. Do not hand-author `.out` files for new tests or added cases.
3. **Validate the test fails without the fix**: Reproduce at the unfixed revision when safely feasible within the assigned checkout/resource permissions; a separate worktree requires authorization. Never stash or revert the user's changes. Follow [pr-test](skills/pr-test/SKILL.md) and the component targets in [PostgreSQL testing](postgres.TESTING.md), serialize shared installations, and rebuild final fixed sources before their validation. A missing expected file or a mismatch alone does not prove the bug: identify the wrong behavior/crash in the actual output. Report when an unfixed reproduction is unavailable.
4. **Validate and promote the fixed output**: Build and execute the focused SQL test, inspect its actual output against the intended behavior, then promote `results/<test>.out` to `expected/<test>.out`. Review the resulting diff and rerun the focused test. Do not bless an error or unintended behavior simply to make the baseline comparison pass.
5. **Verify style**: Run `citus_indent --check`; if corrections are needed, format first and repeat the affected build/tests against the final sources.
6. **Update the changelog**: Maintain one entry per feature or logical behavior change in [CHANGELOG.md](../CHANGELOG.md) under the current unreleased version, using the appropriate tag and issue number when available. Amend that entry as the feature evolves; do not add separate entries for each commit or follow-up build, test, or implementation fix.

## Building & Testing

- Use [PostgreSQL testing](postgres.TESTING.md) for actual OSS build/test commands, PostgreSQL version selection, prerequisites, artifacts and side effects. Installation, version switches involving services, and server restarts require explicit authorization.
- For new tests, apply the [shared behavioral-coverage rules](TESTING.md#selection-and-execution) and [pr-test workflow](skills/pr-test/SKILL.md).
- Generate expected output by executing the owning test and reviewing its behavior before promotion, for both new tests and added cases. Follow the [run-inspect-promote-rerun workflow](postgres.TESTING.md#changing-regression-expectations).
- Use an approved isolated local stack when interactive investigation is needed; a focused regression test can be the reproducer.
- Always check for PG version-specific variants (e.g. `_pg16`, `_pg17`) of expected output files that may also need updating.

## Test Construction Tips

- For a test specifically targeting a secondary-index or pushdown path, the owning suite's `documentdb_test_helpers.drop_primary_key` can remove the competing primary-key path on a disposable collection. This helper removes catalog metadata and the constraint, not just a planner preference. Do not use it as routine setup or for live data; keep a normal primary-key-present case when making performance claims. Check separately owned consumer helpers before using them.
- Prefer adding test cases to existing test files that already cover the feature area (e.g., append shard_collection crash tests to `commands_shard_collection.sql`) rather than creating new standalone files.
- For normal deterministic SQL test setup, set `documentdb.next_collection_id` and `documentdb.next_collection_index_id` to the same unique high value (e.g. `25701000`) to avoid collisions and ordering-sensitive expectations. Tests intentionally exercising ID collisions may override this; do not rewrite those cases to hide their trigger. The [ID candidate helper](../scripts/find_next_collection_id.sh) scans a test folder's collection-ID assignments, not index-ID uniqueness; inspect both before choosing the value.
- After test runs, inspect the owning suite's actual `log/summary*.out`, `.exitcode`, results and diffs as documented in [PostgreSQL testing](postgres.TESTING.md#side-effects-and-evidence), rather than assuming one summary filename or scanning only console output.
- If a test output has `No function matches the given name and argument types` error, this typically is a test authoring error and should be revisited.

## Coding Guidelines

- State what each feature counter counts and whether failed attempts should count. Place increments at the actual subfeature-use boundary and check all execution paths; entering a command does not mean that every optional subfeature was requested.

## Comment Guidelines
- Don't write overly verbose comments.
- Add comments only when they explain ideas that are not obvious from code, naming, or structure.
- Comments should explain **why**, not **what** (document intent and tradeoffs behind decisions).
- As you build or correct your understanding of an area, update the relevant file, function, or block doc comments with verified design intent, flow, and non-obvious constraints. Keep that context beside the code it explains and revise it when the implementation changes.
- When a developer challenges an approach and the discussion establishes a correction or design choice, document that choice in the affected code's doc comments: explain the chosen behavior, why the rejected approach was incorrect or unnecessarily complex, and the conditions that make the choice valid. Preserve the technical lesson, not a conversation transcript or personal attribution.
- Keep area-specific decisions in those source comments; put reusable backend architectural rules in these instructions rather than duplicating detailed guidance across both.
- Document constraints and invariants that are not enforced by type system.
- Call out **surprising or counter-intuitive** behavior.
- Preserve historical context or backward-compatibility requirements
- For planner, dispatch, and background-job comments, explain execution ownership, frequency, state lifetime, and cleanup where those are not visible from the local statements, including settings that must survive internal commits and session teardown versus explicit restoration.

## Distribution-Specific Knowledge Leaking Into Core Extensions

The OSS core extensions (`pg_documentdb_core/`, `pg_documentdb/`, `pg_documentdb_extended_rum/`) must stay distribution-agnostic - no dependency on Citus or any other multi-node plugin. Provider-specific execution belongs in the owning provider, not public OSS code. When core needs to consult provider-specific behavior, use the existing [API hooks](../pg_documentdb/include/api_hooks.h) pattern: a distribution-neutral contract with a nullable hook registered by the provider.

Flag as an **error** any change under the core layers that introduces:
- References to Citus symbols (`worker_partial_agg`, `worker_binary_partial_agg`, `citus_*`, `pg_dist_*`, `master_*`, `read_intermediate_result`) via string compare, oid lookup, or `get_func_name`/`get_rel_name`.
- Calls to Citus UDFs (`citus_is_coordinator`, `create_distributed_table`, etc.) - must go behind a hook.
- Coordinator-vs-worker branching using anything other than an existing hook (e.g. `IsMetadataCoordinator()`).
- Includes from `citus/` or internal distribution-provider headers.

Before flagging:
1. Confirm the file is in a public core layer; do not apply this restriction as if the provider's own implementation were core.
2. Check `pg_documentdb/include/api_hooks_def.h` and `pg_documentdb/src/api_hooks.c` for a contract that fits; the PR should use it.
3. If a new hook is needed, its name and call-site comment must describe the *contract* in distribution-neutral terms, not the Citus mechanism (e.g. `GetEffectiveAggregateFunctionOid_HookType`, not `TryGetWorkerPartialAggInnerFunctionOid_HookType`).
