---
title: Testing and CI
description: Select and run BaudBound editor, runner, desktop, schema, container, release, and wiki quality gates.
tags: [developers, testing, ci, quality]
---
# Testing and CI

BaudBound tests contracts at their owning layer and repeats cross-language checks where a change can break editor-to-runner compatibility. Run the narrow tests while developing and the complete affected quality gate before review.

## Rust workspace

From the repository root:

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --locked -- -D warnings
cargo test --workspace --locked
cargo build -p baudbound --locked
```

The workspace includes unit and integration tests for package parsing, security abuse cases, SQLite migrations and lifecycle, graph execution, actions, triggers, core orchestration, CLI behavior, Tauri-facing application functions, and supported native adapters.

Windows and Linux CI both run the workspace. Platform-gated tests prove only the implementation compiled for that runner. They do not make unsupported native behavior portable.

### Rust dependency advisories

Install the exact scanner version used by CI:

```text
cargo install cargo-deny --version 0.19.8 --locked
```

Audit both supported release targets:

```text
cargo deny --all-features --locked --target x86_64-pc-windows-msvc check advisories
cargo deny --all-features --locked --target x86_64-unknown-linux-gnu check advisories
```

`deny.toml` owns advisory policy. Vulnerabilities and yanked versions fail the build. Direct workspace dependencies with unsoundness advisories fail as well. Do not add an ignored advisory without a written impact analysis, an owner, and a removal condition.

Runner CI repeats this check on pull requests, pushes, manual runs, and a daily schedule. The release workflow repeats it before packaging. The action is pinned to a complete commit SHA and evaluates the committed `Cargo.lock`.

Dependabot opens weekly Cargo and GitHub Actions update pull requests from `.github/dependabot.yml`. Repository maintainers must also keep **Dependency graph**, **Dependabot alerts**, and **Dependabot security updates** enabled under GitHub **Settings > Code security**. The repository configuration file schedules updates but cannot enable those repository-level switches.

## Editor

From the editor repository root, install dependencies once:

```text
pnpm install --frozen-lockfile
```

Run:

```text
pnpm lint
pnpm typecheck
pnpm schemas:check
pnpm test
pnpm build
pnpm e2e
```

`schemas:check` regenerates node and program contracts in memory and fails when committed schemas or the Rust capability contract are stale. Editor contract tests cover package writing, schemas, verification, simulation, variables, runtime compatibility, and related serialization behavior.

Playwright exercises the built UI and should be used for interaction, responsive layout, canvas, modal, keyboard, and export changes.

## Desktop UI

Run these commands from the runner repository root:

```text
pnpm --dir ui install --frozen-lockfile
pnpm --dir ui typecheck
pnpm --dir ui test
pnpm --dir ui build
```

Vitest contracts cover view-model formatting and client behavior. Typecheck verifies Tauri payload assumptions on the TypeScript side. The production Vite build is also consumed by the Rust/Tauri bundle.

## Wiki and publisher

```text
pnpm --dir publisher install --frozen-lockfile
pnpm --dir publisher test
pnpm --dir publisher validate
```

Publisher tests cover metadata, links, assets, navigation, GraphQL safety, ownership, adoption, deletion limits, and reconciliation. Validation checks every managed page and documentation coverage contract before publication.

## GitHub workflows

| Workflow | Purpose |
| --- | --- |
| **Runner CI** | Windows and Linux Rust workspace, desktop UI gates, contract snapshots, and scheduled RustSec advisory checks |
| **Runner Release** | Full quality and advisory gates, version verification, signed Windows/Linux packages, updater metadata, and draft release |
| **Editor CI** | Validate the editor, its pinned contracts submodule, and its container image in `BaudBound/editor` |
| **Contracts CI** | Validate JSON contracts and publish their static container image in `BaudBound/contracts` |
| **Wiki Documentation** | Validate and reconcile repository pages and static navigation with Wiki.js |

### Cross-repository contract checks

The editor produces the package language consumed by Rust. The editor and runner pin exact reviewed commits from `BaudBound/contracts` through Git submodules. Local CI verifies the pinned revision against the consumer implementation. Contract updates therefore require a contracts change followed by coordinated consumer pull requests that update their submodule pointers.

## Change-to-test matrix

| Change | Minimum required gates |
| --- | --- |
| Editor UI only | editor lint, typecheck, tests, build. Playwright for interaction/layout |
| Node definition | editor schemas, tests, typecheck, build. Rust security/runtime tests. Documentation coverage |
| Package or schema contract | editor schema and package tests. Full Rust workspace. Schema image |
| Rust crate | fmt, clippy, affected tests, then workspace tests |
| Native action or trigger | workspace tests on affected platforms. Target compatibility. Editor contract |
| SQLite schema | storage migration tests, core/CLI lifecycle tests, backup documentation review |
| CLI command | Clap tests, CLI integration tests, CLI documentation coverage |
| Tauri command or payload | Rust tests, desktop UI typecheck/tests/build |
| Desktop responsive UI | desktop tests/build plus manual narrow/wide inspection |
| Container/deployment | image build, health check, clean deployment test, rollback review |
| Wiki page/navigation/publisher | publisher tests and validate, rendered wide/narrow inspection |
| Release tooling/version | release verification helper and full release quality gate |

## Development helpers

From the `tooling` repository, `./development.ps1` opens an arrow-key menu for all local development helpers. It can launch the runner development menu, editor, website, and get service. It can also validate contracts and run shared checks, builds, or dependency installation.

Choose **Build runner packages** for a local runner package build. The next menu asks for **Both**, **Linux**, or **Windows**. Windows packages are NSIS installers built directly on Windows. Linux builds produce the AppImage, Debian package, and RPM package in a local Ubuntu 22.04 Docker container. Docker Desktop must be running with Linux containers when building Linux on Windows.

Local packages are unsigned and intended only for development and installation testing. The Linux build also inspects the Debian and RPM metadata, dependencies, files, desktop entry, and package scripts. Published releases must still use the signed release workflow.

The same task can be started without the menu:

```powershell
./development.ps1 -Action Runner -RunnerAction RunnerBuild -Platform Both
```

Build output is kept separate from normal development binaries:

| Platform | Output directory |
| --- | --- |
| Windows | `target/local-build/windows/release/bundle/nsis` |
| Linux host | `target/local-build/linux/release/bundle/appimage`, `deb`, and `rpm` |
| Linux through Docker | `target/local-build/linux/artifacts` |

Direct commands remain authoritative and are preferable in CI or when diagnosing one failing tool.

## Local and CI limitations

CI cannot prove every physical-device, desktop-session, window-manager, audio, serial, firewall, reverse-proxy, or updater condition. Tests should isolate deterministic contracts and reject unavailable native features clearly.

Before release, maintainers separately perform platform acceptance checks described in [Release Engineering](releases.md). Do not weaken a deterministic test because a physical environment is unavailable. Separate the adapter contract from manual acceptance instead.

## Failure discipline

Fix warnings rather than suppressing them globally. Preserve the first useful error and add context at ownership boundaries. A flaky test must be diagnosed as a product or test-isolation defect, not merely retried until green.

Do not update snapshots, schemas, or generated contracts without reviewing the semantic change they record.
