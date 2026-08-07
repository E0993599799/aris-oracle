# Cloudflare Templates: Code Snippets & Patterns

## Root Package Configuration

### Package.json Structure
```json
{
  "name": "templates",
  "packageManager": "pnpm@10.2.0",
  "engines": {
    "node": ">=20.16.0 || >=22.3.0"
  },
  "pnpm": {
    "onlyBuiltDependencies": ["esbuild", "sharp", "workerd"]
  }
}
```

**Pattern**: Pinned pnpm version and Node range ensures consistent builds across all templates.

---

## Turborepo Task Configuration

### turbo.json
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": {
      "outputs": [".astro/**", ".next/**", ".react-router/**", ".wrangler/**", "build/**", "dist/**"]
    },
    "check": {
      "dependsOn": ["build"]
    },
    "cf-typegen": {
      "dependsOn": ["build"]
    },
    "deploy": {
      "env": ["CLOUDFLARE_ACCOUNT_ID", "CLOUDFLARE_API_TOKEN"]
    },
    "test": {}
  }
}
```

**Pattern**: 
- Tasks declare output caching to speed up rebuilds
- `check` depends on `build` to validate built artifacts
- `deploy` requires environment variables (secrets)
- Turbo parallelizes independent template builds

---

## Root Scripts & CLI Integration

### Key NPM Scripts

```json
"scripts": {
  "check": "pnpm run check:templates && pnpm run check:lockfiles && pnpm run check:turbo && pnpm run check:prettier && git diff --exit-code",
  "deploy": "turbo run deploy",
  "fix": "pnpm run fix:templates && pnpm run fix:lockfiles && pnpm run fix:turbo && pnpm run fix:prettier",
  "postinstall": "turbo run build && templates setup-hooks",
  "test": "turbo run test -- --passWithNoTests",
  "test:e2e": "playwright test",
  "info:deps": "templates deps-info"
}
```

**Pattern**:
- **check** — Multi-step validation: linting → lockfiles → turbo → prettier → git diff
- **fix** — Auto-fixes templates, lockfiles, formatting (idempotent)
- **postinstall** — Automatically builds and sets up git hooks on install
- **test** — Runs turbo tests with lenient failure handling (`--passWithNoTests`)

---

## CLI Tool Patterns

### Commands Structure (in `/cli/`)

```bash
# Template validation & linting
templates lint .                          # Lint all templates
templates lint . --fix                    # Auto-fix linting issues

# Lockfile management
templates lint-npm-lockfiles .            # Validate lockfiles
templates generate-npm-lockfiles .        # Regenerate lockfiles

# Deployment
templates deploy-live-demos .             # Deploy all demo sites
templates setup-hooks                     # Install git hooks

# Dependency analysis
templates deps-info                       # Show dependency information
```

**Pattern**: Single `templates` CLI command with subcommands reduces cognitive load for developers.

---

## E2E Testing with Playwright

### Test Configuration Pattern

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './playwright-tests',
  fullyParallel: true,
  workers: process.env.CI ? 1 : undefined,
  retries: process.env.CI ? 2 : 0,
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
  },
  webServer: {
    // Dev server setup for local testing
  },
});
```

**Pattern**:
- Uses environment variables to adjust CI vs local behavior
- Enables full parallelization for speed
- Traces on first retry for debugging
- Can test against local dev servers or deployed live demos

### E2E Test Modes

**Local Development Mode** (default):
```bash
pnpm run test:e2e
pnpm run test:e2e astro-blog-starter-template.spec.ts
```
Starts dev server, runs tests, cleans up.

**Live Mode** (deployed):
```bash
PLAYWRIGHT_USE_LIVE=true pnpm run test:e2e
```
Tests against production deployed templates.

**UI Mode** (debugging):
```bash
pnpm run test:e2e --ui
```

---

## Monorepo Workspace Configuration

### pnpm-workspace.yaml Pattern

```yaml
packages:
  - 'cli'
  - '*-template'
  - 'playwright-tests'
```

**Pattern**: Glob patterns declare all workspace packages. Each template directory becomes an independent workspace with its own `package.json`.

---

## Template Structure (Individual Template Example)

Each template follows this structure:

```
astro-blog-starter-template/
├── package.json
├── src/
│   ├── components/
│   ├── layouts/
│   └── pages/
├── astro.config.mjs
├── wrangler.toml
├── README.md
└── .npmrc
```

**Common Build Outputs** (declared in turbo.json):
- `.astro/` → Astro build output
- `.next/` → Next.js build output
- `.react-router/` → React Router SSR output
- `.wrangler/` → Wrangler build output
- `build/`, `dist/` → Generic build directories

---

## Dependency Management Patterns

### Syncpack for Version Consistency

```bash
# Check for mismatched dependency versions
pnpm run check:deps

# Auto-fix version mismatches
pnpm run fix:deps
```

**Pattern**: Ensures all templates using (e.g.) React use the same version, preventing incompatibilities.

---

## Git Hooks Integration

**Pattern**: Auto-installed on `postinstall` via `templates setup-hooks`

Typical hooks:
- Pre-commit: Run linting + formatting
- Pre-push: Run tests
- Commit-msg: Validate message format

---

## Type Generation Pattern

### Cloudflare TypeScript Types

```bash
pnpm run cf-typegen    # Generates TypeScript types for Cloudflare services
```

**Pattern**: Turborepo task depends on build outputs, ensuring types reflect deployed environment.

---

## Prettier Code Formatting

### Shared Configuration

```bash
pnpm run check:prettier    # Check formatting
pnpm run fix:prettier      # Auto-fix formatting

# Prettier config likely:
prettier .                 # Formats all JS/TS/JSON/MD in monorepo
```

**Pattern**: Single Prettier config shared across all templates ensures consistency.

---

## Environment Management for Deployment

### Deploy Task Requires

```json
"deploy": {
  "env": ["CLOUDFLARE_ACCOUNT_ID", "CLOUDFLARE_API_TOKEN"]
}
```

**Pattern**: Turborepo enforces required env vars before running deploy tasks. Prevents accidental deploys with missing credentials.

---

## Git Workflow Integration

### Pre-check Validation

```bash
pnpm run check    # Runs all checks and verifies git diff is empty
```

This ensures:
1. All templates lint successfully
2. All lockfiles are valid
3. Turbo cache is consistent
4. Prettier formatting is applied
5. No uncommitted changes remain

**Pattern**: Comprehensive validation gate before committing.

---

## Key Architectural Patterns

| Pattern | Purpose | Implementation |
|---------|---------|-----------------|
| **Monorepo as Marketplace** | Centralized template discovery + testing | 36+ templates in single repo |
| **Turbo Parallelization** | Fast builds across independent projects | Task orchestration with caching |
| **CLI Automation** | Reduce manual steps for contributors | `templates` command with subcommands |
| **Playwright Validation** | Ensure templates work end-to-end | Local + live test modes |
| **pnpm Workspaces** | Efficient dependency sharing | Single lockfile for all templates |
| **Environment Guards** | Prevent deployment without credentials | `deploy` task requires env vars |
| **Git Hooks** | Enforce quality before commits | Auto-setup on `postinstall` |
| **Syncpack Consistency** | Prevent version mismatches | Version linting across workspace |

