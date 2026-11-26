---
name: run-in-devcontainer
description: Transport DocumentDB tooling into the matching running VS Code devcontainer, or execute directly in an already verified development environment. Check direct execution before container discovery; verify checkout, workspace, user, PostgreSQL version, and attached command results without changing provisioning or services.
---

# Run with devcontainer tooling

## Ownership and permissions

This skill owns where and how an already selected command runs, not which build or suite is appropriate. Use [build-documentdb](../build-documentdb/SKILL.md) for builds and [pr-test](../pr-test/SKILL.md) with its [component strategies](../../TESTING.md) for testing. Return transport/results to the requester without recursively invoking it.

Read trusted [operating policy](../../instructions.md) and the selected component guidance before executing contribution-controlled code. A devcontainer is not a sandbox or permission grant. Require approved disposable resources without production credentials; preserve restrictions on installation, elevation, database resets, listeners, services, and Git mutations.

## Check direct execution first

Before any Docker, Podman, or devcontainer CLI call, resolve the current Git root and compare it with the assigned checkout. Use the execution host/client's established environment metadata or an already verified transport context to determine whether this process is in that checkout's development container. An explicitly approved non-container Linux build environment can also use direct execution.

The presence of `.devcontainer/devcontainer.json`, Linux, `PG_VERSION`, or `/.dockerenv` alone does not identify this checkout's devcontainer. The [local configuration](../../../.devcontainer/devcontainer.json) and [Dockerfile](../../../.devcontainer/Dockerfile) do not set a workspace-root environment marker. Do not invent one or infer the workspace from the Dockerfile's `/home/documentdb` working directory.

If the current environment is verified for this checkout, run directly and do not discover containers, inspect sockets/processes/cgroups, or wrap commands in `docker exec`. Cache that decision for the task, invalidating it if the checkout or environment changes. Verification does not make an unapproved environment safe.

If already inside an unidentified container, report unresolved execution context rather than attempting nested engine discovery. For a known host checkout, proceed to host transport. Missing context is a blocker, not permission to pick another machine.

## Host transport

Keep source edits, searches, diffs, and ordinary Git operations in the assigned host checkout. Only environment-dependent tooling needs transport. Use the existing bind mount; do not copy the repository into a container.

### Find exactly one running container

Resolve the absolute host repository/configuration paths, accounting for symlinks using tools available on the host. Use the configured container engine; do not probe alternative engines after a connection failure.

For Docker, substitute the actual configuration path in this command template:

```
docker ps --filter 'label=devcontainer.config_file=<absolute-config-path>' --format '{{.ID}}'
```

If there is no exact-label match, enumerate running containers carrying the `devcontainer.config_file` label and inspect only their configuration labels and bind-mount source/destination pairs. Match the canonical repository bind source, or a verified ancestor bind containing that checkout; do not select by generated container name, image, or matching commit alone. Do not dump the complete inspect JSON, environment, or credentials.

Require exactly one candidate. With none, report that the matching container is unavailable and request that the user start it. With multiple, report the relevant non-secret labels/mounts and obtain a selection; never choose the first arbitrarily. Do not start, rebuild, or reconfigure a container automatically. These procedures use ordinary CLI output and do not require host `mapfile`, GNU `realpath`, or a new discovery helper.

### Verify the shared workspace

Derive the container workspace from the verified bind destination plus the checkout's relative path within its host bind source. It is not necessarily `/workspaces/documentdb`. Verify the mapping is writable and shared, and that the container's default configured user is non-root before executing tooling.

Compare host and container Git root (under the path mapping), branch, HEAD, and full status including untracked files. A matching HEAD alone cannot establish that dirty sources are shared. Use direct executable arguments, for example:

```
docker exec --workdir '<container-workspace>' '<container-id>' git rev-parse --show-toplevel
docker exec --workdir '<container-workspace>' '<container-id>' git rev-parse HEAD
docker exec --workdir '<container-workspace>' '<container-id>' git branch --show-current
docker exec --workdir '<container-workspace>' '<container-id>' git status --short --untracked-files=all
```

Replace placeholders with verified values; never execute them literally. A mismatch blocks execution until resolved. Do not switch branches, reset, copy files, or weaken Git ownership checks to make the comparison pass.

Reuse the resolved container/workspace within the task. Revalidate after a restart/rebuild, checkout change, or failed transport; never persist an ID as a permanent selection.

### Run attached commands

Use the default non-root user, explicit working directory, and executable arguments:

```
docker exec --workdir '<container-working-directory>' '<container-id>' <executable> <arguments>
```

Working directory may be a verified component subdirectory. Keep commands attached and noninteractive so output and exit status reach the requesting workflow. Do not use `-it`, interactive shell profiles, privileged execution, or detached commands for bounded builds/tests.

Use existing checked-in scripts for compound workflows where available; inspect them first. Do not present generated shell scripts as substitutes for repository tooling. Preserve dependent ordering and propagate failures. Do not commit or push through host transport; ordinary Git authorization still applies in direct mode.

## PostgreSQL and dependent environment

The image provisions the PostgreSQL major selected by its build argument (currently PG16 by default); it does not promise several installed versions or a runtime version-switch helper. Honor an explicitly requested major and verify the available installation before executing.

Resolve the exact `pg_config` executable and check `--version` and `--pgxs`. In the reference image, the intended installation is under `/usr/lib/postgresql/<major>/bin/`; verify it rather than relying on `PG_VERSION` or PATH. Confirm compatible server/client binaries, libraries, installed extensions, and Pgrx configuration.

Pass the verified absolute `PG_CONFIG` explicitly to each Make invocation when selecting a non-default installation. Supply the matching `PGVERSION` to Pgrx-host tasks and verify their feature/configuration mapping separately. Environment variables set in one tool invocation do not persist to the next: use explicit arguments or environment options on every dependent invocation, or an existing checked-in workflow that keeps setup and execution together. Recheck if the selected installation changes.

Changing `PG_VERSION`, `PGVERSION`, or `PG_CONFIG` does not provision missing binaries or update installed artifacts. If the requested major/tool is absent, report provisioning as blocked; do not install tools on the host, invoke package managers, rebuild the image, alter configuration, or substitute another major automatically. Follow the build skill's clean-build rules before reusing a checkout across versions.

## Results and failures

Capture command, location, source identity, transport mode, version/configuration, output, exit status, and current-run artifacts. Inspect generated files in the shared checkout and preserve unrelated edits. Report transport failure separately from a build/test failure; do not silently retry on the host or against another database.

Never print authentication state, credentials, or secret-bearing environment dumps. A timeout or partial log is not success. Direct-mode metadata checks, static transport walkthroughs, live container discovery, and command execution are distinct evidence; report only what occurred.
