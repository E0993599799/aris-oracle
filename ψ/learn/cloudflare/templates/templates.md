# Cloudflare Templates Learning Index

## Source

- **Origin**: `./origin/` (symlink to `/home/marcuz/ghq/github.com/cloudflare/templates`)
- **GitHub**: https://github.com/cloudflare/templates
- **Project Type**: Monorepo template marketplace for Cloudflare Workers
- **License**: See repository

---

## Explorations

### 2026-08-07 09:34 (default: 3 agents)

**Documentation Generated:**
- [[2026-08-07/0934_ARCHITECTURE|Architecture]] — Repository structure, components, entry points, build system
- [[2026-08-07/0934_CODE-SNIPPETS|Code Snippets]] — Key configurations, CLI patterns, E2E testing, monorepo setup
- [[2026-08-07/0934_QUICK-REFERENCE|Quick Reference]] — Getting started, template catalog, common patterns, troubleshooting

---

## Key Insights

### 1. **Monorepo as Template Marketplace**
The repository packages 36+ production-ready starter templates in a single, centralized, heavily-tested space. Each template is end-to-end validated via Playwright before exposure. This reduces friction for developers — instead of "how do I build X with Workers," they have reference implementations.

### 2. **Turborepo + pnpm = Parallel Efficiency**
Build orchestration via Turbo enables parallelization across independent templates. Single pnpm lockfile manages dependencies for entire workspace, preventing version conflicts. The system auto-generates lockfiles, validates consistency, and enforces formatting in a single `pnpm run check` command. This is a scalable pattern for multi-project workflows.

### 3. **Zero-Friction Onboarding**
Two entry points (dashboard for instant deployment, C3 CLI for local dev) + auto-generated git hooks + auto-deployed live demos = developers go from "zero to deployed" in <5 minutes with minimal cognitive load. The postinstall hook runs `turbo build` + `setup-hooks`, so `pnpm install` is your only setup step.

### 4. **CLI Abstraction Over Raw Commands**
The `/cli/` tool exposes high-level commands (`templates lint`, `templates deploy-live-demos`, `templates setup-hooks`) instead of requiring developers to know Turbo, Prettier, Playwright, etc. individually. This reduces onboarding time and creates a single point of version control.

### 5. **Comprehensive E2E Validation**
Playwright tests run in two modes:
- **Local**: Spins up dev servers, validates against localhost
- **Live**: Tests against deployed templates on Cloudflare infrastructure

This ensures templates work not just in development but in production. The system is used to validate every template before it ships.

---

## Architecture Overview

```
cloudflare/templates/ (monorepo)
├── 36+ template directories (React, Remix, Astro, Next.js, AI, Full-stack, etc.)
├── cli/ (validation + deployment CLI)
├── playwright-tests/ (end-to-end test suite)
└── Root configs (turbo.json, pnpm-workspace.yaml, package.json)

Build pipeline:
  C3 CLI or Dashboard → pnpm install
    ↓
  postinstall hook: turbo build + setup-hooks
    ↓
  pnpm run dev (local testing)
    ↓
  pnpm run deploy (wrangler → Cloudflare Workers/Pages)
    ↓
  Playwright E2E validation (local or live mode)
    ↓
  Deployed on 300+ edge locations globally
```

---

## Template Categories (36+)

| Category | Examples |
|----------|----------|
| **Frontend Frameworks** | React, Remix, Astro, Next.js, React Router |
| **Full-Stack** | React + Postgres, React + Router + Postgres, React + Router + Postgres SSR |
| **Backend/Workers** | Hello World, Hono, HTTP Server |
| **Database** | D1, D1 Sessions, PostgreSQL Hyperdrive, MySQL Hyperdrive |
| **Storage** | R2 Explorer, KV To-Do List |
| **Compute** | Durable Objects, Durable Chat, Workflows, Containers |
| **AI/LLM** | LLM Chat App, Text-to-Image, Agent Commerce Analytics, Agent Visibility, Brand Visibility |
| **Auth** | OpenAuth |
| **Advanced** | Microfrontend, Multiplayer Globe, Worker Publisher, X402 Proxy, nlweb |
| **Domain-Specific** | SaaS Admin, Commerce LLMs, E2E testing with Playwright |

---

## Quick Stats

- **Templates**: 36+
- **Test Coverage**: Playwright E2E for every template
- **Node Requirement**: 20.16.0+
- **Package Manager**: pnpm 10.2.0 (pinned)
- **Build Orchestrator**: Turbo
- **Deployment Target**: Cloudflare Workers/Pages/D1/KV/R2/Durable Objects/Hyperdrive/Workflows
- **Zero-Config Onboarding**: ~2 minutes to deployed via C3 CLI or dashboard

---

## Key Technologies

- **Monorepo**: pnpm workspaces + Turborepo
- **Testing**: Playwright (E2E) + Vitest (Unit)
- **Deployment**: Wrangler CLI + Cloudflare API
- **Quality**: Syncpack (version consistency) + Prettier (formatting) + Custom CLI (validation)
- **TypeScript**: Full type safety across all templates
- **Git Hooks**: Auto-setup for pre-commit validation

---

## Next Steps for Learning

1. **Quick Start**: Try C3 CLI on React or Hello World template
2. **Deep Dive**: Explore `/cli/` for validation/deployment patterns
3. **Testing**: Review `/playwright-tests/` for E2E testing approach
4. **Scaling**: Study `turbo.json` and `pnpm-workspace.yaml` for multi-project management
5. **Architecture**: Reference specific templates (e.g., react-postgres-fullstack) for full-stack patterns

---

## References

- **Repository**: https://github.com/cloudflare/templates
- **Cloudflare Docs**: https://developers.cloudflare.com/workers/
- **Turborepo**: https://turbo.build/
- **pnpm**: https://pnpm.io/
- **Playwright**: https://playwright.dev/
- **C3 CLI**: https://developers.cloudflare.com/pages/get-started/c3/
- **Community**: https://workers.community/

---

*Learning session: 2026-08-07 09:34 | Generated via /learn skill with 3-agent exploration*
