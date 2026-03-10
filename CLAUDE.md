# CLAUDE.md — AI Assistant Guide for `workflows`

This file documents the codebase structure, development conventions, and key workflows for AI assistants working in this repository.

---

## Project Overview

**Workflows** is a data analysis platform developed by Diamond Light Source. It provides:

- A Kubernetes virtual cluster (vcluster) running **Argo Workflows**
- **Rust microservices** for auth, GraphQL proxy, session management, and telemetry
- A **React/TypeScript** frontend dashboard
- A **Rust CLI** (`workflows-cli`) for linting, submitting, and creating workflow templates
- **Helm charts** for infrastructure deployment via ArgoCD

---

## Repository Structure

```
/
├── backend/                  # Rust microservices (Cargo workspace)
│   ├── argo-workflows-openapi/  # OpenAPI types for Argo Workflows
│   ├── auth-daemon/          # Authentication proxy service
│   ├── graph-proxy/          # Async-GraphQL proxy (Axum)
│   ├── sessionspaces/        # Session management (LDAP, Kubernetes)
│   └── telemetry/            # OpenTelemetry setup
├── frontend/                 # Yarn workspaces monorepo
│   ├── workflows-lib/        # Shared React component library
│   ├── relay-workflows-lib/  # Relay/GraphQL type-safe library
│   └── dashboard/            # Main React dashboard app (Vite)
├── workflows-cli/            # Rust CLI tool
├── charts/                   # Helm charts (16 chart directories)
├── docs/                     # MkDocs documentation
├── examples/                 # Workflow template examples
└── .github/workflows/        # CI/CD (GitHub Actions)
```

---

## Languages & Key Technologies

| Layer | Technology |
|---|---|
| Backend services | Rust, Axum 0.8, async-graphql 7.0, Tokio |
| Frontend | TypeScript, React 18, MUI v5-6, Relay/GraphQL v20.1, Vite 7.2 |
| Testing (FE) | Vitest 4.0, @testing-library/react |
| CLI | Rust, Clap |
| Infrastructure | Kubernetes, Helm, ArgoCD, vcluster, Kyverno, Kueue |
| Observability | OpenTelemetry, OTLP, Prometheus |
| Docs | MkDocs (Tech Docs Diamond theme) |

---

## Development Commands

### Frontend

All frontend commands run from the `/frontend` directory using Yarn workspaces.

```bash
# Install dependencies
yarn install

# Development server (dashboard)
yarn dev

# Development with mocked data (MSW)
yarn mock

# Build (TypeScript + Vite)
yarn build

# Run tests
yarn test

# Type checking
yarn tsc

# Lint (ESLint + Prettier check)
yarn lint

# Fix formatting
yarn format

# Compile Relay GraphQL queries
yarn relay

# Run Storybook
yarn storybook

# Full pre-commit check (lint + tsc + test + format)
yarn precommit
```

**First-time setup for dashboard:**
1. `yarn install`
2. Add the GraphQL supergraph schema
3. `yarn relay` to compile Relay queries
4. Create `frontend/dashboard/.env.local` with Keycloak config
5. `yarn dev`

### Backend (Rust)

Run from `/backend`:

```bash
cargo build
cargo test
cargo clippy
```

### CLI

Run from `/workflows-cli`:

```bash
cargo build
cargo test
```

The CLI requires `helm` and `argo` CLI tools to be available in `$PATH`.

CLI subcommands:
- `lint` — lint workflow templates
- `lint-config` — lint from a config file (e.g. `.workflows-lint.yaml`)
- `submit` — submit workflows/templates
- `create` — scaffold new template repositories

### Infrastructure / Deployment

```bash
# Install main Helm chart
helm install workflows-cluster charts/workflows-cluster

# Dev mode deployment
helm install workflows-cluster charts/workflows-cluster \
  -f charts/workflows-cluster/dev-values.yaml

# Run a command inside the vcluster
vcluster connect workflows-cluster --silent -- <COMMAND>

# Serve documentation locally
mkdocs serve
```

---

## Code Conventions

### Commit Messages

This repository enforces **Conventional Commits** (`@commitlint/config-conventional`).

Format: `type(scope): description`

Common types: `feat`, `fix`, `chore`, `docs`, `refactor`, `test`, `ci`

Examples:
```
feat(frontend): add template filter persistence
fix(auth-daemon): resolve container startup error
chore(charts): bump graph-proxy to v0.1.18
```

### Frontend Formatting

Configured via `/frontend/.prettierrc`:
- Line width: **80 characters**
- Indent: **2 spaces**
- Semicolons: **yes**
- Trailing commas: **yes**
- Single quotes for strings

Always run `yarn format` before committing or run `yarn precommit` for the full suite.

### TypeScript

- Strict TypeScript enabled via `tsconfig.json` project references
- Relay compiler generates types automatically — do not edit `__generated__` files
- ESLint enforces TypeScript and React Hooks rules

