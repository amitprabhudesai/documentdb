# Gateway testing strategy

Use the [shared selection/report contract](TESTING.md) and [component instructions](gateway.instructions.md). Commands describe existing behavior, not authorization to install tools, mutate databases, or start shared services.

## Projects and prerequisites

The [gateway workspace](../pg_documentdb_gw/Cargo.toml) contains `documentdb_gateway`, `documentdb_gateway_core`, `documentdb_gateway_otel`, and `documentdb_tests`. [The Pgrx host](../pg_documentdb_gw_host/Cargo.toml) is separate; workspace tests/lint do not cover it.

Use the pinned [Rust toolchain](../rust-toolchain.toml) and project Cargo configuration. Check available Cargo/cargo-make/nextest tooling, dependency/lockfile state, native dependencies, and the selected PostgreSQL environment. `--frozen` failures are blockers unless a dependency-fetch/install action is separately authorized.

Some core unit tests also create PostgreSQL connections; `--lib` does not guarantee database-free tests. Inspect the selected tests and their fixtures. Integration tests need a disposable running PostgreSQL instance with the required DocumentDB extensions, user/peer-auth setup, and privileges.

The integration harness uses the `mongodb` Rust driver, creates test users/roles, and drops/recreates test databases. It starts shared gateway code in the test process rather than requiring a separately running gateway binary. Default test setup uses gateway port `10260`, backend defaults including port `9712`, and a configurable `PostgresSystemUser`; inspect the actual configuration before running. Do not run tests against a production/shared database or alongside another listener on the test port.

Sources: [setup initialization](../pg_documentdb_gw/documentdb_tests/src/test_setup/initialize.rs), [configuration](../pg_documentdb_gw/documentdb_tests/src/test_setup/config.rs), [PostgreSQL setup](../pg_documentdb_gw/documentdb_tests/src/test_setup/postgres.rs), [clients/database setup](../pg_documentdb_gw/documentdb_tests/src/test_setup/clients.rs), and [in-process gateway](../pg_documentdb_gw/documentdb_tests/src/test_setup/gateway.rs).

## Targeted and workspace commands

Run these from `pg_documentdb_gw/` after preflight:

| Coverage | Existing command |
| --- | --- |
| Shared core unit tests, no doctests | `cargo test -p documentdb_gateway_core --lib -- --test-threads=1` |
| Core unit and doctests | `cargo test -p documentdb_gateway_core -- --test-threads=1` |
| Command integration test target | `cargo test -p documentdb_tests --test command_tests -- --test-threads=1` |
| Focused test by name | `cargo test -p documentdb_tests --test command_tests '<test-name>' -- --test-threads=1` |
| OTel unit tests | `cargo test -p documentdb_gateway_otel --lib -- --test-threads=1` |
| Existing workspace test wrapper | `cargo make test` |
| Formatting check | `cargo make format-check` |
| Gateway lint aggregation | `cargo make lint` |

Substitute an actual test name; verify selection counts and results. Choose other existing integration targets for pools, configuration, TLS, RBAC, cursors, sessions, transactions, killop, or errors as affected. Binary/CLI changes need their relevant binary tests, not only `documentdb_gateway_core`.

Keep these runs serialized when they share PostgreSQL, ports, process-global state, or test artifacts. The wrapper's single-thread settings are part of the existing harness contract, not an invitation to run several wrapper processes in parallel.

## Wrapper behavior and artifacts

[The cargo-make task](../pg_documentdb_gw/Makefile.toml) invokes [run_rust_tests_with_junit.sh](../scripts/run_rust_tests_with_junit.sh):

- With nextest, it runs workspace tests with `--no-fail-fast --test-threads=1 --profile ci` and copies `target/nextest/ci/junit.xml` to `target/test-results/junit.xml` if present. Nextest does not provide doctest coverage.
- Without nextest, it runs `cargo test --workspace -- --test-threads=1`, without JUnit output. This includes Cargo's applicable doctest execution.
- The script accepts extra arguments on its nextest path but does not forward them on the cargo-test fallback. Do not describe wrapper arguments as portable selectors; use explicit Cargo commands for the targeted examples above.

Record which path ran, its exit status, actual selections, and logs. Preserve previous reports and distinguish current artifacts from stale JUnit files. No JUnit file is not itself proof of failure; a report file is not proof of success.

## Lint, formatting, builds, and legal artifacts

Gateway lint includes formatting, workspace/all-target/all-feature Clippy with warnings denied and frozen dependencies, cargo-deny bans, and a generated NOTICE comparison. It is not just Clippy. [The gateway Makefile](../pg_documentdb_gw/Makefile.toml) defines the exact steps.

`cargo make build` regenerates NOTICE before building; `cargo make notice` intentionally updates it. These are working-tree writes and may need dependency/network prerequisites. Do not use a build as a supposedly read-only preflight or update NOTICE merely to conceal a lint failure.

`cargo make format` applies the project-specific formatter; bare `cargo fmt` does not express all enforced options. Formatting aliases are in [gateway Cargo configuration](../pg_documentdb_gw/.cargo/config.toml).

## PostgreSQL-hosted gateway

Run host commands from `pg_documentdb_gw_host/`, with Pgrx/PGXS and its selected PostgreSQL installation configured. For PG16, existing commands are:

```
PGVERSION=16 cargo make build
PGVERSION=16 cargo make lint
```

Substitute the intended supported major version explicitly. The host defaults to PG16 and exposes pg15/pg16/pg17/pg18 feature selections; [host tasks](../pg_documentdb_gw_host/Makefile.toml) and [Cargo configuration](../pg_documentdb_gw_host/.cargo/config.toml) are separate from the gateway workspace.

Build/lint are not host lifecycle tests. For worker registration, GUCs, preload, signals, startup failures, or shutdown changes, require an approved disposable hosted-runtime exercise of the affected behavior. The existing gateway CI builds/lints the host but runs integration tests through their in-process shared-core harness; do not describe that as complete Pgrx lifecycle coverage. If host execution is unavailable, report that coverage as blocked/not run.

## Compatibility and cross-component evidence

SQL/query-catalog/result-shape and error-code changes need [PostgreSQL coverage](postgres.TESTING.md) alongside gateway tests. Test malformed input, authorization/ownership boundaries, response framing, and cancellation/lifecycle behavior as relevant; do not apply every category mechanically to every patch.

Document the tested host topology, versions, and unsupported paths. Existing [gateway CI](../.github/workflows/gateway_tests.yml) covers a specific PG15-18/architecture matrix; one local run does not establish the entire matrix. New user-facing behavior additionally needs separately reported functional-test coverage.
