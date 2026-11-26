# PostgreSQL testing strategy

Use the [shared selection/report contract](TESTING.md) and [component instructions](postgres.instructions.md). Commands below describe checked-in targets; they are not an authorization to build, install, reset data, or execute tests in the current environment.

## Prerequisites and build scope

Use an approved Linux development environment with PostgreSQL/PGXS, extension dependencies, locale support, and required test tooling. Check `pg_config --version` and `pg_config --pgxs`; select the intended `PG_CONFIG` before building or testing and keep its server, binaries, libraries, and installed extensions consistent. The checked-in devcontainer selects PG16; use an explicitly requested version when applicable.

The regression Makefiles refer to PostgreSQL test binaries through PGXS variables; recovery/background-worker suites require a PostgreSQL build with TAP enabled and its Perl test infrastructure. A missing binary, extension, locale, or TAP configuration blocks that suite; do not silently skip it.

Root generic build/test targets fan out beyond the OSS components. For OSS-only component builds, invoke the existing component Makefiles in dependency order, from the repository root:

```
make -C pg_documentdb_core
make -C pg_documentdb
make -C pg_documentdb_extended_rum
```

Builds generate artifacts; installation modifies PostgreSQL extension directories. Installation additionally trims some installed SQL files. Inspect [root routing](../Makefile), [core](../pg_documentdb_core/Makefile), [document](../pg_documentdb/Makefile), and [RUM](../pg_documentdb_extended_rum/Makefile) definitions. Do not install into a shared PostgreSQL instance without authorization.

## Existing suites and selection

Run from the repository root after preflight. A full document `check` runs several suites; `check-regress` is narrower but includes its regression-directory isolation/output-validation targets.

| Coverage | Existing command |
| --- | --- |
| Core SQL regression | `make -C pg_documentdb_core check` |
| Core minimal plus named extra tests | `make -C pg_documentdb_core/src/test/regress check-bson-minimal EXTRA_TESTS='<test>'` |
| Document regression, isolation, output validation | `make -C pg_documentdb check-regress` |
| Document minimal plus named extra tests | `make -C pg_documentdb check-minimal EXTRA_TESTS='<test>'` |
| Document isolation | `make -C pg_documentdb check-isolation` |
| Full document test aggregation | `make -C pg_documentdb check` |
| Extended RUM regression/output validation | `make -C pg_documentdb check-extended-rum` |
| Extended RUM minimal plus named extra tests | `make -C pg_documentdb check-extended-rum-minimal EXTRA_TESTS='<test>'` |
| Extended RUM built-in resource-manager variant | `make -C pg_documentdb check-extended-rum-builtin-rmgr` |
| Document recovery TAP | `make -C pg_documentdb check-recovery` |
| Extended RUM recovery TAP | `make -C pg_documentdb check-extended-rum-recovery` |
| Background-worker TAP | `make -C pg_documentdb/src/test check-background-worker` |

`<test>` is an existing SQL test basename without `.sql`; substitute the intended test name. `EXTRA_TESTS` adds to the minimal schedule, not replaces it. Full schedules have version-dependent mutation; do not assume all versions run identical tests. Isolation tests use their own specification/schedules, not SQL expectation files.

For targeted TAP execution, `PROVE_TESTS` selects the suite-relative test path. For example:

```
make -C pg_documentdb/src/test/background_worker_tests PROVE_TESTS=t/001_production_jobs.pl
```

Inspect the relevant [regression](../pg_documentdb/src/test/regress/Makefile), [core](../pg_documentdb_core/src/test/regress/Makefile), [RUM](../pg_documentdb/src/test/extended_rum_tests/Makefile), [recovery](../pg_documentdb/src/test/recovery_tests/Makefile), [RUM recovery](../pg_documentdb/src/test/extended_rum_recovery_tests/Makefile), and [worker](../pg_documentdb/src/test/background_worker_tests/Makefile) definitions before changing selectors. Select the RUM mode affected by the contribution; one mode is not proof of all modes.

## Side effects and evidence

SQL regression suites create local temporary PostgreSQL instances and overwrite outputs under their test directories. Current source defaults use core port `58040`, document regression `58050`, document isolation `58051`, and extended RUM `58060`; configuration and suite behavior must be checked before execution. Do not run overlapping copies of a suite or assume a free forwarded development port covers these ports.

Results, `regression.diffs`, summaries, and `.exitcode` live under the suite directory. TAP Makefiles remove and recreate their `tmp_check/` and `log/` directories; preserve previous evidence first. TAP creates test clusters and may exercise recovery/replication and worker lifecycles.

`installcheck`, valgrind, and server-helper targets have different existing-server/startup/reset semantics. Do not substitute them for temporary-instance regression runs without inspecting the target and obtaining authorization for its exact resources. In particular, the server helper's cleanup option is destructive and its default data directory is not automatically a task-owned resource.

Planner/index changes should establish result correctness with and without the affected index/path, relevant isolation/recovery behavior, and plan/performance evidence for performance claims. SQL declarations/upgrades require validation of the affected install/upgrade path; a new-cluster result alone is insufficient.

## Changing regression expectations

Never hand-author a pg_regress expected output file to predict a successful result.

1. Add or change the SQL input in the relevant existing suite and include it in an applicable schedule/selection when necessary.
2. Run it in the approved environment. Preserve the generated `results/<test>.out`, diffs, command, version, and exit status.
3. Read the generated output and verify the intended semantics, including errors and ordering. A failure caused by missing prerequisites is not promotable expected behavior.
4. Only after that review, copy the exact generated result to the suite's `expected/<test>.out`; do not normalize unexpected behavior away.
5. Rerun the same test selection, verify it passes, and run required broader/output-validation coverage. Review the final diff for accidental expectation changes.

If the environment cannot run the SQL, leave expectation generation blocked and report it; do not fabricate output. Isolation/TAP tests need their own assertion/expectation discipline.

## Additional checks and cross-component coverage

Use the existing formatter and file-state checks when applicable in an approved environment:

```
citus_indent --check
./scripts/validate_extension_file_state.sh
```

The file-state script's platform/tool prerequisites must be checked; do not claim it ran based on a static reading. `make -C pg_documentdb citus-indent` applies formatting and is a write operation. Preserve copyright/license conventions using [the tooling guide](../scripts/README-copyright-checks.md).

Error mappings, SQL return shapes, authentication/role semantics, cursor continuation, and transaction behavior can require [gateway integration coverage](gateway.TESTING.md) even for C/SQL-only diffs. New user-facing features also require functional coverage in the separate functional-tests repository; follow the shared policy for authorization and reporting of external work.
