# Cloudflare Templates Repository Architecture

## Overview

A monorepo containing 36+ production-ready starter templates for building serverless applications on Cloudflare Workers, Pages, and related services. Managed as a **Turborepo** workspace with pnpm package manager.

## Directory Structure

```
cloudflare/templates/
├── agent-*-template/                    # AI agent templates (commerce, visibility, brand)
├── astro-*-template/                    # Astro framework templates
├── react-*-template/                    # React framework templates (various starters)
├── remix-*-template/                    # Remix framework templates
├── next-*-template/                     # Next.js templates
├── *-fullstack-template/                # Full-stack templates (postgres, mysql, etc.)
├── saas-admin-template/                 # SaaS admin dashboard
├── d1-*-template/                       # Cloudflare D1 database templates
├── durable-*-template/                  # Durable Objects templates
├── hyperdrive-*-template/               # Database connectivity templates
├── kv-*-template/                       # Key-value storage templates
├── r2-*-template/                       # R2 object storage templates
├── *-chat-*-template/                   # Chat/LLM application templates
├── container-*-template/                # Container runtime templates
├── worker-*-template/                   # Worker utility templates
├── openauth-*-template/                 # Authentication templates
├── workflows-*-template/                # Workflows engine templates
│
├── cli/                                 # Custom CLI tool for template management
├── playwright-tests/                    # E2E testing suite
│
├── package.json                         # Root workspace config
├── pnpm-workspace.yaml                  # pnpm monorepo definition
├── turbo.json                           # Turborepo configuration
├── pnpm-lock.yaml                       # Lock file
├── templates.json                       # Template metadata registry
├── playwright.config.ts                 # E2E test configuration
├── vitest.config.ts                     # Unit test configuration
│
└── AGENTS.md / CLAUDE.md / README.md   # Documentation
```

## Core Components

### 1. Template Directories (36+ templates)
Each template is a self-contained project with:
- `package.json` — Dependencies and scripts
- Source code (TypeScript, JavaScript, React, etc.)
- Configuration files (wrangler.toml, next.config.js, etc.)
- `README.md` — Setup and usage instructions

**Categories:**
- **Framework-based**: React, Remix, Astro, Next.js
- **Database**: D1, PostgreSQL, MySQL (via Hyperdrive)
- **Features**: AI/LLM, Full-stack, Streaming, Chat
- **Infrastructure**: Durable Objects, KV, R2, D1, Workflows
- **Auth & Security**: OpenAuth, Proxy templates

### 2. CLI (`/cli/`)
Custom CLI tool for:
- Template validation and linting
- Generating lockfiles across templates
- Deploying live demos
- Setting up git hooks

### 3. Testing Infrastructure
- **Playwright E2E tests** (`/playwright-tests/`) — End-to-end testing for all templates
- **Local mode**: Spins up dev servers for testing
- **Live mode**: Tests against deployed templates
- **Vitest config** — Unit testing support

### 4. Build & Monorepo Management
- **Turborepo** (`turbo.json`) — Orchestrates build, check, deploy across workspaces
- **pnpm workspaces** (`pnpm-workspace.yaml`) — Dependency management
- **Node.js 20.16+** — Runtime requirement

## Entry Points & Key Files

### Root Configuration
- `turbo.json` — Defines tasks: build, check, cf-typegen, deploy, test
- `pnpm-workspace.yaml` — Declares workspace packages
- `package.json` — Root scripts and devDependencies
- `templates.json` — Template metadata and registry

### CLI Integration
- `/cli/` — Linting, validation, deployment commands
- Hooked into `postinstall` for automatic setup
- Commands: `check`, `lint`, `deploy-live-demos`, `generate-npm-lockfiles`

### Documentation
- `README.md` — Getting started (dashboard vs CLI)
- `CONTRIBUTING.md` — Contribution guidelines
- `AGENTS.md` / `CLAUDE.md` — Agent/AI configuration
- `CODE_OF_CONDUCT.md` — Community guidelines

## Dependencies & Tooling

### DevDependencies
- **@playwright/test@1.57.0** — E2E testing
- **turbo@2.6.1** — Monorepo task orchestration
- **prettier@3.7.4** — Code formatting
- **vitest@4.0.14** — Unit testing
- **syncpack@13.0.4** — Dependency version consistency
- **@types/node@24.10.1** — TypeScript types

### Runtime
- **node-fetch@3.3.2** — HTTP client (shared)
- **pnpm@10.2.0** — Package manager (pinned)

### Special Built Dependencies
```json
"onlyBuiltDependencies": ["esbuild", "sharp", "workerd"]
```
These are pre-compiled to avoid build issues across templates.

## Module Interactions

```
CLI (validation/deployment)
  ↓
Turbo (task orchestration)
  ├→ Templates (build, check)
  ├→ Playwright (E2E tests)
  └→ TypeScript generation (cf-typegen)

Package Manager (pnpm)
  ↓
Workspaces (36+ templates)
  ├→ Framework templates (React, Remix, Astro, etc.)
  ├→ Infrastructure (D1, KV, R2, Durable Objects)
  └→ Full-stack applications
```

## Deployment Model

- **Local dev**: C3 CLI or npm create cloudflare@latest
- **Dashboard**: Web UI for quick starts
- **Live demos**: Deployed versions tested via Playwright
- **Production**: Wrangler deploys to Cloudflare Workers/Pages

## Quality Assurance

**Pre-commit checks** (via CLI hooks):
1. Template linting
2. Prettier formatting
3. npm lockfile validation
4. Turbo cache validation

**CI/CD pipeline**:
1. E2E tests (Playwright) — local and live modes
2. Unit tests (Vitest)
3. Type generation and checking
4. Deployment to live demos

## Key Patterns

- **Monorepo as template marketplace** — Centralized, tested, approved templates
- **Zero-config onboarding** — C3 or dashboard removes setup friction
- **Turbo for parallelization** — Fast CI/CD with task caching
- **Playwright for comprehensive validation** — Ensures templates work end-to-end
- **CLI-driven automation** — Consistent linting, formatting, deployment
