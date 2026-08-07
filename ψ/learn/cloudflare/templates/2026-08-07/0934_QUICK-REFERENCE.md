# Cloudflare Templates: Quick Reference

## What Is This?

**Cloudflare Templates** is a curated monorepo of 36+ production-ready starter projects for building serverless applications on **Cloudflare Workers**, **Pages**, and complementary services (D1, KV, R2, Durable Objects, Hyperdrive, Workflows). Each template is tested end-to-end and can be deployed instantly via the Cloudflare dashboard or C3 CLI.

**Key benefit**: Zero-friction project startup with professional architecture baked in.

---

## Installation & Getting Started

### Method 1: Using C3 CLI (Recommended)

```bash
npm create cloudflare@latest
# or
pnpm create cloudflare@latest
# or
yarn create cloudflare@latest
```

This launches an interactive prompt to:
- Select a template
- Configure project name
- Choose deployment settings
- Auto-deploy to Cloudflare

**Requires**: Cloudflare account (free tier available)

### Method 2: Using Cloudflare Dashboard

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com/)
2. Navigate to [Workers Templates](https://dash.cloudflare.com/?to=/:account/workers-and-pages/templates)
3. Browse and select a template
4. Click "Create" to deploy without local setup

**Best for**: Quick testing, no local development environment needed.

### Method 3: Clone from Repository

```bash
git clone https://github.com/cloudflare/templates.git
cd templates
pnpm install
pnpm run dev          # Run all templates in dev mode
```

**Requires**: Node.js 20.16+, pnpm 10.2.0

---

## Available Templates

### Framework-Based (Full-Stack)
- **React Starter** — Minimal React app with Workers backend
- **React + Router** — React Router v7 with full-stack support
- **React + Postgres** — React frontend + Postgres database (Hyperdrive)
- **Remix Starter** — Remix full-stack framework
- **Astro Blog** — Astro static site generator with blog
- **Next.js Starter** — Next.js app with Workers integration

### Backend/Workers
- **Hello World** — Minimal Worker (best starting point)
- **Hono** — Lightweight web framework for Workers
- **Express-like frameworks** — Various framework options

### Database & Storage
- **D1 Starter** — Cloudflare's SQLite database
- **D1 Sessions API** — Session management with D1
- **PostgreSQL Hyperdrive** — External Postgres via Hyperdrive
- **MySQL Hyperdrive** — External MySQL via Hyperdrive
- **R2 Explorer** — Object storage UI and API

### Serverless Compute
- **Durable Objects Chat** — Persistent state + messaging
- **Durable Objects** — Stateful serverless compute
- **Workflows Starter** — Serverless workflow orchestration
- **Containers** — Container runtime templates

### AI/LLM Features
- **LLM Chat App** — Chat interface with language models
- **Agent Commerce Analytics** — AI agents for analytics
- **Agent Visibility** — Observability with AI
- **Text-to-Image** — AI image generation
- **Commerce LLMs** — E-commerce AI features

### Specialized
- **SaaS Admin** — Multi-tenant admin dashboard
- **OpenAuth** — Authentication & authorization
- **Microfrontend** — Module federation patterns
- **Multiplayer Globe** — Real-time collaboration example
- **To-Do List + KV** — Simple KV storage example
- **Worker Publisher** — Event publishing patterns
- **X402 Proxy** — Web monetization proxy
- **nlweb** — Natural language web

---

## Key Features

### ✅ All Templates Include

- **Production-ready code** — Follows Cloudflare best practices
- **TypeScript support** — Full type safety
- **Local dev environment** — `pnpm run dev` for testing
- **Deployment scripts** — Instant Cloudflare deployment
- **README & docs** — Setup and usage instructions
- **E2E tests** — Playwright test suite validates functionality

### ⚡ Performance & Deployment

- **Global deployment** — Deploy to 300+ edge locations instantly
- **Zero cold starts** — Workers start in milliseconds
- **Auto-scaling** — Infinitely scalable on demand
- **Integrated CDN** — Free global caching
- **Free tier** — 100k requests/day included

### 🔒 Security

- **No server management** — Cloudflare handles infrastructure
- **DDoS protection** — Built-in
- **TLS/SSL encryption** — Default on all deployments
- **IAM integration** — Cloudflare account security

---

## Common Usage Patterns

### Start a New Project (Dashboard)

```
1. Cloudflare Dashboard → Workers Templates
2. Pick template (e.g., "React Starter")
3. Click "Create" → auto-deployed
4. Use dashboard URL or connect custom domain
```

### Start a New Project (CLI)

```bash
npm create cloudflare@latest -- --template react-starter
cd my-project
pnpm install
pnpm run dev         # Test locally
pnpm run deploy      # Deploy to Cloudflare
```

### Develop Locally

```bash
pnpm install
pnpm run dev         # Spins up local dev server
# Edit code, auto-reload
```

### Deploy to Production

```bash
pnpm run deploy
# Deploys to Cloudflare Workers/Pages
# URL: https://<project-name>.<account>.workers.dev
```

### Run Tests

```bash
pnpm run test        # Unit tests (Vitest)
pnpm run test:e2e    # End-to-end tests (Playwright)
```

### Add Environment Variables

```bash
# Create .env.local
VITE_API_KEY=your_key

# Or via Cloudflare dashboard:
# Workers → Settings → Environment Variables
```

### Connect to Database

**For D1 (Cloudflare SQLite)**:
```javascript
import { drizzle } from 'drizzle-orm/d1';
const db = drizzle(env.DB);
const users = await db.select().from(users);
```

**For PostgreSQL/MySQL (Hyperdrive)**:
```javascript
const response = await fetch('https://your-hyperdrive.example.com/query', {
  method: 'POST',
  body: JSON.stringify({ query: 'SELECT * FROM users' })
});
```

---

## Project Structure (Typical)

```
my-project/
├── src/
│   ├── index.ts          # Worker entrypoint
│   └── components/       # (if using React)
├── public/               # Static assets
├── wrangler.toml         # Cloudflare config
├── package.json          # Dependencies
├── tsconfig.json         # TypeScript config
├── README.md
└── test/                 # Unit & E2E tests
```

### Key Files

| File | Purpose |
|------|---------|
| `wrangler.toml` | Cloudflare Workers configuration (routes, env vars, bindings) |
| `src/index.ts` | Worker entry point (request handler) |
| `package.json` | Dependencies and build scripts |
| `vite.config.ts` | (React/frontend) Build tool config |
| `.env.local` | Local environment variables (not committed) |
| `playwright.config.ts` | E2E test configuration |

---

## Troubleshooting

### Issue: "Node version mismatch"
**Fix**: Update Node.js to 20.16.0 or later
```bash
node --version   # Check current
nvm install 20.16.0
```

### Issue: "pnpm command not found"
**Fix**: Install pnpm globally
```bash
npm install -g pnpm@10.2.0
```

### Issue: "Module not found" after `pnpm install`
**Fix**: Rebuild dependencies
```bash
pnpm install --force
pnpm run build
```

### Issue: "Deployment fails with 403"
**Fix**: Check Cloudflare API token in `wrangler.toml` or env vars
```bash
wrangler login
# or set CLOUDFLARE_API_TOKEN env var
```

### Issue: Tests fail locally but pass in CI
**Fix**: Clear Turbo cache and rebuild
```bash
pnpm run clean    # (if available)
pnpm install
pnpm run test
```

---

## Configuration & Customization

### Environment Variables

```toml
# wrangler.toml
[env.production]
vars = { ENVIRONMENT = "production" }

[env.staging]
vars = { ENVIRONMENT = "staging" }
```

Deploy to environment:
```bash
wrangler deploy --env staging
```

### Add a Database

```toml
# wrangler.toml
[[d1_databases]]
binding = "DB"
database_name = "my-database"
database_id = "abc123..."
```

### Add KV Storage

```toml
# wrangler.toml
[[kv_namespaces]]
binding = "CACHE"
id = "xyz789..."
```

### Add Custom Domain

```bash
# Via Cloudflare dashboard:
# Workers → Custom Domains → Add Domain
# Then point DNS to Cloudflare nameservers
```

---

## Important Links

- **Cloudflare Dashboard**: https://dash.cloudflare.com/
- **Workers Documentation**: https://developers.cloudflare.com/workers/
- **Pages Documentation**: https://developers.cloudflare.com/pages/
- **D1 Database**: https://developers.cloudflare.com/d1/
- **Durable Objects**: https://developers.cloudflare.com/durable-objects/
- **GitHub Repository**: https://github.com/cloudflare/templates
- **Discord Community**: https://workers.community/

---

## Learning Path

1. **Start**: Deploy "Hello World" template via dashboard (no local setup)
2. **Learn**: Read template README + Cloudflare docs
3. **Develop**: Use C3 CLI to create "React Starter" locally
4. **Database**: Add D1 or Hyperdrive to connect data
5. **Deploy**: Push to production with `pnpm run deploy`
6. **Optimize**: Monitor performance in Cloudflare dashboard

---

## Contribution Guidelines

Want to add a template or improve an existing one?

```bash
git clone https://github.com/cloudflare/templates.git
cd templates
pnpm install
# Edit/add templates
pnpm run check          # Validate changes
pnpm run fix            # Auto-fix formatting
git commit -am "feat: add new template"
git push origin feature-branch
# Open pull request
```

**See**: `CONTRIBUTING.md` in repository root.

