---
title: Contributing and Development Setup
description: Choose a BaudBound repository, prepare its toolchain, and run its quality gates.
tags: [developers, setup, contributing, testing]
---
# Contributing and Development Setup

BaudBound uses separate repositories. Clone only the product you plan to change. A change that alters a shared format may require coordinated pull requests in the contracts, editor, runner, and documentation repositories.

## Requirements

Install Git. Runner development also needs Rust 1.95 or newer, Node.js 24, pnpm 11.10.0, and the platform dependencies required by Tauri 2. Editor, website, documentation, and service development need Node.js 24 and pnpm 11.10.0. Container builds need Docker.

## Clone a repository

Replace `REPOSITORY` with the repository name you need. Cloning with `--recurse-submodules` also initializes the pinned contracts used by the runner and editor. The option is harmless for repositories that do not have a submodule.

```text
git clone --recurse-submodules https://github.com/BaudBound/REPOSITORY.git
cd REPOSITORY
```

Common choices are `baudbound`, `editor`, `contracts`, `documentation`, `get`, `repository`, `tooling`, and `website`.

For an existing runner or editor checkout, initialize its pinned contracts with:

```text
git submodule update --init --recursive
```

## Runner development

Install desktop UI dependencies from the runner repository root:

```text
pnpm --dir ui install --frozen-lockfile
```

The runner source is at the repository root. `src/` contains the application, `ui/` contains the desktop interface, `crates/` contains the Rust libraries, and `contracts/` is the pinned contracts submodule.

Clone the `tooling` repository beside the runner repository, then start the runner development menu from the tooling repository:

```powershell
./development.ps1 -Action Runner
```

Run the CLI directly:

```text
cargo run -p baudbound -- status
```

Use a disposable runner directory while testing state changes.

```powershell
$env:BAUDBOUND_HOME = Join-Path $env:TEMP "baudbound-development"
cargo run -p baudbound -- status
```

## Editor development

From the editor repository root:

```text
pnpm install --frozen-lockfile
pnpm dev
```

The editor uses the shared contracts repository as a submodule under `contracts/`. Update its pinned commit deliberately when shared contracts change.

## Multi-repository development

Clone the `tooling` repository beside the other BaudBound repositories.

Use repository names for the sibling directories. A complete local checkout can look like this:

```text
BaudBound/
  baudbound/
  contracts/
  documentation/
  editor/
  get/
  repository/
  tooling/
  website/
```

From the tooling repository root, open the shared development menu:

```powershell
./development.ps1
```

The shared helper can launch the runner, editor, website, and get service. It can also validate contracts and run checks or builds across the maintained code repositories. Keep the repositories as sibling directories so the helper can discover them by their GitHub repository names.

## Documentation development

From the documentation repository root:

```text
pnpm --dir publisher install --frozen-lockfile
pnpm --dir publisher test
pnpm --dir publisher validate
```

Public pages live under `wiki/`.

## Quality gates

Runner changes normally require:

```text
cargo fmt --all -- --check
cargo clippy --workspace --all-targets --all-features -- -D warnings
cargo test --workspace --all-targets --all-features
pnpm --dir ui typecheck
pnpm --dir ui test
pnpm --dir ui build
```

Editor changes normally require:

```text
pnpm lint
pnpm typecheck
pnpm schemas:check
pnpm test
pnpm build
```

Run the checks owned by every repository changed by your work.

## Contribution standards

Start from current `master`. Keep a pull request focused on one coherent behavior. Add tests at the owning layer. Update public documentation when user behavior, configuration, operations, or contracts change.

Code contributions use the PolyForm Noncommercial License 1.0.0. Documentation and original non-code creative contributions use CC BY-NC-SA 4.0. See [Licensing and Attribution](../licensing.md).

Do not commit generated build directories, signing keys, runner state, private `.bbs` packages, or local plan files.
