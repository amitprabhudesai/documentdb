# Gateway component instructions

Apply these instructions to `pg_documentdb_gw/` and `pg_documentdb_gw_host/`, and to changes elsewhere that affect gateway startup, configuration, protocol/SQL contracts, or deployment. First read the [shared policy](instructions.md); use current implementation and call-site evidence and the [gateway testing strategy](gateway.TESTING.md).

## Hosts and dependency boundaries

The gateway workspace contains the standalone binary, shared core, OpenTelemetry adapter, and integration tests. The PostgreSQL Pgrx host is a separate Cargo project depending on the shared core; a workspace-only command does not validate it.

The active `run_gateway` entry point delegates to runtime v2; `run_legacy_gateway` remains separate. Do not assume older `service/connection_loop` transport code is the active listener. Trace call sites before changing behavior or claiming host parity.

Keep protocol/command semantics and host-independent behavior in the shared core. Keep standalone process setup and exporter ownership in their existing layers. Avoid importing PostgreSQL/Pgrx host assumptions into general gateway code.

Sources: [workspace manifest](../pg_documentdb_gw/Cargo.toml), [core entry points](../pg_documentdb_gw/documentdb_gateway_core/src/lib.rs), [standalone startup](../pg_documentdb_gw/documentdb_gateway/src/main.rs), and [host startup](../pg_documentdb_gw_host/src/bgworker.rs).

## Protocol, authentication, and SQL boundaries

Read the applicable shared [collation](documentdb-common.instructions.md#collation-coverage) and [response/error contracts](documentdb-common.instructions.md#error-contracts-and-failure-visibility), including tests of those behaviors. These section references do not load unrelated backend implementation requirements.

Preserve framing and size bounds, request/response IDs, BSON ownership, and required-response versus one-way behavior. Reject invalid input explicitly; do not manufacture a success response when decoding or execution fails.

Keep authentication transitions and authorized/unauthorized routing consistent with the current router. Authorization includes resource ownership and backend privileges; authenticating a connection alone does not establish permission for every operation.

Trace command processing through the query catalog, data client, PostgreSQL declaration, and response/error conversion. Preserve parameterization, expected column names/types, BSON shape, command status, error codes, and transaction-sensitive function/procedure selection. Any interface change requires cross-component coverage.

Sources: [runtime handler](../pg_documentdb_gw/documentdb_gateway_core/src/runtime/v2/handler.rs), [request router](../pg_documentdb_gw/documentdb_gateway_core/src/service/connection_loop/routing.rs), [query catalog](../pg_documentdb_gw/documentdb_gateway_core/src/postgres/query_catalog.rs), [data client](../pg_documentdb_gw/documentdb_gateway_core/src/postgres/documentdb_data_client.rs), and [errors](../pg_documentdb_gw/documentdb_gateway_core/src/error.rs).

## Async state and resource ownership

Identify per-connection, per-request, per-principal, session, cursor, and transaction state separately. Preserve ownership checks across continuation, invalidation, expiry, logout, and cancellation. Do not equate a cursor lookup with a harmless read; the store has removal/ownership-transfer operations.

Do not hold blocking locks or map guards across an await without understanding the specific API's guarantees. Avoid blocking work on async workers, unbounded buffering, and unsupervised tasks. Establish how spawned tasks terminate and how connection permits, pinned PostgreSQL connections, and transaction state are released.

Handle shutdown and request cancellation as distinct lifecycle events. Preserve the difference between expected shutdown during startup retry and genuine failure. Do not add a retry without checking idempotency and the possibility of an already-applied write.

Sources: [listener supervision](../pg_documentdb_gw/documentdb_gateway_core/src/runtime/v2/listener.rs), [pool manager](../pg_documentdb_gw/documentdb_gateway_core/src/postgres/conn_mgmt/pool_manager.rs), [cursor store](../pg_documentdb_gw/documentdb_gateway_core/src/context/cursor/cursor_store.rs), [session manager](../pg_documentdb_gw/documentdb_gateway_core/src/context/session/session_manager.rs), and [transaction state](../pg_documentdb_gw/documentdb_gateway_core/src/context/transaction).

## Configuration, hosting, and telemetry

Preserve configuration precedence, validation, listener exposure, TLS, and authentication contracts. A permissive development setting is not a production default to recommend. Treat environment overlays and packaged startup behavior as interfaces requiring tests.

For the Pgrx host, inspect `_PG_init`, background-worker registration, signal/latch handling, configured database, Tokio runtime, and teardown. Preserve PostgreSQL API thread restrictions; SPI access in the host does not mean arbitrary async workers may call PostgreSQL backend APIs safely.

Distinguish shared telemetry hooks from the standalone OpenTelemetry exporter and documentdb-local usage telemetry. Do not add credentials, raw sensitive documents, or unbounded/high-cardinality identifiers to logs or metrics. Preserve provider initialization and shutdown ownership.

Sources: [configuration](../pg_documentdb_gw/documentdb_gateway_core/src/configuration/setup.rs), [host registration](../pg_documentdb_gw_host/src/lib.rs), [host worker](../pg_documentdb_gw_host/src/bgworker.rs), [OTel adapter](../pg_documentdb_gw/documentdb_gateway_otel/src/lib.rs), and [local-image privacy disclosure](../documentdb-local/PRIVACY.md).

## Rust style and dependency changes

Follow the [Rust API Guidelines](https://rust-lang.github.io/api-guidelines/naming.html), [Rust API checklist](https://rust-lang.github.io/api-guidelines/checklist.html), [Microsoft Rust checklist](https://microsoft.github.io/rust-guidelines/guidelines/checklist/index.html), and workspace/host lint configuration. Use `crate::` imports within a crate and the existing typed error/result helpers. Do not introduce broad panic/unwrap exceptions, silent catches, or unnecessary type-erasing casts.

Use `cargo make format` and `cargo make format-check` in the relevant project, not bare rustfmt defaults. The [gateway](../pg_documentdb_gw/.cargo/config.toml) and [host](../pg_documentdb_gw_host/.cargo/config.toml) aliases enforce import and code-document formatting.

Inspect manifests, lockfiles, pinned/patched dependencies, licensing, and the required NOTICE update for dependency changes. Do not update dependency versions or generated legal material opportunistically. Gateway build and lint have different NOTICE effects; see [testing](gateway.TESTING.md).

Before handoff, select both shared-core and host-specific coverage where applicable. A core integration test is not proof of standalone CLI behavior or PostgreSQL-hosted lifecycle correctness.
