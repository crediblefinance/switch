# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Hyperswitch — an open-source payment router written in Rust. It routes payment transactions across multiple payment processors (connectors) with support for smart routing, retries, and multi-tenancy. Built by Juspay.

## Common Commands

### Build & Run

```bash
cargo build                          # Debug build
cargo build --release --features release  # Optimized release build
cargo run --package router           # Run the payment router server
just run                             # Same via justfile
just build_v2                        # Build with v2 features only
just run_v2                          # Run with v2 features only
```

### Lint & Format

```bash
cargo +nightly fmt --all             # Format (requires nightly toolchain)
just clippy                          # Lint v1 (auto-excludes v2 features)
just clippy_v2                       # Lint v2 (auto-excludes v1 features)
just precommit                       # fmt + clippy
```

Important: `v1` and `v2` features are **mutually exclusive**. The justfile recipes handle this automatically by computing the correct feature set via `cargo metadata` + `jq`. Don't pass `--all-features` to clippy/check — use the justfile recipes instead.

### Testing

```bash
cargo test --all-features            # Run all unit tests
cargo nextest run                    # Run tests with nextest (faster, no doctests)
cargo test --doc                     # Doctests only (nextest skips these)
cargo test -p <crate_name>           # Test a single crate
```

E2E tests use Cypress in `cypress-tests/` (v1) and `cypress-tests-v2/` (v2). Load tests use K6 in `loadtest/`.

### Database Migrations

Requires `diesel_cli` and `just`. Default DB: `postgresql://db_user:db_pass@localhost:5432/hyperswitch_db`

```bash
just migrate                         # Run v1 migrations
just migrate_v2                      # Run v2 migrations (includes v1 + compatible + v2)
just migrate_v2_compatible           # Run v1 + v2-compatible migrations only
just migrate redo                    # Redo last v1 migration
just resurrect                       # Drop and recreate the database
```

Migration schema outputs:
- v1 → `crates/diesel_models/src/schema.rs` (configured in `diesel.toml`)
- v2 → `crates/diesel_models/src/schema_v2.rs` (configured in `diesel_v2.toml`)

### Docker Local Setup

```bash
scripts/setup.sh                     # Interactive setup (detects Docker/Podman)
docker compose up -d                 # Start infrastructure (Postgres, Redis)
```

## Architecture

### Crate Layers (bottom-up dependency order)

| Layer | Crates | Purpose |
|-------|--------|---------|
| **Primitives** | `common_enums`, `cards`, `masking`, `router_derive`, `router_env` | Shared enums, derive macros, environment config |
| **Utilities** | `common_utils`, `common_types` | Crypto, hashing, ID generation, shared types |
| **Interfaces** | `api_models`, `hyperswitch_domain_models`, `hyperswitch_interfaces` | API request/response types, domain models, connector trait definitions |
| **Storage** | `diesel_models`, `redis_interface`, `storage_impl` | Diesel ORM models, Redis cache, DB read/write with encryption |
| **Connectors** | `hyperswitch_connectors` | 100+ payment processor integrations |
| **Business Logic** | `payment_methods`, `payment_link`, `subscriptions`, `pm_auth` | Payment method management, links, subscriptions |
| **Application** | `router` | Main HTTP server (Actix-web), routes, core payment flows |
| **Background** | `scheduler`, `drainer`, `analytics` | Job scheduling, event draining, analytics |
| **DSL** | `euclid`, `euclid_wasm`, `kgraph_utils` | Routing rules DSL, compiled to WebAssembly |

### Key Abstractions

**Connector integration** is trait-based. Each connector implements:
```
ConnectorIntegration<Flow, Request, Response>
```
where `Flow` is a payment operation (Authorize, Capture, Void, Refund, etc.). The core generic type threading everything together is:
```
RouterData<Flow, Request, Response>
```

**Model conversion** flows through three layers: API models ↔ Domain models ↔ Storage (Diesel) models, using `ForeignFrom`/`ForeignTryFrom` and `Conversion`/`ReverseConversion` traits.

**Error handling** uses the `error_stack` crate with `.change_context()` for propagation. Error hierarchy: `ApiErrorResponse` (HTTP) → `ConnectorError` / `StorageError` (internal).

### Storage Architecture

`RouterStore<T: DatabaseStore>` combines:
- PostgreSQL via Diesel (master for writes, replica for reads)
- Redis cache layer with pub/sub
- Transparent encryption/decryption of sensitive fields

### Configuration

TOML-based config loaded from `config/` directory. Key files:
- `config/development.toml` — local dev settings
- `config/docker_compose.toml` — Docker environment
- `config/config.example.toml` — all available settings

Infrastructure: PostgreSQL (port 5432), Redis (port 6379).

### Feature Flags

v1 (default) and v2 are **mutually exclusive** feature flags controlling API versions. Many crates have conditional compilation (`#[cfg(feature = "v1")]` / `#[cfg(feature = "v2")]`). Other notable features: `payouts`, `frm`, `recon`, `dynamic_routing`, `revenue_recovery`, `dummy_connector`.

### Adding a New Connector

Use `scripts/add_connector.sh` — an interactive wizard. Also see `add_connector.md` and `connector-template/` for the boilerplate structure.

## Code Conventions

- **MSRV**: Rust 1.85.0
- **Formatter**: nightly rustfmt with `group_imports = "StdExternalCrate"`, `imports_granularity = "Crate"`
- `unsafe_code` is **forbidden**
- Clippy warns on: `unwrap_used`, `expect_used`, `panic`, `todo`, `print_stdout`, `indexing_slicing`, and ~20 more (see `Cargo.toml` workspace lints)
- In tests: `unwrap`, `expect`, `panic`, `dbg!`, `print!` are allowed (`.clippy.toml`)
- Conventional commits: `feat`, `fix`, `perf`, `refactor`, `test`, `docs`, `chore`, `build`, `ci`
