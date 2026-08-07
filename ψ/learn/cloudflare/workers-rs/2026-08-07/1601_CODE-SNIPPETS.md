# Cloudflare Workers Rust SDK — Code Snippets & Patterns

## 1. Basic HTTP Handler

### Entry Point with Event Macro

```rust
use worker::*;

#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: worker::Context,
) -> Result<Response> {
    // Handle request
    Response::ok("Hello, World!")
}
```

**Pattern**:
- `#[event(fetch)]` — Macro generates event listener registration
- Three parameters: Request, Env (bindings), Context (timers)
- Returns `Result<Response>` (error handling via `?` operator)
- `async fn` for async I/O (database, network)

---

## 2. Routing with Conditional Logic

```rust
#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    match req.method() {
        Method::Get => handle_get(&req, &env).await,
        Method::Post => handle_post(&mut req, &env).await,
        Method::Delete => handle_delete(&req, &env).await,
        _ => Response::error("Method Not Allowed", 405),
    }
}

async fn handle_post(
    req: &mut Request,
    env: &Env,
) -> Result<Response> {
    let body = req.text().await?;
    // Process POST body
    Response::ok(format!("Received: {}", body))
}
```

**Pattern**:
- Match on `req.method()` for routing
- Helper functions for each handler
- `req.text()`, `req.json()`, `req.form_data()` for body parsing
- Separate function signatures for `POST` (mut req) vs `GET` (immutable)

---

## 3. JSON Request/Response

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct CreateRequest {
    name: String,
    email: String,
    age: u32,
}

#[derive(Serialize)]
struct CreateResponse {
    id: String,
    created_at: String,
}

#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    if req.method() == Method::Post {
        let input: CreateRequest = req.json().await?;
        
        let response = CreateResponse {
            id: format!("user-{}", uuid::Uuid::new_v4()),
            created_at: chrono::Utc::now().to_rfc3339(),
        };
        
        return Response::from_json(&response);
    }
    
    Response::error("Bad Request", 400)
}
```

**Pattern**:
- `#[derive(Serialize, Deserialize)]` for serde integration
- `req.json().await?` — automatic deserialization with error propagation
- `Response::from_json()` — automatic serialization
- No manual stringify/parse needed

---

## 4. Environment Bindings & Secrets

```rust
#[event(fetch)]
pub async fn main(
    req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    // Get D1 database binding
    let db = env.d1("DB")?;
    
    // Get KV namespace binding
    let kv = env.kv("CACHE")?;
    
    // Get environment variable
    let api_key = env.var("API_KEY")?;
    
    // Get secret (from Cloudflare dashboard)
    let db_password = env.secret("DB_PASSWORD")?;
    
    Response::ok("Configured")
}
```

**Pattern**:
- `env.d1("name")` — typed D1 database
- `env.kv("name")` — typed KV namespace
- `env.var("name")` / `env.secret("name")` — string values
- All return `Result` with error handling
- Bindings defined in `wrangler.toml`

---

## 5. D1 Database Queries

```rust
#[derive(Serialize)]
struct User {
    id: i64,
    name: String,
    email: String,
}

#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    let db = env.d1("DB")?;
    
    // SELECT query
    let query = db
        .prepare("SELECT id, name, email FROM users WHERE email = ?")
        .bind(&["user@example.com"])?;
    
    let results: Vec<User> = query.all().await?
        .results::<User>()?;
    
    // INSERT query
    db.prepare("INSERT INTO users (name, email) VALUES (?, ?)")
        .bind(&["Alice", "alice@example.com"])?
        .run()
        .await?;
    
    Response::from_json(&results)
}
```

**Pattern**:
- `db.prepare("SQL")` — statement preparation
- `.bind()` — parameter binding (positional `?`)
- `.all()` — fetch all rows
- `.results::<Type>()` — type conversion via serde
- `.run()` — execute without returning rows

---

## 6. R2 Object Storage

```rust
#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    let bucket = env.bucket("FILES")?;
    
    // Upload object
    let body = req.bytes().await?;
    bucket.put("document.pdf", body).execute().await?;
    
    // Get object
    if let Some(object) = bucket.get("document.pdf").execute().await? {
        let data = object.body().bytes().await?;
        return Response::from_bytes(data)
            .with_header("Content-Type", "application/pdf");
    }
    
    // List objects
    let objects = bucket.list().execute().await?;
    for obj in objects.objects() {
        console_log!("Object: {}", obj.key());
    }
    
    Response::ok("Done")
}
```

**Pattern**:
- `env.bucket("name")` — typed R2 bucket
- `.put(key, body).execute()` — async upload
- `.get(key).execute()` — fetch with streaming support
- `.list()` — enumerate keys (paginated)
- Response headers for content type

---

## 7. KV Namespace (Key-Value Storage)

```rust
#[event(fetch)]
pub async fn main(
    req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    let kv = env.kv("CACHE")?;
    
    // Get cached value
    if let Some(cached) = kv.get("user:123").await? {
        return Response::from_bytes(cached);
    }
    
    // Compute and cache result (with TTL)
    let value = compute_expensive_value().await?;
    kv.put_bytes("user:123", &value)?
        .expiration_ttl(3600)  // 1 hour
        .execute()
        .await?;
    
    Response::from_bytes(value)
}
```

**Pattern**:
- `kv.get(key)` — returns `Option<Vec<u8>>`
- `kv.put_bytes(key, bytes)` — builder pattern for options
- `.expiration_ttl(seconds)` — auto-expire after TTL
- `.execute()` — async persistence

