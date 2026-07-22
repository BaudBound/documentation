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

Replace `REPOSITORY` with the repository name you need.

```text
git clone https://github.com/BaudBound/REPOSITORY.git
cd REPOSITORY
```

Common choices are `BaudBound`, `editor`, `contracts`, `documentation`, `get`, and `website`.

## Runner development

Install desktop UI dependencies from the runner repository root:

```text
pnpm --dir apps/baudbound/ui install --frozen-lockfile
```

Start the Windows development helper:

```powershell
./tools/development.ps1
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

The editor vendors its pinned schema and runner contract snapshots under `contracts/`. Update them deliberately when the shared contracts revision changes.

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
pnpm --dir apps/baudbound/ui typecheck
pnpm --dir apps/baudbound/ui test
pnpm --dir apps/baudbound/ui build
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
