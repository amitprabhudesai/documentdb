---
description: 'Select DocumentDB backend instruction layers and main-extension behavior.'
applyTo: 'pg_documentdb/**,pg_documentdb_core/**,pg_documentdb_extended_rum/**'
---

# DocumentDB backend selection and main extension

This is the backend entry point and the owner of main-extension behavior under `pg_documentdb/`, including affected callers and overrides outside that directory. Read the selected files; links alone do not load their requirements. Reuse guidance already loaded and select by assigned behavior and affected contracts, not unrelated paths in a review.

## Instruction Selection

For backend C, SQL, and tests, first read the trusted [operating policy](instructions.md), [PostgreSQL safety](postgres.instructions.md), [shared backend contracts](documentdb-common.instructions.md), and [PostgreSQL testing strategy](postgres.TESTING.md).

Apply the [repository authoring conventions](instructions.md#authoring-conventions), including computed-result reuse and existing-code style, during construction as well as review.

| Component or affected behavior | Read |
| --- | --- |
| `pg_documentdb_core/`, BSON representations or primitives | [Core entry point](documentdb-core.instructions.md) |
| `pg_documentdb/`, collection, mutation, planner, cursor or feature-flag behavior | The main-extension sections below |
| `pg_documentdb_extended_rum/`, or its SQL API boundary | [Extended RUM](documentdb-rum.instructions.md) |
| SQL-to-wire results, errors, authentication, cursors or transactions | [Gateway instructions](gateway.instructions.md) and [gateway testing](gateway.TESTING.md) |

Canonical references live in `.agents/`; Copilot path discovery is wired through `.github/instructions/`. Frontmatter in these references describes scope but is not a substitute for those adapters or explicit loading.

Follow affected consumers for SQL/catalog dependencies, signatures, serialized formats, upgrade behavior, routing, pushdown, remote payloads and aggregate-state serialization, even when only one component's files change. Shared name-based identity and distribution-agnostic boundary rules still apply to local OSS work. This does not authorize expanding the assignment or executing build/test workflows.

Private-service deployment overlays are not part of this OSS policy. The separately imported [distribution guidance](documentdb-distributed.instructions.md) contains service-specific requirements and unavailable references; do not treat it as verified OSS requirements or load it automatically for local backend work. For an explicitly assigned distributed change, reconcile those requirements against its owning implementation and trusted policy before using them.

The following sections apply to main-extension behavior, not automatically to every core or RUM change.

## Implementation Style

- Keep BSON parsing, validation, and per-entity control flow in C. Parse each entity into a struct, then execute a small parameterized query for that entity. Prefer an explicit C iterator and per-entity writes over a large SQL statement that expands a BSON or JSON payload and implements the operation itself.

## Mutation Design and Validation

- Reuse or extract the complete existing index-creation flow when adding another entry point. Keep catalog registration, physical index creation, and required statistics setup together rather than duplicating a subset of those steps. Preserve collection-specific statistics settings and failure handling.
- When sharding or repartitioning requires statistics objects to be rebuilt, preserve the collection's stored statistics settings unless those settings also need to change. Check locks acquired by called helpers, including lock upgrades, and verify that the configured statistics still work afterward.

## Effective Options and Collection Metadata

- Before reusing a collection metadata object, verify that its loader populated the fields the next operation needs, including shard keys and default collation. A lookup by collection ID can return a different set of fields from a full catalog lookup. Reuse complete existing metadata or populate command-owned state; do not add fields to a long-lived cache merely to eliminate a lookup.
- If an insert or upsert can create a collection, resolve inherited options from the final collection metadata after creation or a concurrent creator has won, under the required lock. Do this before deriving shard routing and building writes that depend on those options. Add a deterministic concurrency test for that interleaving; do not assume every delete path has the same creation race.
- Preserve each command's documented distinction between an absent option, null, an empty document, and an explicit default such as simple collation. Keep "needs resolution" separate from "resolved to the default": an empty parsed string alone does not distinguish them.
- Carry resolved options through both find and aggregate continuations and the affected worker paths. Do not resolve them again on `GetMore` or copy unresolved state into a continuation. Apply the [command-representation contract](#original-and-parsed-command-representations) and shared [ownership rules](documentdb-common.instructions.md#computed-results-and-cache-ownership) before removing an original or parsed representation.

### Original and parsed command representations

Original input and parsed/effective values are not automatically duplicate state. A worker command may need the original BSON while another path consumes a parsed value. Check the consumers and where parsing errors must be reported before removing a representation or moving parsing earlier.

## Maintenance Commands and Diagnostics

- For each maintenance-command mode, establish whether it may block writes and how long it may wait for locks. Choose an engine operation with that behavior rather than relying on a similar command name.
- For long-running commands that must support `currentOp` and `killOp`, check the API/schema names and worker execution used by clients. Verify that `currentOp` shows the operation and `killOp` stops the work, not merely that the cancellation request succeeds. Check the command label separately: an empty label does not mean the operation is absent.

## Planner Cost and Selectivity Gating (Mandatory)

- **Always gate new or changed planner cost and selectivity behavior through the existing collection-aware planner gate.** This includes full scans, index scans, row/cardinality estimates, startup/per-tuple costs, and cost-based path selection, even when the calculation does not directly read statistics. The current gate also permits the established composite-index planner override; preserve that explicit behavior rather than claiming it requires collection statistics in every mode.
- Reuse the existing collection-statistics-aware helpers: `EnablePlannerCostSelectivity(planner, args)` or `EnablePlannerCostSelectivityFromRelOptInfo(planner, rel)`, as appropriate.
- A new feature GUC alone is not sufficient. Keep any feature-specific switch as an additional condition, and preserve the previous estimates and costs when the planner gate is false. Check every affected cost and selectivity entry point; gating one does not protect the others.
- Include focused regression coverage with collection statistics enabled and disabled, with the composite-index planner off so it cannot mask a missing collection gate. Also cover any retained override and feature switch, checking both estimates and costs when both change.
- During review, treat a reachable ungated cost/selectivity change or missing disabled-statistics coverage as an **error**. Verify any claimed upstream gate protects all affected paths before reporting a finding.

## PostgreSQL Selectivity Memory Boundaries

Run PostgreSQL selectivity and expression-statistics estimation in short-lived scratch contexts. Reuse `ExecSelectivityFunc4`, `ExecSelectivityFunc5`, and `ExecGenericRestrictionSelectivity` where available; otherwise wrap the native call in a temporary context. Restore the caller before resetting/deleting scratch, preserve borrowed inputs and per-iteration state, and delete owned contexts on every normal return. An existing enclosing temporary wrapper is sufficient when it covers the complete estimation work.

For selectivity or statistics-estimation changes, inspect every affected native call and its enclosing context. A wrapper at one entry point does not protect sibling paths. Report a reachable allocation-lifetime defect with its concrete failure scenario; do not infer coverage from a helper's existence.

## Building & Testing

- New SQL regression tests should be added to `pg_documentdb/src/test/regress/basic_schedule` in an appropriate parallel test group. Follow the [expectation workflow](postgres.TESTING.md#changing-regression-expectations); this does not authorize a test run or environment mutation.

## Test Construction Tips

- The `shard_collection` API is available in the non-distributed OSS extension (`documentdb_api.shard_collection`), so sharded-collection tests can run in the base OSS test suite without requiring `documentdb_distributed`.
- If `make -C pg_documentdb check-minimal` produces unrelated ordering-dependent collection-ID diffs, inspect them and use the owning full regression selection, `make -C pg_documentdb check-regress`, before promoting output. A full document aggregation is `make -C pg_documentdb check`; do not substitute unavailable private-repository targets or treat a mismatch as permission to bless output.
- Do not set `citus.next_shard_id` in tests under `pg_documentdb`; that GUC is only relevant for a separately tested distribution provider.

## Coding Guidelines

- Add new shared continuation BSON field names as PascalCase `StringView` members of `ContinuationFieldNames` in `pg_documentdb/include/commands/cursor_common.h`, with matching entries in the immutable `ContinuationFields` initializer in `pg_documentdb/src/customscan/custom_scan_continuation.c`. Keep both lists alphabetically ordered by member name. Compute each length with `sizeof("literal") - 1`, and use the member's `.string` and `.length` in serializers and parsers instead of repeating literals. Preserve existing serialized key values; do not conflate continuation keys with SQL column names, ordinary response fields, or private fields with different meanings.
- Code that updates explain logic must add any new explain output fields, with descriptions, to [the explain output schema](../pg_documentdb_gw/documentdb_gateway_core/src/explain/documentdb_explain_output.schema.json). Do **not** change the explain output schema version (`explainVersion`) as an incidental field addition; a version change requires an explicit compatibility decision.
- GUCs added in `pg_documentdb/src/configs/feature_flag_configs.c` MUST have a comment indicating the status of the feature flag. For new default-on flags use "Added on vX.XXX, enabled on vX.XXX, remove after vX.XXX"; for new default-off flags use "Added on vX.XXX, pending stabilization". Use the current OSS DocumentDB version convention, including its major version. Default-off flags should not have a "Remove after" comment.
  - Inspect existing comments and registration directly. The imported feature-status generator and generated Markdown are absent in this checkout; do not claim their validation ran or invent a replacement tool. If that tooling is supplied later, keep its status document synchronized and require its check to pass.
  - Error if a GUC is declared but not registered.
  - Error if a feature flag's retirement target is two releases behind the version in `pg_documentdb/documentdb.control`; identify the intended release sequence rather than subtracting version strings.
  - Error if the retire date of a feature flag is less than 1 release after its enablement.
  - If a generated status document is available, default-off flags must have its warning and pending-stabilization labels.
  - Error if any feature has an "Unknown stabilization time".
  - Error if any feature flag does not conform to the comment format described above.
  - Error if any feature flag is named "Disable" - feature flags should always "enable" features (even when they are default on).