---

## 8. Durable Objects (Stateful Compute)

### Server Side

```rust
use worker::{durable_object, Env, Request, Response, Result};

#[durable_object]
pub struct Counter {
    state: State,
    env: Env,
}

#[durable_object]
impl Counter {
    pub fn new(state: State, env: Env) -> Self {
        Self { state, env }
    }
    
    pub async fn increment(&mut self) -> Result<i32> {
        let count: i32 = self.state.get("count").await?.unwrap_or(0);
        let new_count = count + 1;
        self.state.put("count", new_count).await?;
        Ok(new_count)
    }
    
    pub async fn handle(
        &mut self,
        req: Request,
    ) -> Result<Response> {
        match req.path().as_str() {
            "/increment" => {
                let count = self.increment().await?;
                Response::from_json(&serde_json::json!({ "count": count }))
            }
            _ => Response::error("Not Found", 404),
        }
    }
}
```

### Client Side (from another Worker)

```rust
#[event(fetch)]
pub async fn main(
    req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    let stub = env.durable_object("Counter")?
        .id_from_name("my-counter")?
        .get_stub()?;
    
    let resp = stub
        .fetch_with_str("/increment")
        .await?;
    
    Response::from_json(&resp.json().await?)
}
```

**Pattern**:
- `#[durable_object]` macro for class definition
- `state.get()` / `state.put()` for transactional storage
- `handle()` method for RPC-style calls
- Compile-time type safety for method calls

---

## 9. Scheduled Handlers (Cron)

```rust
#[event(scheduled)]
pub async fn scheduled(
    _event: ScheduledEvent,
    env: Env,
    _ctx: Context,
) -> Result<()> {
    let db = env.d1("DB")?;
    
    console_log!("Running scheduled job at {:?}", _event.cron());
    
    // Clean up old records
    db.prepare("DELETE FROM logs WHERE created_at < datetime('now', '-7 days')")
        .run()
        .await?;
    
    Ok(())
}
```

**Pattern**:
- `#[event(scheduled)]` for cron handlers
- `_event.cron()` — cron expression that triggered
- Configured via `wrangler.toml` `[[triggers.crons]]`
- Returns `Result<()>` (no HTTP response)

---

## 10. CORS & Middleware

```rust
#[event(fetch)]
pub async fn main(
    req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    // CORS preflight
    if req.method() == Method::Options {
        return Ok(Response::empty()?
            .with_header("Access-Control-Allow-Origin", "*")
            .with_header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE")
            .with_header("Access-Control-Allow-Headers", "Content-Type")
        );
    }
    
    let mut response = Response::ok("Hello")?;
    response.headers_mut().set(
        "Access-Control-Allow-Origin",
        "*"
    )?;
    
    Ok(response)
}
```

**Pattern**:
- Explicit CORS handling in handler
- `req.method() == Method::Options` for preflight
- `response.with_header()` for setting headers
- Builder pattern for response configuration

---

## 11. Form Data Parsing

```rust
use worker::FormEntry;

#[event(fetch)]
pub async fn main(
    mut req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    if req.method() != Method::Post {
        return Response::error("Method Not Allowed", 405);
    }
    
    let form = req.form_data().await?;
    
    // Get text field
    if let Some(FormEntry::Field(name)) = form.get("name") {
        console_log!("Name: {}", name);
    }
    
    // Get file upload
    if let Some(FormEntry::File(file)) = form.get("file") {
        let bytes = file.bytes().await?;
        console_log!("File size: {} bytes", bytes.len());
    }
    
    Response::ok("Form processed")
}
```

**Pattern**:
- `req.form_data()` — async form parsing
- Pattern match on `FormEntry` enum
- `FormEntry::Field` for text inputs
- `FormEntry::File` for file uploads with streaming

---

## 12. Error Handling & Logging

```rust
use worker::{console_log, console_error};

#[event(fetch)]
pub async fn main(
    req: Request,
    env: Env,
    _ctx: Context,
) -> Result<Response> {
    match dangerous_operation(&req, &env).await {
        Ok(result) => {
            console_log!("Success: {:?}", result);
            Response::from_json(&result)
        }
        Err(e) => {
            console_error!("Error occurred: {}", e);
            Response::error(&format!("Internal error: {}", e), 500)
        }
    }
}

async fn dangerous_operation(
    req: &Request,
    env: &Env,
) -> Result<String> {
    let value = env.var("REQUIRED_VAR")
        .map_err(|_| worker::Error::Internal("Missing config".into()))?;
    
    Ok(value)
}
```

**Pattern**:
- `console_log!()` / `console_error!()` for debugging
- Custom error messages via `map_err()`
- Error propagation with `?` operator
- HTTP status codes for error responses (4xx, 5xx)

---

## Key Patterns Summary

| Pattern | Use Case |
|---------|----------|
| **Macro-based entry points** | Type-safe event declaration |
| **Environment bindings** | Type-safe Cloudflare service access |
| **Builder pattern** | Fluent configuration (Response, Request options) |
| **Serde integration** | JSON serialization without manual parsing |
| **Result type** | Error propagation with `?` operator |
| **Async-first** | Non-blocking I/O with `.await` |
| **Form data parsing** | multipart/form-data with streaming |
| **CORS headers** | Browser compatibility |
| **Durable Objects** | Persistent state + RPC |
| **Logging** | Debug output to Cloudflare logs |

