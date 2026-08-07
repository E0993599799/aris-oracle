# Cloudflare Workers Rust SDK Learning Index

## Source

- **Origin**: `./origin/` (symlink to `/home/marcuz/ghq/github.com/cloudflare/workers-rs`)
- **GitHub**: https://github.com/cloudflare/workers-rs
- **Project Type**: Rust framework for serverless Cloudflare Workers
- **License**: Apache 2.0

---

## Explorations

### 2026-08-07 16:01 (default: 3 agents)

**Documentation Generated:**
- [[2026-08-07/1601_ARCHITECTURE|Architecture]] — Repository structure, crate organization, build pipeline, core components
- [[2026-08-07/1601_CODE-SNIPPETS|Code Snippets]] — Handler patterns, bindings, database queries, storage APIs, error handling
- [[2026-08-07/1601_QUICK-REFERENCE|Quick Reference]] — Getting started, built-in APIs, common patterns, configuration, deployment

---

## Key Insights

### 1. **Rust → WASM → Global Edge**
Workers-rs compiles Rust to WebAssembly and deploys to Cloudflare's global network. Entire backend written in type-safe Rust, no JavaScript needed. Direct bindings to Cloudflare services (D1, R2, KV, Durable Objects, AI APIs) with compile-time verification. Zero cold starts due to V8 snapshot optimization.

### 2. **Macro-Driven Development**
Entry points use procedural macros (`#[event(fetch)]`, `#[event(scheduled)]`, `#[event(queue)]`) for clean syntax. Macros generate event listener registration automatically. Combined with Rust's type system, provides excellent compile-time safety. No runtime reflection or configuration boilerplate—everything verified at build time.

