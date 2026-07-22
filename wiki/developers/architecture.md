---
title: Repository Architecture
description: Understand the BaudBound repositories, Rust crates, contracts, process boundaries, and state flow.
tags: [developers, architecture, rust, editor]
---
# Repository Architecture

BaudBound is developed as a group of focused repositories in the
[BaudBound GitHub organization](https://github.com/BaudBound). Each repository owns one deployable product or one clearly defined shared contract.

## Repository map

| Repository | Responsibility |
| --- | --- |
| [`BaudBound/BaudBound`](https://github.com/BaudBound/BaudBound) | Runner CLI, headless service, Tauri desktop application, desktop UI, Rust crates, and runner release tooling |
| [`BaudBound/editor`](https://github.com/BaudBound/editor) | Next.js visual editor, node registry, verification, simulation, package export, and editor container |
| [`BaudBound/contracts`](https://github.com/BaudBound/contracts) | Published JSON Schemas and machine-readable editor-to-runner contracts |
| [`BaudBound/documentation`](https://github.com/BaudBound/documentation) | Public wiki source, navigation, validation, and Wiki.js publisher |
| [`BaudBound/get`](https://github.com/BaudBound/get) | Hosted Linux and Windows installation scripts at `get.baudbound.app` |
| [`BaudBound/website`](https://github.com/BaudBound/website) | Public BaudBound website and container image |
| [`BaudBound/repository`](https://github.com/BaudBound/repository) | Official BaudBound script repository metadata and packages |

## Contract boundary

The contracts repository publishes reviewed snapshots. The editor and runner vendor the exact contract revision they build against. Builds do not download moving files from another repository.

When a contract changes:

1. Update the owning implementation and generate the changed contract.
2. Review and publish the contract change in `BaudBound/contracts`.
3. Update the vendored snapshot in each affected consumer.
4. Run the consumer's local contract checks.
5. Update this documentation when user-visible behavior changed.

This keeps builds reproducible while still making compatibility changes explicit in code review.

## Runner crate ownership

| Crate | Owns |
| --- | --- |
| `baudbound-script` | `.bbs` archive parsing, package documents, path validation, asset validation, and package models |
| `baudbound-security` | Permission and capability recalculation, risk, approval policy, and security validation |
| `baudbound-storage` | SQLite schema, installed packages, approvals, runs, variables, signals, and encrypted secret records |
| `baudbound-runtime` | Graph execution, templates, conditions, loops, variables, cancellation, reports, and redaction |
| `baudbound-actions` | Cross-platform action adapters and action-handler contracts |
| `baudbound-triggers` | Schedule, file, process, webhook, WebSocket, hotkey, startup, and serial listeners |
| `baudbound-core` | Validation and orchestration across package, security, storage, runtime, actions, and triggers |

The `apps/baudbound` package in the runner repository composes these crates and provides the CLI, Tauri host, desktop adapters, runner paths, service lifecycle, and update integration.

## Package execution flow

1. The editor creates program steps and security declarations from registered node definitions.
2. Export writes manifest, program, permissions, capabilities, optional editor data, and assets into a `.bbs` package.
3. The runner parses bounded package documents and validates normalized archive paths.
4. The runner recalculates permissions, capabilities, and risk from the program. It does not trust the package declarations.
5. The runner checks identity, target compatibility, installed hash, approval, and required secrets.
6. A manual action or trigger event selects an exact trigger node and payload.
7. The runtime executes connected destinations in their explicit edge order.
8. Action adapters perform supported side effects.
9. The final redacted report is stored and exposed through the CLI and desktop application.

## Process boundaries

The installed `baudbound` executable hosts both the CLI and Tauri desktop application. Desktop UI calls cross the typed Tauri command boundary. The UI does not open SQLite or parse packages itself.

`baudbound serve` runs trigger listeners in the foreground for an external service manager. Independent CLI commands update durable SQLite state. A running service observes reload state and can also use the authenticated loopback control protocol for immediate stop and reload requests.

Configuration lives in `config.toml`. Installed scripts, approvals, runs, variables, and service state live in the runner data directory. See [Storage, Backups, and Recovery](../runner/storage-backups.md) for operator behavior.

Continue with [Editor and Package Development](editor.md), [Runner Development](runner.md), and [Testing and CI](testing-ci.md).
