---
title: Developer Overview
description: BaudBound architecture, ownership boundaries, state, communication, and trust model.
tags: [developers, architecture]
---
# Developer Overview

This section is for people who want to build BaudBound from source, contribute a change, review architecture, publish documentation, or prepare a release. Product users do not need the developer tools described here.

BaudBound is developed across focused repositories in the [BaudBound GitHub organization](https://github.com/BaudBound). The runner repository contains a Rust workspace with the runner application and its internal crates. The editor, contracts, documentation, installer service, website, official script repository, and shared tooling each have their own repository. Changes should preserve explicit ownership boundaries, deterministic validation, native platform implementations, and tests proportional to behavior and risk.

## Project areas

The [editor repository](https://github.com/BaudBound/editor) contains the Next.js visual editor. The [runner repository](https://github.com/BaudBound/baudbound) contains the unified runner CLI and Tauri desktop application. Shared schemas and machine-readable contracts live in the [contracts repository](https://github.com/BaudBound/contracts). The editor and runner pin that repository as a Git submodule so each build uses one reviewed contracts commit.

| Area | Owns | Start here |
| --- | --- | --- |
| Editor | Project authoring, registry definitions, verification, simulation, package export, generated schemas | [Editor and Package Development](editor.md) |
| Unified runner app | Clap commands, service process, Tauri commands, tray, updater, desktop native adapters | [Runner Development](runner.md) |
| Desktop UI | Operator views and typed Tauri client | [Runner Development](runner.md) |
| Schemas | Public package-document and per-node JSON Schemas | [BBS Package Format](../package-format/index.md) |
| Shared tooling | Development menu, multi-repository checks, local package builds, and release helper | [Contributing and Setup](setup.md) |
| Documentation | Wiki source, validation, Wiki.js publisher, and navigation | [Wiki Documentation](wiki-documentation.md) |
| Workflows | Repository-specific CI, release packaging, containers, contracts, and wiki publication | [Testing and CI](testing-ci.md) |

## Rust crates

`baudbound-script` owns package and language models plus archive validation. `baudbound-security` derives permissions, capabilities, and policy decisions. `baudbound-storage` owns SQLite schema and durable repositories. `baudbound-runtime` executes validated graphs. `baudbound-actions` implements actions. `baudbound-triggers` implements listener services. `baudbound-core` coordinates these domains.

Crate roots expose public APIs and delegate implementation into domain folders. Split unrelated responsibilities before files become difficult to review, but avoid abstractions that obscure ownership.

The normal dependency direction is:

```text
package models -> security/storage/runtime -> actions and triggers -> core orchestration -> CLI/Tauri
```

`baudbound-core` may coordinate lower-level crates. Lower-level crates must not reach upward into the application UI or command modules. See [Repository Architecture](architecture.md) for process and runtime flows.

## State and communication

SQLite is durable runner state and coordinates independent processes. Tauri commands bridge the desktop UI to Rust. TOML is operator configuration, and `.bbs` is the portable script artifact. JSON is used for cross-language package documents and explicit machine-readable output, not as a parallel mutable runner database.

## Trust boundary

The editor produces package declarations. The runner independently verifies package integrity, compatibility, requested access, approval, policy, secrets, and node configuration before native side effects. UI visibility never substitutes for backend authorization.

Unsupported platform behavior must be declared and rejected by both editor and runner rather than approximated with shell scripts.

## Sources of truth

- Editor node definitions own authoring metadata, config fields, outputs, access declarations, and supported target runtimes.
- Generated node schemas are build artifacts checked against those definitions.
- Rust package and security code independently enforce the execution contract. It does not trust editor output.
- `RunnerConfig` Rust structs own `config.toml` shape and defaults.
- Clap enums own CLI commands and options.
- Desktop `navigationGroups` owns operator tab IDs and labels.
- SQLite migrations own durable state. JSON files are not a mutable runner database.
- `wiki/coverage.json` maps these product surfaces to mandatory public documentation.

When two layers need the same contract, derive or test it across the boundary. Do not add another handwritten compatibility table merely because it is convenient locally.

## Choose your next guide

- Set up a clean checkout with [Contributing and Development Setup](setup.md).
- Understand module and process boundaries in [Repository Architecture](architecture.md).
- Add a node or package field through [Editor and Package Development](editor.md).
- Add native runtime behavior through [Runner Development](runner.md).
- Select the required gates using [Testing and CI](testing-ci.md).
- Prepare signed artifacts with [Release Engineering](releases.md).
- Change public documentation with [Wiki Documentation](wiki-documentation.md).