### 3. **Type-Safe Service Integration**
All Cloudflare APIs accessed through typed bindings:
- `env.d1("DB")` returns `D1Database` (compile error if binding doesn't exist)
- `env.kv("CACHE")` returns `KvNamespace`
- `env.bucket("FILES")` returns `R2Bucket`
- Serde integration for automatic JSON serialization (no manual stringify/parse)
- Error handling via `Result` type with `?` operator

---

## Architecture Overview

```
workers-rs/ (monorepo)
├── worker/                 # Main SDK (high-level APIs)
├── worker-macros/          # Procedural macros (@event, @durable_object)
├── worker-build/           # Build system integration
├── worker-sys/             # Low-level JS bindings (wasm-bindgen)
└── templates/              # cargo-generate project templates

Compilation flow:
  Rust source → wasm32-unknown-unknown → wasm-bindgen → worker-build → .wasm + .js → wrangler deploy
```

---

## Service Support Matrix

| Service | Rust Type | Pattern | Example |
|---------|-----------|---------|---------|
| **D1** | `D1Database` | Query builder | `db.prepare("SELECT...").bind(...).all().await?` |
| **R2** | `R2Bucket` | Put/Get/List | `bucket.put(key, bytes).execute().await?` |
| **KV** | `KvNamespace` | Get/Put/Delete | `kv.put_bytes(key, bytes).expiration_ttl(3600).execute().await?` |
| **Durable Objects** | `DurableObject` | RPC-style | `stub.fetch_with_str("/method").await?` |
| **AI APIs** | `Ai` | Run-based | `ai.run("@cf/mistral/...", json!({...}))?` |
| **Cache** | `Cache` | HTTP-level | Via `Response.with_header("Cache-Control", ...)`  |
| **Analytics Engine** | `Analytics` | Event ingest | `analytics.write_data_point(...).execute().await?` |
| **Queue** | `Queue` | Producer | `queue.send(message).execute().await?` |

---

## Key Technologies

- **Language**: Rust (2021 edition)
- **Compilation Target**: WebAssembly (wasm32-unknown-unknown)
- **Interop**: wasm-bindgen (Rust ↔ JavaScript)
- **Async Runtime**: Tokio-style futures (non-blocking I/O)
- **Serialization**: Serde (JSON, form data)
- **Deployment**: Wrangler CLI (Cloudflare's toolchain)

---

## Common Workflows

### Create & Deploy
```bash
cargo generate cloudflare/workers-rs
cd my-worker
npx wrangler dev          # Test locally
npx wrangler deploy       # Push to edge
```

### Add Database
```toml
# wrangler.toml
[[d1_databases]]
binding = "DB"
database_name = "production"
```

```rust
let db = env.d1("DB")?;
let users = db.prepare("SELECT * FROM users").all().await?;
```

### Add Storage
```toml
[[r2_buckets]]
binding = "FILES"
bucket_name = "my-bucket"
```

```rust
let bucket = env.bucket("FILES")?;
bucket.put("document.pdf", bytes).execute().await?;
```

---

## Notable Patterns

| Pattern | Purpose | Benefit |
|---------|---------|---------|
| **Macro-based entry points** | Event handler declaration | Type-safe, auto-registered, no boilerplate |
| **Builder pattern** | Response/Request configuration | Fluent, readable, composable |
| **Error propagation (`?`)** | Clean error handling | No if-let chains, early returns |
| **Async-first** | Non-blocking I/O | Scales to thousands of concurrent requests |
| **Typed bindings** | Service access | Compile-time verification, IDE autocomplete |
| **Serde integration** | JSON handling | Automatic serialization, no manual parsing |

---

## Performance Profile

- **Bundle size**: ~150-300 KB typical worker (Wasm + glue)
- **Cold start**: ~1ms (V8 snapshot, no JIT compile needed)
- **Memory**: Linear WASM memory (256 KB default, up to 1 GB)
- **CPU**: Cloudflare's shared CPU pool (no guarantees)
- **Concurrency**: Handled by Cloudflare runtime (can run thousands of workers)

---

## Learning Path

1. **Quickstart** — Deploy hello-world template (5 min)
2. **HTTP Handlers** — Build routing logic, JSON responses (30 min)
3. **Database** — Query D1 with bindings (45 min)
4. **Storage** — Upload/download R2 objects (30 min)
5. **Caching** — Use KV for session/cache data (20 min)
6. **Stateful Compute** — Build Durable Objects (1 hour)
7. **Scheduled Jobs** — Cron-triggered tasks (20 min)
8. **Deployment** — Production setup with wrangler (30 min)

---

## Strengths

✅ **Type safety** — Compile-time verification of Cloudflare bindings  
✅ **Zero cold starts** — Instant response to requests  
✅ **Global deployment** — Edge-first architecture  
✅ **Rich API surface** — All Cloudflare services accessible  
✅ **Developer experience** — Cargo ecosystem, IDE support, great documentation  
✅ **Backward compatible** — Existing Projects Registry friendly  

---

## Considerations

⚠️ **WASM compilation** — Takes 30-60s vs 5-10s for JS (first time)  
⚠️ **Ecosystem size** — Smaller than Node.js ecosystem (but core crates solid)  
⚠️ **Debugging** — WASM stack traces less readable than JavaScript  
⚠️ **Learning curve** — Rust ownership rules, async/await semantics  

---

## Use Cases

| Use Case | Fit | Notes |
|----------|-----|-------|
| **REST APIs** | Excellent | Type-safe routing, JSON serialization, database integration |
| **Real-time Apps** | Good | Durable Objects for persistent connections |
| **Data Processing** | Good | High CPU efficiency, easy parallel processing |
| **Static Site** | Good | Static hosting + edge computing for dynamic endpoints |
| **Machine Learning** | Fair | Inference via Cloudflare AI APIs (models hosted) |
| **WebSocket Proxies** | Limited | WebSocket support in Durable Objects only |

---

## Next Steps for Learning

1. **Quick Start**: Follow Getting Started section in QUICK-REFERENCE
2. **Deep Dive**: Study code patterns in CODE-SNIPPETS
3. **Architecture**: Review ARCHITECTURE for crate organization
4. **Build & Deploy**: Create a simple project with D1 + R2
5. **Production**: Configure environments, monitoring, CI/CD

---

## References

- **Docs**: https://docs.rs/worker
- **GitHub**: https://github.com/cloudflare/workers-rs
- **Wrangler**: https://developers.cloudflare.com/workers/wrangler/
- **Workers Platform**: https://developers.cloudflare.com/workers/
- **Rust Edition**: https://doc.rust-lang.org/edition-guide/rust-2021/

---

*Learning session: 2026-08-07 16:01 | Generated via /learn skill with 3-agent exploration*