### Rust

- Follow standard Rust idioms; `cargo clippy` is enforced in CI
- Use `async`/`await` with Tokio throughout
- Serde derives for serialization: `#[derive(Serialize, Deserialize)]`

---

## Testing Conventions

### Frontend Tests

- Framework: **Vitest**
- Test files live in `tests/` mirroring `lib/`:
  - `tests/functions/*.test.ts` — unit tests for functions
  - `tests/components/*.test.tsx` — component tests
- Use `@testing-library/react` for component rendering

Run with: `yarn test`

### Backend Tests

- Tests live alongside source (Rust standard: `#[cfg(test)]` modules or `tests/` directories)
- Integration test dependencies include `wiremock` and `serial_test`
- Run with: `cargo test`

---

## CI/CD

The main orchestrator is `.github/workflows/ci.yaml`, which coordinates 14 jobs:

| Job | What it does |
|---|---|
| `helm_lint` | Validates Helm charts |
| `kyverno_policy` | Checks Kyverno policies |
| `sessionspaces_code` | Tests + lints sessionspaces crate |
| `graph_proxy_code` / `graph_proxy_schema` | Tests graph-proxy; validates GraphQL schema |
| `auth_daemon_code` | Tests auth-daemon crate |
| `telemetry_code` | Tests telemetry crate |
| `frontend_code` | TypeScript check + lint + tests |
| `cli_code` | Validates CLI crate |
| `commit_lint` | Enforces conventional commits |
| `release_please` | Automated releases on push/tag |
| Container jobs | Build and push Docker images |
| `github_pages` | Deploys MkDocs documentation |

Reusable workflow definitions are in `_*.yaml` files in `.github/workflows/`.

### Release Management

Uses **Release Please** with independent versioning per package:
- `backend/graph-proxy` — Rust strategy
- `backend/sessionspaces` — Rust strategy
- `backend/auth-daemon` — Rust strategy
- `frontend/workflows-lib` — Node strategy
- `frontend/relay-workflows-lib` — Node strategy
- `frontend/dashboard` — Node strategy
- `workflows-cli` — Rust strategy
- Root — Simple strategy

Version tracking is in `.release-please-manifest.json`.

---

## GraphQL Architecture

The frontend uses **Relay Modern** for type-safe GraphQL:

- Relay config: `frontend/relay.config.json`
- Generated types output to `__generated__` directories (do not edit)
- Always run `yarn relay` after modifying `.graphql` files or adding queries/fragments
- The dashboard uses `relay.config.json` with TypeScript language plugin
- Subscriptions use `graphql-ws` over WebSocket

The backend `graph-proxy` serves the GraphQL API using `async-graphql` with Axum.

---

## Key Architectural Notes

1. **Monorepo with separate workspaces**: Frontend uses Yarn workspaces; backend uses a Cargo workspace. They are independent.

2. **vcluster isolation**: The Argo Workflows cluster runs inside a vcluster, isolated from the host cluster.

3. **Multi-package releases**: Each service/package versions independently. Bumping one does not automatically bump others.

4. **Helm chart per service**: Infrastructure changes require updating the relevant chart in `/charts/` and re-deploying via ArgoCD.

5. **LDAP + Keycloak auth**: `sessionspaces` integrates with LDAP; the dashboard frontend requires Keycloak OIDC configuration in `.env.local`.

6. **Workflow linting config**: `.workflows-lint.yaml` at the repo root configures what the `workflows lint-config` CLI command validates.

---

## Common Tasks for AI Assistants

### Adding a new frontend component

1. Create the component in `frontend/workflows-lib/lib/components/`
2. Export it from `frontend/workflows-lib/lib/main.ts`
3. Add a Storybook story in `frontend/workflows-lib/stories/`
4. Write tests in `frontend/workflows-lib/tests/components/`

### Modifying GraphQL schema/queries

1. Update the schema or query/fragment files
2. Run `yarn relay` to regenerate types
3. Do not manually edit files in `__generated__` directories

### Adding a new Helm chart value

1. Edit the appropriate `charts/<name>/values.yaml`
2. Update `charts/<name>/templates/` as needed
3. Run `helm lint charts/<name>` to validate

### Adding a new CLI command

1. Add the variant to the `Commands` enum in `workflows-cli/src/main.rs`
2. Create a new module under `workflows-cli/src/`
3. Wire it up in the `match` block in `main.rs`

### Working with the Rust workspace

- Add new crates to `backend/Cargo.toml` `[workspace]` members
- Shared dependencies should use workspace-level `[workspace.dependencies]`

---

## Package Manager

- **Frontend**: Yarn 1.22.22 (pinned via `.yarnrc.yml`)
- **Backend/CLI**: Cargo (Rust 2024 edition for `workflows-cli`)

Do not use `npm` or `pnpm` for frontend — use `yarn` only.
