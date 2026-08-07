# Cloudflare Workers Rust SDK — Quick Reference

## What Is This?

**Workers-rs** is a Rust framework for building serverless applications on Cloudflare Workers. Write your entire backend in Rust, compile to WebAssembly, and deploy globally without managing servers. Access all Cloudflare services (D1, R2, KV, Durable Objects, AI APIs) with type-safe Rust bindings.

---

## Quick Start

### 1. Create a New Project

```bash
# Install cargo-generate first
cargo install cargo-generate

# Create from template
cargo generate cloudflare/workers-rs

# Answer prompts:
# - Project name: my-worker
# - Enable panic=unwind? (y/n, recommended: y)
```

### 2. Project Structure

```
my-worker/
├── src/
│   └── lib.rs           # Worker code
├── Cargo.toml           # Rust dependencies
├── wrangler.toml        # Deployment config
└── package.json         # Node dependencies (wrangler)
```

### 3. Basic Handler

```rust
use worker::*;

#[event(fetch)]
pub async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    Response::ok("Hello from Rust on Cloudflare Workers!")
}
```

### 4. Run Locally

```bash
# Install dependencies
npm install

# Start dev server
npx wrangler dev

# Test
curl http://localhost:8787
```

### 5. Deploy to Cloudflare

```bash
# Authenticate
npx wrangler login

# Deploy
npx wrangler deploy

# View logs
npx wrangler tail
```

---

## Key Concepts

### Event Handlers

Three types of events:

```rust
// HTTP request (default)
#[event(fetch)]
pub async fn fetch_handler(req: Request, env: Env, ctx: Context) -> Result<Response> { }

// Scheduled/cron
#[event(scheduled)]
pub async fn scheduled_handler(event: ScheduledEvent, env: Env, ctx: Context) -> Result<()> { }

// Queue consumer
#[event(queue)]
pub async fn queue_handler(batch: MessageBatch, env: Env, ctx: Context) -> Result<()> { }
```

### Environment Bindings

Define in `wrangler.toml`:

```toml
[env.production]
vars = { ENVIRONMENT = "production" }

[[d1_databases]]
binding = "DB"
database_name = "my_database"

[[kv_namespaces]]
binding = "CACHE"
id = "abc123..."

[[r2_buckets]]
binding = "FILES"
bucket_name = "my-bucket"

[durable_objects]
bindings = [{ name = "COUNTER", class_name = "Counter" }]
```

Access in code:

```rust
let db = env.d1("DB")?;
let kv = env.kv("CACHE")?;
let bucket = env.bucket("FILES")?;
let secret = env.secret("API_KEY")?;
```

---

## Built-in APIs

### HTTP (Fetch)

```rust
// GET request
let response = worker::fetch_raw(request).await?;

// POST with JSON
let client = reqwest::Client::new();
let json = serde_json::json!({ "key": "value" });
client.post("https://api.example.com")
    .json(&json)
    .send()
    .await?;
```

### Database (D1 — SQLite)

```rust
let db = env.d1("DB")?;

// SELECT
let results: Vec<User> = db
    .prepare("SELECT * FROM users WHERE id = ?")
    .bind(&[id])?
    .all()
    .await?
    .results::<User>()?;

// INSERT
db.prepare("INSERT INTO users (name) VALUES (?)")
    .bind(&["Alice"])?
    .run()
    .await?;

// Transaction
db.batch(vec![
    db.prepare("INSERT INTO users ...").bind(&[...])?,
    db.prepare("UPDATE logs ...").bind(&[...])?,
])
.await?;
```

### Object Storage (R2)

```rust
let bucket = env.bucket("FILES")?;

// Upload
bucket.put("document.pdf", bytes).execute().await?;

// Download
let obj = bucket.get("document.pdf").execute().await?;
let data = obj.body().bytes().await?;

// List
let objects = bucket.list().execute().await?;
for obj in objects.objects() {
    println!("{}", obj.key());
}

// Delete
bucket.delete("document.pdf").execute().await?;
```

### Key-Value Storage (KV)

```rust
let kv = env.kv("CACHE")?;

// Get
if let Some(value) = kv.get("user:123").await? {
    let json: User = serde_json::from_slice(&value)?;
}

// Set with TTL
kv.put_bytes("session:abc", data)?
    .expiration_ttl(3600)  // 1 hour
    .execute()
    .await?;

// Delete
kv.delete("user:123").await?;

// List keys
let keys = kv.list().execute().await?;
```

### Durable Objects (Stateful Compute)

Server:

```rust
#[durable_object]
pub struct Counter {
    state: State,
    env: Env,
}

#[durable_object]
impl Counter {
    pub fn new(state: State, env: Env) -> Self { Self { state, env } }
    
    pub async fn increment(&mut self) -> Result<i32> {
        let count: i32 = self.state.get("count").await?.unwrap_or(0);
        self.state.put("count", count + 1).await?;
        Ok(count + 1)
    }
    
    pub async fn handle(&mut self, req: Request) -> Result<Response> {
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

Client (from another Worker):

```rust
let stub = env.durable_object("Counter")?
    .id_from_name("my-counter")?
    .get_stub()?;

let response = stub.fetch_with_str("/increment").await?;
let json: serde_json::Value = response.json().await?;
```

### AI APIs

```rust
let ai = env.ai();

// Text generation (LLM)
let response = ai.run("@cf/mistral/mistral-7b-instruct-v0.1", json!({
    "prompt": "What is 2+2?"
}))?;

