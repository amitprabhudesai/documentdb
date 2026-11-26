---
name: build-documentdb
description: Select and run DocumentDB component builds using existing PostgreSQL and Rust toolchains. Use for changed code, SQL, headers, build inputs, or stale artifacts; distinguish incremental, clean, SQL-clean, debug, gateway, and separate Pgrx-host builds and separately authorized installation.
---

# Build DocumentDB

## Ownership and permissions

This skill owns build scope, depth, and artifact freshness. [pr-test](../pr-test/SKILL.md) owns test selection; [run-in-devcontainer](../run-in-devcontainer/SKILL.md) owns execution location and transport. Return evidence to the requesting workflow rather than invoking it recursively. A build-only request does not authorize tests.

Read trusted [CONTRIBUTING.md](../../../CONTRIBUTING.md), [operating policy](../../instructions.md), and [shared testing](../../TESTING.md). Establish approved baseline guidance before using contribution-modified instructions or tasks. Invocation does not authorize installation, elevation, dependency fetching, database administration, service changes, or Git publication.

Use an approved disposable Linux build environment without production credentials or sensitive host mounts. Do not install missing build tools on the host or repair container provisioning automatically. Confirm the execution location and selected toolchain first; report missing prerequisites as blocked. Dependency restoration requires separate authorization and a manifest change or an actual missing-dependency failure, not a speculative install.

## Select the owning surface

| Changed surface | References and build scope |
| --- | --- |
| BSON primitives/types | [Backend selection](../../documentdb.instructions.md#instruction-selection), [PostgreSQL instructions](../../postgres.instructions.md), and [PostgreSQL build/test details](../../postgres.TESTING.md); core plus affected consumers. |
| Document APIs, SQL, planner, or shared backend headers | The same backend references; main extension plus changed dependencies and consumers. |
| Extended RUM/access method | Backend references; extended RUM, its core modules, and affected main-extension consumers. |
| Gateway workspace | [Gateway instructions](../../gateway.instructions.md) and [gateway build/test details](../../gateway.TESTING.md); workspace and affected hosts. |
| PostgreSQL gateway host | Gateway references; separate Pgrx project and shared gateway dependencies. |
| Guidance/documentation only | No product build unless changed executable examples require an authorized probe. |

Inspect current Makefiles, manifests, scripts, generated inputs, and callers. Choose all affected components, not merely the directory containing the edit. The root generic `make`, `clean`, and `install` routes include a distributed component; they are not public-only shortcuts. Do not assume that component or its provider is available.

## Select PostgreSQL and build depth

Record the intended PostgreSQL major, exact `PG_CONFIG`, compiler/build flags, Rust toolchain where applicable, and source snapshot. Verify `pg_config --version` and `--pgxs` for that installation. Keep server binaries, libraries, extension directories, and Pgrx configuration consistent. Follow the transport skill's [version selection](../run-in-devcontainer/SKILL.md#postgresql-and-dependent-environment); changing an environment variable does not install or switch PostgreSQL.

The public backend build sequence, from the repository root, is:

```
make -C pg_documentdb_core
make -C pg_documentdb
make -C pg_documentdb_extended_rum
```

Pass the verified absolute `PG_CONFIG` as a Make command-line assignment to every relevant invocation when it is not the verified default. Narrow this sequence only when dependencies and affected consumers are accounted for.

| Build depth | Selection |
| --- | --- |
| Incremental | Default component build after a small C edit with unchanged PostgreSQL major, flags, headers/dependencies, and generated inputs. |
| Clean | Owning component `clean`, then rebuild; required after PostgreSQL-major or compiler/build-configuration changes, or evidence of stale objects. Rebuild dependent components when their inputs changed. |
| SQL-clean | Owning component `clean-sql`, then rebuild after SQL template, upgrade, or generation-input changes. These targets remove `.deps/` and `build/`; inspect their current recipes and preserve relevant evidence first. |
| Debug | Core/main builds accept `DEBUG=yes`. Clean before changing this flag and use it consistently for the selected build/install. Do not assume it configures extended RUM or Rust debugging. |

Do not clean unrelated work or use elevated cleanup to work around file ownership. Serialize clean builds across PostgreSQL versions when they share output directories. A clean build is not itself an installation or upgrade-path test.

## Installation is separate

Only install when explicitly authorized for the identified disposable PostgreSQL installation. From the repository root, `make install-documentdb` installs core, main, and extended RUM without the distributed route. Component `make -C <component> install` targets permit a justified narrower installation. Substitute an actual component and pass the same `PG_CONFIG` and applicable flags as the build.

Installation writes PostgreSQL extension directories and trims installed SQL; it does not update an existing database's extension version automatically. If privileges are insufficient, report the exact blocked target and follow host restrictions. Do not automatically add `sudo`, change ownership, restart a server, or run `ALTER EXTENSION`.

## Gateway and Pgrx builds

Run gateway tasks from `pg_documentdb_gw/`:

```
cargo make build
```

This task builds the workspace and regenerates `NOTICE`; inspect and preserve pre-existing edits before running it. Do not overwrite conflicting legal-artifact edits. `cargo make dev` combines build and lint when both are requested; lint is not test execution. Release tasks have distinct frozen-dependency/profile behavior documented in the [gateway strategy](../../gateway.TESTING.md#lint-formatting-builds-and-legal-artifacts).

Run host builds separately from `pg_documentdb_gw_host/`, with the intended supported `PGVERSION` supplied explicitly. For an available PG16 installation:

```
PGVERSION=16 cargo make build
```

Substitute the actual major only after checking its installation and Pgrx configuration. A gateway workspace build does not build the separate host. Select both for shared-core changes affecting both consumers; neither build establishes hosted-runtime lifecycle correctness.

## Finish and return evidence

Build after the final source edit. Record exact commands, working directories, source/diff identity, PostgreSQL/toolchain/configuration, exit status, logs, generated changes, and any authorized installation destination. Do not infer success from partial output or reuse evidence after relevant inputs changed.

Return to the requesting workflow with passed/failed/blocked/not-run distinctions. If only a build was requested, stop after the build report. Test coverage, human testing attestations, commits, and publication remain separate.
