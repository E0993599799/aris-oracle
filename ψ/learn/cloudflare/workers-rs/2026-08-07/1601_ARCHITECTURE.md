# Cloudflare Workers Rust SDK — Architecture

## Overview

A Rust crate providing **ergonomic bindings** to the Cloudflare Workers JavaScript environment. Enables developers to write entire Workers in Rust, compiled to WebAssembly (WASM), running on Cloudflare's global network.

**Core concept**: Rust → WASM → Cloudflare Workers runtime (V8 JavaScript engine)

---

## Repository Structure

```
workers-rs/
├── worker/                 # Main worker SDK crate (primary)
│   └── src/
│       ├── lib.rs         # Entry point
│       ├── ai.rs          # Cloudflare AI bindings
│       ├── analytics_engine.rs  # Analytics Engine API
│       ├── cache.rs       # Cache API (KV-like)
│       ├── cf.rs          # Cloudflare metadata (IP, coordinates, etc.)
│       ├── context.rs     # Worker context (timers, waitUntil)
│       ├── cors.rs        # CORS helper
│       ├── crypto.rs      # Web Crypto API
│       ├── d1/            # D1 database bindings
│       ├── date.rs        # Date utilities
│       ├── delay.rs       # setTimeout equivalent
│       ├── durable.rs     # Durable Objects API
│       ├── email.rs       # Email sending (WorkersEmail)
│       ├── env.rs         # Environment variables & bindings
│       ├── error.rs       # Error types
│       ├── fetch.rs       # Fetch API + utilities
│       ├── form.rs        # Form data parsing
│       ├── headers.rs     # HTTP headers
│       ├── http.rs        # HTTP constants & types
│       ├── middleware.rs  # Middleware chains
│       ├── queue.rs       # Queue API
│       ├── r2.rs          # R2 object storage
│       ├── request.rs     # Request type
│       ├── response.rs    # Response type
│       ├── schedule.rs    # Scheduled handlers
│       ├── session.rs     # Session management
│       ├── stream.rs      # Streaming APIs
│       └── bindings/      # JS binding wrappers (generated)
│
├── worker-macros/         # Procedural macros for decorators
│   └── src/
│       ├── event.rs       # #[event(...)] macro
│       ├── durable.rs     # Durable Object macros
│       └── ... (derive macros)
│
├── worker-build/          # Build system integration (wrangler)
├── worker-codegen/        # Code generation utilities
├── worker-sys/            # Low-level JS bindings (wasm-bindgen)
├── examples/              # Reference implementations
├── templates/             # Project templates (cargo-generate)
├── test/                  # Test suites
├── benchmark/             # Performance benchmarks
├── types/                 # TypeScript type definitions
└── Cargo.toml            # Workspace manifest
```

---

## Core Components

### 1. **Main Worker Crate** (`worker/`)

Central SDK providing all high-level APIs.

**Key modules**:
- **`Request` / `Response`** — HTTP types with Cloudflare extensions
- **`Env`** — Environment configuration (bindings, secrets, variables)
- **`Context`** — Execution context (timers, waitUntil, abort signal)
- **`Cf`** — Cloudflare metadata (IP geolocation, ASN, threat scores, etc.)

**Supported Handlers**:
- `#[event(fetch)]` — HTTP requests
- `#[event(scheduled)]` — Cron-triggered execution
- `#[event(queue)]` — Queue consumers

### 2. **Durable Objects** (`durable.rs`)

Stateful compute primitive. Bindings + marshaling for RPC-style calls.

```rust
#[durable_object]
pub struct MyObject {
    state: State,
    env: Env,
}
```

### 3. **Database Support**

**D1** (SQLite):
- SQL queries via `D1Database` binding
- Parameter binding
- Result streaming

**Hyperdrive** (PostgreSQL/MySQL):
- TCP connection pooling
- Native database client

**Durable Objects** (key-value state)
- Transactional writes
- Secondary indexes

### 4. **Storage APIs**

- **R2** (object storage) — `R2Bucket` with multipart upload, streaming
- **KV** (key-value) — `KvNamespace` with TTL, metadata
- **Queue** — Message producer/consumer pattern
- **Analytics Engine** — Time-series analytics ingest

### 5. **AI & ML**

Cloudflare AI API integration (`ai.rs`):
- Text generation (LLMs)
- Image generation
- Text classification
- Embeddings

### 6. **Middleware System** (`middleware.rs`)

Composable request/response interceptors.

```rust
let middleware = logging_middleware
    .chain(cors_middleware)
    .chain(rate_limit_middleware);
```

### 7. **Type System**