// Image generation
let image = ai.run("@cf/stabilityai/stable-diffusion-xl-lightning", json!({
    "prompt": "A mountain landscape"
}))?;

// Text classification
let classification = ai.run("@cf/huggingface/distilbert-sst-2-int8", json!({
    "text": "This is amazing!"
}))?;
```

### Cloudflare Metadata (CF)

```rust
let cf = req.cf()?;

// Geolocation
if let Some(coords) = cf.coordinates() {
    println!("Latitude: {}, Longitude: {}", coords.latitude(), coords.longitude());
}

// Country/Region
println!("Country: {}", cf.country());
println!("Region: {}", cf.region());
println!("Postal Code: {}", cf.postal_code());

// Network info
println!("ASN: {}", cf.asn());
println!("ISP: {}", cf.isp());
println!("Threat Score: {}", cf.threat_level());

// TLS info
if let Some(cipher) = cf.tlsCipher() {
    println!("Cipher: {}", cipher);
}
```

---

## Common Patterns

### JSON Serialization

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize)]
struct User {
    id: i64,
    name: String,
    email: String,
}

// Parse JSON from request
let user: User = req.json().await?;

// Send JSON response
Response::from_json(&user)
```

### Error Handling

```rust
// Propagate errors with `?`
let value = env.var("KEY")?;

// Custom error messages
let value = env.var("KEY")
    .map_err(|_| Error::Internal("Missing KEY env var".into()))?;

// Error response
Response::error("Bad Request", 400)
Response::error("Server Error", 500)
```

### Logging

```rust
use worker::{console_log, console_error, console_warn};

console_log!("Processing request: {}", req.path());
console_warn!("This is a warning");
console_error!("An error occurred: {}", error);
```

### CORS

```rust
// Handle preflight
if req.method() == Method::Options {
    return Ok(Response::empty()?
        .with_header("Access-Control-Allow-Origin", "*")
        .with_header("Access-Control-Allow-Methods", "GET, POST, DELETE")
        .with_header("Access-Control-Allow-Headers", "Content-Type")
    );
}

// Add CORS to response
Response::ok("data")?
    .with_header("Access-Control-Allow-Origin", "*")
```

### Form Data

```rust
use worker::FormEntry;

let form = req.form_data().await?;

// Text field
if let Some(FormEntry::Field(name)) = form.get("name") {
    println!("Name: {}", name);
}

// File upload
if let Some(FormEntry::File(file)) = form.get("attachment") {
    let bytes = file.bytes().await?;
    bucket.put("upload.bin", bytes).execute().await?;
}
```

---

## Configuration (wrangler.toml)

```toml
name = "my-worker"
type = "javascript"
account_id = "your-account-id"
workers_dev = true
route = "example.com/*"
zone_id = "your-zone-id"

[env.production]
vars = { ENVIRONMENT = "production" }

[[d1_databases]]
binding = "DB"
database_name = "production"

[[kv_namespaces]]
binding = "CACHE"
id = "kv-id"

[[r2_buckets]]
binding = "FILES"
bucket_name = "my-bucket"

[[triggers.crons]]
crons = ["0 0 * * *"]  # Daily at midnight
```

---

## Deployment

### Environments

```bash
# Deploy to development
npx wrangler deploy --env development

# Deploy to production
npx wrangler deploy --env production

# View specific environment
npx wrangler deployments list --env production
```

### Custom Domains

```toml
route = "api.example.com/*"
zone_id = "your-zone-id"
```

### Triggers

**Cron (scheduled)**:
```toml
[[triggers.crons]]
crons = ["0 0 * * MON"]  # Weekly Monday
```

**Queue**:
```toml
[[queues.consumers]]
queue = "my-queue"
max_batch_size = 10
max_batch_timeout = 30
```

---

## Testing

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_basic() {
        assert_eq!(2 + 2, 4);
    }
}
```

Run tests:

```bash
cargo test
```

---

## Performance Tips

1. **Minimize dependencies** — Smaller WASM bundle = faster cold starts
2. **Use connection pooling** — Reuse DB connections across requests
3. **Cache aggressively** — Use KV for frequently accessed data
4. **Stream large files** — Use `Response::from_stream()` for R2 downloads
5. **Batch database queries** — Use `db.batch()` for multiple operations
6. **Lazy initialization** — Initialize clients only when needed

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| `wasm32-unknown-unknown target not found` | `rustup target add wasm32-unknown-unknown` |
| `panic in worker` | Enable `panic = "unwind"` in Cargo.toml, set panic recovery hook |
| `Module too large` | Use `wasm-opt` for optimization, remove unused dependencies |
| `Connection timeout` | Check Cloudflare firewall rules, ensure `CORS` headers correct |
| `Out of memory` | Stream large responses, limit batch sizes |

---

## Documentation Links

- **Docs**: https://docs.rs/worker
- **GitHub**: https://github.com/cloudflare/workers-rs
- **Wrangler CLI**: https://developers.cloudflare.com/workers/wrangler/
- **Workers Docs**: https://developers.cloudflare.com/workers/
- **D1 Database**: https://developers.cloudflare.com/d1/
- **R2 Storage**: https://developers.cloudflare.com/r2/
- **Durable Objects**: https://developers.cloudflare.com/durable-objects/

---

## Learning Path

1. **Start**: Create hello-world with template
2. **Fetch Data**: Add D1 database query
3. **Store Files**: Integrate R2 object storage
4. **Cache Results**: Use KV for session storage
5. **Advanced**: Build Durable Objects for stateful compute
6. **Deploy**: Push to production with wrangler

---