- **Typed bindings** for Cloudflare services (compile-time verified)
- **Serialization** (serde for JSON)
- **Error handling** (`Result` type with `WorkerError`)
- **Async-first** (tokio-style futures)

---

## Build Pipeline

### Compilation

```
Rust source → rustc (wasm32-unknown-unknown target) → WebAssembly binary
                ↓
         wasm-bindgen (JS glue code generation)
                ↓
         worker-build (optimization, linking)
                ↓
         .wasm + .js bundle
```

### Deployment

```
wrangler.toml → npx wrangler deploy → Cloudflare API → Global edge deployment
```

---

## Macros & Code Generation

### Procedural Macros (`worker-macros/`)

- **`#[event(fetch)]`** — Entry point for HTTP requests
- **`#[event(scheduled)]`** — Scheduled/cron handler
- **`#[event(queue)]`** — Queue consumer
- **`#[durable_object]`** — Durable Object class definition

### Auto-Generated Bindings (`worker-sys/`)

Uses `wasm-bindgen` to expose JavaScript APIs:
- Fetch
- Headers
- FormData
- CryptoKey
- ScheduledEvent
- etc.

---

## Key Abstractions

### `Env` (Environment Configuration)

```rust
pub struct Env {
    js_env: JsValue,  // Underlying JS object
}
```

Provides access to:
- Bindings (D1, R2, KV, Durable Objects, etc.)
- Environment variables
- Secrets (stored in Cloudflare)

### `Request` (HTTP Request)

Extends `web-sys::Request` with:
- `req.cf()` — Cloudflare metadata
- `req.form_data()` — Form parsing
- `req.json()` — JSON parsing
- Automatic error handling

### `Response` (HTTP Response)

Builder pattern with helpers:
- `Response::ok()`, `Response::error()`
- `Response::from_bytes()`, `Response::from_stream()`
- `with_headers()`, `with_status()`
- TTL configuration for caching

### `Context` (Execution Context)

```rust
pub struct Context {
    wait_until: /* ... */,
    abort_signal: AbortSignal,
}
```

Manages:
- Background task scheduling (`waitUntil`)
- Cancellation signals (`abort_signal`)
- Request lifecycle

---

## Error Handling

**Custom error type**: `worker::Error`

- Conversion from common error types (IO, serde, async)
- Display formatting for console logs
- HTTP status code mapping for responses

```rust
pub enum Error {
    Json(serde_json::Error),
    Io(std::io::Error),
    Internal(String),
    // ...
}
```

---

## Testing Infrastructure

**Test crate** (`test/`):
- Unit tests for SDK components
- Integration tests with mock Cloudflare environment
- Benchmark suite (`benchmark/`)

**Container tests** (`test/container-echo`):
- Tests for container runtime compatibility

---

## Template System

**cargo-generate templates** (`templates/`):
- Minimal Worker (hello-world)
- HTTP routing patterns
- Durable Objects example
- D1 database example
- Full-stack example

Templates include:
- `wrangler.toml` configuration
- Project layout
- Development workflow docs

---

## Dependency Graph

**Core dependencies**:
- `wasm-bindgen` — JS interop
- `web-sys` — Web APIs
- `js-sys` — JavaScript types
- `serde` / `serde_json` — Serialization
- `chrono` — Date handling
- `uuid` — ID generation

**Workspace dependencies** (shared versions):
- `async-trait` — Async trait definitions
- `futures-*` — Async utilities
- `http` — HTTP constants

---

## Workflow

1. **Create project** from template: `cargo generate cloudflare/workers-rs`
2. **Write handler** with `#[event(fetch)]` macro
3. **Use bindings** (D1, R2, KV) via `Env`
4. **Test locally**: `npx wrangler dev`
5. **Deploy**: `npx wrangler publish` or `cargo deploy`

---

## Key Design Patterns

| Pattern | Purpose | Example |
|---------|---------|---------|
| **Macro-based entry points** | Type-safe event handlers | `#[event(fetch)]` |
| **Builder pattern** | Fluent response construction | `Response::ok().with_status(201)` |
| **Typed bindings** | Compile-time safety for Cloudflare APIs | `let db: D1Database = env.d1("DB")?` |
| **Error propagation** | `?` operator for clean error handling | `req.json().await?` |
| **Async-first** | Futures-based async with tokio conventions | `async fn handler()` |
| **Serde integration** | Automatic serialization | `serde::to_string()` |

---

## Performance Considerations

- **No runtime overhead** — Direct wasm-bindgen calls to JS
- **Memory efficient** — WASM linear memory (150KB typical worker)
- **Zero cold starts** — V8 snapshot optimization
- **Parallel execution** — Cloudflare's CPU scheduling across cores

