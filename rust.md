# Rust — Complete Development Reference

Covers: lifecycle roles, design patterns, DevOps/containers, security, best practices.

---

## Table of Contents

1. [Lifecycle Roles](#1-lifecycle-roles)
2. [Design Patterns](#2-design-patterns)
3. [Error Handling](#3-error-handling)
4. [Async & Concurrency](#4-async--concurrency)
5. [DevOps & Containers](#5-devops--containers)
6. [Security](#6-security)
7. [Things to Avoid](#7-things-to-avoid)

---

## 1. Lifecycle Roles

### Architect

**Project structure** — packages (`Cargo.toml`), crates (compilation units), modules (logical grouping).

```
my-project/
├── Cargo.toml              # workspace root
├── crates/
│   ├── core/               # library crate
│   │   ├── Cargo.toml
│   │   └── src/lib.rs
│   ├── cli/                # binary crate
│   │   └── src/main.rs
│   └── server/             # binary crate
│       └── src/main.rs
└── tests/                  # integration tests
```

```toml
# Workspace Cargo.toml
[workspace]
resolver = "2"
members = ["crates/core", "crates/cli", "crates/server"]

[workspace.dependencies]
serde = { version = "1", features = ["derive"] }
tokio = { version = "1", features = ["full"] }
```

**When to split**: extract a library crate when logic is reusable or testable independent of the binary. Use a workspace when multiple binaries share a core library.

**API design** (Rust API Guidelines):
- Eagerly implement `Debug`, `Clone`, `PartialEq`, `Default`, `Send`, `Sync`
- Naming: `as_` (cheap ref→ref), `to_` (expensive convert), `into_` (ownership transfer)
- Accept `impl AsRef<str>` over `&str` when flexibility matters
- Use newtypes for type-safe distinctions

**Trait-based abstractions**:

```rust
pub trait Storage: Send + Sync {
    type Error: std::error::Error + Send + Sync + 'static;
    fn get(&self, key: &str) -> Result<Option<Vec<u8>>, Self::Error>;
    fn put(&self, key: &str, value: &[u8]) -> Result<(), Self::Error>;
}

// Static dispatch (zero-cost): fn process<S: Storage>(s: &S)
// Dynamic dispatch (flexible): fn process(s: &dyn Storage<Error = anyhow::Error>)
```

### Developer

**Cargo workflow**:

```bash
cargo new my-app             # binary
cargo new my-lib --lib       # library
cargo build --release        # optimized build
cargo run -- --flag arg      # build + run
cargo check                  # type-check only (fast)
cargo doc --open             # generate docs
cargo tree                   # dependency tree
cargo build --timings        # compilation profiling
```

**Core crate ecosystem**:

| Crate | Purpose |
|-------|---------|
| serde + serde_json/toml | Serialization (`#[derive(Serialize, Deserialize)]`) |
| tokio | Async runtime, channels, timers |
| clap | CLI parsing (`#[derive(Parser)]`) |
| reqwest | HTTP client |
| sqlx | Async SQL with compile-time query checking |
| anyhow / thiserror | Error handling (app / library) |
| tracing | Structured logging with spans |
| rayon | Data parallelism (`.par_iter()`) |

### Tester

**Unit tests** — alongside code in `#[cfg(test)]`:

```rust
pub fn add(a: i32, b: i32) -> i32 { a + b }

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn test_add() { assert_eq!(add(2, 3), 5); }

    #[test]
    #[should_panic(expected = "overflow")]
    fn test_overflow() { add(i32::MAX, 1); }

    #[test]
    fn test_fallible() -> Result<(), Box<dyn std::error::Error>> {
        let val: i32 = "42".parse()?;
        assert_eq!(val, 42);
        Ok(())
    }
}
```

**Integration tests** — `tests/` directory, each file is a separate crate.

**Doc tests** — embedded in `///` comments, run with `cargo test --doc`.

**Property-based testing** (proptest):

```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn roundtrip(input in "\\PC*") {
        let encoded = encode(&input);
        let decoded = decode(&encoded).unwrap();
        prop_assert_eq!(input, decoded);
    }
}
```

**Fuzzing**: `cargo install cargo-fuzz && cargo fuzz run target`

**Benchmarking** (criterion):

```rust
use criterion::{black_box, criterion_group, criterion_main, Criterion};

fn bench_sort(c: &mut Criterion) {
    let data: Vec<i32> = (0..1000).rev().collect();
    c.bench_function("sort_1000", |b| {
        b.iter(|| { let mut v = black_box(data.clone()); v.sort(); v })
    });
}
criterion_group!(benches, bench_sort);
criterion_main!(benches);
```

### Reviewer

**Clippy**:

```bash
cargo clippy -- -D warnings              # CI: fail on warnings
cargo clippy -- -W clippy::pedantic      # stricter
cargo clippy -- -W clippy::unwrap_used   # flag all .unwrap()
```

```rust
#![warn(clippy::all, clippy::pedantic)]
#![deny(clippy::unwrap_used)]
```

**Unsafe audit**: `cargo geiger` (count unsafe per dep), `cargo +nightly miri test` (detect UB)

**Code smells**:

| Smell | Fix |
|-------|-----|
| `.unwrap()` in lib code | `?` with proper error types |
| `String` where `&str` works | Accept `impl AsRef<str>` |
| `clone()` to appease borrow checker | Restructure ownership or `Cow` |
| `pub` on everything | Default private, expose minimal API |
| Stringly-typed APIs | Use enums or newtypes |

### Release Engineer

```bash
cargo install cargo-release
cargo release patch --execute  # 0.1.0 → 0.1.1, commit, tag, publish, push
```

**Feature flags**:

```toml
[features]
default = ["json"]
json = ["dep:serde_json"]
tls = ["reqwest/rustls-tls"]
```

```rust
#[cfg(feature = "json")]
pub mod json;

#[cfg(all(feature = "tls", not(target_arch = "wasm32")))]
fn setup_tls() { /* ... */ }
```

### Operator

**Systemd unit**:

```ini
[Service]
Type=simple
ExecStart=/usr/local/bin/myapp --config /etc/myapp/config.toml
Restart=on-failure
Environment=RUST_LOG=info,myapp=debug
Environment=RUST_BACKTRACE=1
NoNewPrivileges=true
ProtectSystem=strict
```

**Tracing setup**:

```rust
use tracing_subscriber::{fmt, EnvFilter};

fn init_tracing() {
    tracing_subscriber::fmt()
        .with_env_filter(EnvFilter::try_from_default_env()
            .unwrap_or_else(|_| EnvFilter::new("info,myapp=debug")))
        .json()
        .init();
}
```

**Crash handling**: `human-panic` for CLIs, custom panic hooks for services.

---

## 2. Design Patterns

### Creational

**Builder** — step-by-step construction with defaults:

```rust
struct ServerBuilder {
    host: String,
    port: u16,
    max_conn: usize,
}

impl ServerBuilder {
    fn new(host: impl Into<String>, port: u16) -> Self {
        Self { host: host.into(), port, max_conn: 100 }
    }
    fn max_conn(mut self, n: usize) -> Self { self.max_conn = n; self }
    fn build(self) -> Server {
        Server { host: self.host, port: self.port, max_conn: self.max_conn }
    }
}
```

**Newtype** — zero-cost type safety:

```rust
struct UserId(u64);
struct OrderId(u64);
fn get_user(id: UserId) -> String { format!("user_{}", id.0) }
// get_user(OrderId(1)); — compile error
```

**Typestate** — invalid state transitions are compile errors:

```rust
struct Locked;
struct Unlocked;

struct Door<State> { _state: std::marker::PhantomData<State> }

impl Door<Locked> {
    fn unlock(self) -> Door<Unlocked> { Door { _state: std::marker::PhantomData } }
}
impl Door<Unlocked> {
    fn open(&self) { println!("opened"); }
}
// Door::<Locked>::new().open(); — compile error
```

### Structural

**Adapter** — implement target trait on a wrapper:

```rust
trait Logger { fn log(&self, msg: &str); }

struct SyslogAdapter { client: SyslogClient, facility: u8 }
impl Logger for SyslogAdapter {
    fn log(&self, msg: &str) { self.client.send(self.facility, msg); }
}
```

**Decorator** — wrap trait to layer behavior (Tower middleware pattern):

```rust
struct LoggingNotifier<N: Notifier> { inner: N }
impl<N: Notifier> Notifier for LoggingNotifier<N> {
    fn send(&self, msg: &str) {
        println!("[LOG] {msg}");
        self.inner.send(msg);
    }
}
```

### Behavioral

**Strategy** — trait objects (open set) vs enums (closed set):

```rust
// Open set — callers provide impls
struct Pipeline { compressor: Box<dyn Compressor> }

// Closed set — you own all variants, exhaustive match
enum Compression { None, Gzip { level: u32 }, Zstd { level: u32 } }
```

**Command** — operations as objects for undo/redo/queuing.

**Observer** — prefer channels (`tokio::sync::broadcast`) over callback registration in async code.

### Rust-Specific

**RAII** — `Drop` for automatic cleanup (files, connections, temp resources).

**Interior Mutability**:
- `Cell`/`RefCell` — single-threaded
- `Mutex`/`RwLock`/atomics — multi-threaded
- `OnceLock` — lazy init, read-many

**Cow** — clone-on-write, avoid allocations on read path:

```rust
fn normalize(name: &str) -> Cow<'_, str> {
    if name.contains(' ') { Cow::Owned(name.trim().replace("  ", " ")) }
    else { Cow::Borrowed(name) }
}
```

**Phantom Types** — zero-sized type parameters for compile-time safety:

```rust
struct Meters;
struct Seconds;
struct Quantity<Unit> { value: f64, _unit: PhantomData<Unit> }
// speed(Quantity<Meters>, Quantity<Seconds>) — can't mix up
```

### Decision Table

| Decision | Choose | When |
|----------|--------|------|
| Error type | `thiserror` | Library, callers match variants |
| Error type | `anyhow` | Application, errors are logged |
| Strategy | Enum match | Closed set you own |
| Strategy | `dyn Trait` | Open set, plugin extensibility |
| Shared state | `Arc<Mutex<T>>` | Simple, few contention points |
| Shared state | Channels/Actor | Complex state, many producers |
| Parallelism | `rayon` | CPU-bound, embarrassingly parallel |
| Parallelism | `tokio` tasks | IO-bound, async |
| Clone avoidance | `Cow<'_, T>` | Read-mostly, occasional mutation |
| Type safety | Newtype | Prevent mixing semantically different values |
| State machine | Typestate | Invalid transitions must be compile errors |

---

## 3. Error Handling

**thiserror** — typed errors for libraries:

```rust
#[derive(Debug, thiserror::Error)]
enum ApiError {
    #[error("network: {0}")]
    Network(#[from] reqwest::Error),
    #[error("not found: {resource} id={id}")]
    NotFound { resource: &'static str, id: u64 },
}
```

**anyhow** — opaque errors for applications:

```rust
use anyhow::{Context, Result};

fn load_config(path: &str) -> Result<Config> {
    let text = std::fs::read_to_string(path)
        .with_context(|| format!("failed to read {path}"))?;
    let config: Config = toml::from_str(&text).context("invalid TOML")?;
    Ok(config)
}
```

**Result combinators**:

```rust
fn parse_port(s: &str) -> Option<u16> {
    s.strip_prefix(":").and_then(|p| p.parse().ok()).filter(|&p| p > 0)
}
```

---

## 4. Async & Concurrency

**Async trait** (Rust 1.75+ — no crate needed):

```rust
trait Repository {
    async fn find_by_id(&self, id: u64) -> Option<Record>;
}
```

**Tower middleware** — composable `Service` trait for axum/tonic/hyper.

**Stream processing**:

```rust
let mut stream = ReceiverStream::new(rx)
    .filter(|e| e.starts_with("event"))
    .map(|e| e.to_uppercase());
while let Some(event) = stream.next().await { /* ... */ }
```

**Arc + Mutex** — shared mutable state across threads:

```rust
let counter = Arc::new(Mutex::new(0u64));
let c = Arc::clone(&counter);
thread::spawn(move || { *c.lock().unwrap() += 1; });
```

**Channels** — message passing, actor pattern:

```rust
enum Msg { Increment(u64), GetTotal(mpsc::Sender<u64>), Shutdown }
// Actor thread owns state, responds to messages via match
```

**Rayon** — parallel iterators:

```rust
let sum: u64 = data.par_iter().filter(|&&x| x % 2 == 0).map(|&x| x * x).sum();
```

**Actor model** — private state + `mpsc` + `oneshot` reply channels.

---

## 5. DevOps & Containers

### Docker Multi-Stage Build

```dockerfile
FROM rust:1.82-bookworm AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim
RUN apt-get update && apt-get install -y --no-install-recommends ca-certificates && rm -rf /var/lib/apt/lists/*
COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
RUN useradd -r -s /bin/false appuser
USER appuser
ENTRYPOINT ["myapp"]
```

**Scratch image** (static musl):

```dockerfile
FROM rust:1.82-bookworm AS builder
RUN rustup target add x86_64-unknown-linux-musl && apt-get update && apt-get install -y musl-tools
WORKDIR /app
COPY . .
RUN cargo build --release --target x86_64-unknown-linux-musl

FROM scratch
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/myapp /myapp
ENTRYPOINT ["/myapp"]
```

**Dependency caching** — use `cargo-chef`:

```dockerfile
FROM rust:1.82-bookworm AS chef
RUN cargo install cargo-chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder
COPY --from=planner /app/recipe.json recipe.json
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release
```

**Image sizes**: scratch 5-20MB, alpine 15-40MB, distroless 30-50MB, debian-slim 80-120MB.

### Binary Optimization

```toml
[profile.release]
opt-level = "z"       # optimize for size
lto = true            # link-time optimization
codegen-units = 1     # single codegen unit
panic = "abort"       # no unwinding
strip = true          # strip symbols
```

### Cross-Compilation

```bash
rustup target add x86_64-unknown-linux-musl aarch64-unknown-linux-gnu
cargo build --release --target x86_64-unknown-linux-musl

# Or use cross for Docker-based cross-compilation
cargo install cross
cross build --release --target aarch64-unknown-linux-gnu
```

**musl vs glibc**: musl = static, self-contained, scratch-compatible. glibc = dynamic, broader compat, full NSS/DNS support.

### CI/CD Pipeline

```yaml
jobs:
  check:
    steps:
      - uses: dtolnay/rust-toolchain@stable
        with: { components: clippy, rustfmt }
      - uses: Swatinem/rust-cache@v2
      - run: cargo fmt --all -- --check
      - run: cargo clippy --all-targets -- -D warnings
      - run: cargo test --all-features
      - run: cargo install cargo-audit && cargo audit
      - run: cargo install cargo-deny && cargo deny check
```

### Observability

**Tracing + OpenTelemetry**:

```rust
use tracing::{info, instrument};

#[instrument(skip(db_pool))]
async fn handle_request(user_id: u64, db_pool: &Pool) -> Result<Response, AppError> {
    info!(user_id, "processing request");
    // ...
}
```

**Prometheus metrics**:

```rust
use metrics::{counter, histogram};
counter!("http_requests_total", "method" => "GET").increment(1);
histogram!("http_request_duration_seconds").record(elapsed.as_secs_f64());
```

### Graceful Shutdown

```rust
use tokio::signal;

async fn shutdown_signal() {
    let ctrl_c = signal::ctrl_c();
    let mut sigterm = signal::unix::signal(signal::unix::SignalKind::terminate()).unwrap();
    tokio::select! {
        _ = ctrl_c => {},
        _ = sigterm.recv() => {},
    }
}

// axum:
axum::serve(listener, app).with_graceful_shutdown(shutdown_signal()).await?;
```

### Configuration (12-Factor)

```rust
use config::{Config, Environment, File};

let cfg = Config::builder()
    .add_source(File::with_name("config/default"))
    .add_source(File::with_name(&format!("config/{run_mode}")).required(false))
    .add_source(Environment::with_prefix("APP").separator("__"))
    .build()?
    .try_deserialize::<AppConfig>()?;
// APP_DATABASE__URL=postgres://... overrides database.url
```

### Kubernetes

```yaml
livenessProbe:
  httpGet: { path: /healthz, port: http }
readinessProbe:
  httpGet: { path: /readyz, port: http }
resources:
  requests: { cpu: 100m, memory: 64Mi }
  limits:   { cpu: 500m, memory: 128Mi }
lifecycle:
  preStop:
    exec: { command: ["/bin/sh", "-c", "sleep 5"] }
```

Rust has no GC — memory is predictable. Set limits close to requests.

---

## 6. Security

### Memory Safety Summary

| Rust Prevents | Does NOT Prevent |
|---------------|------------------|
| Buffer overflows | Logic bugs |
| Use-after-free | Deadlocks |
| Data races | Resource exhaustion |
| Null derefs (`Option`) | Integer overflow (release) |
| Double free | Panics |

### Make Invalid States Unrepresentable

```rust
// BAD
struct Config { ssl: bool, ssl_cert: Option<String> }

// GOOD
enum Security { Plaintext, Tls { cert_path: PathBuf } }
struct Config { security: Security }
```

### Unsafe Code

- Minimize surface area, wrap in safe API
- Every `unsafe` block gets `// SAFETY:` comment
- Use `#[deny(unsafe_op_in_unsafe_fn)]`
- Audit with `cargo +nightly miri test` and `cargo geiger`

### Dependency Security

```bash
cargo audit                  # check RustSec advisories
cargo deny check all         # license, bans, advisories, sources
```

```toml
# deny.toml
[bans]
deny = [
    { name = "openssl" },     # prefer rustls
    { name = "openssl-sys" },
]
[sources]
unknown-registry = "deny"
unknown-git = "deny"
```

### Cryptography — Recommended Crates

| Purpose | Crate |
|---------|-------|
| TLS | rustls |
| Primitives | ring, RustCrypto |
| Password hashing | argon2 (Argon2id) |
| AEAD | aes-gcm, chacha20poly1305 |
| Constant-time | subtle |
| JWT | jsonwebtoken |

**Never roll your own crypto.**

### Parse, Don't Validate

```rust
pub struct EmailAddress(String);

impl EmailAddress {
    pub fn parse(input: &str) -> Result<Self, EmailError> {
        let trimmed = input.trim();
        if trimmed.is_empty() { return Err(EmailError::Empty); }
        if !trimmed.contains('@') { return Err(EmailError::MissingAt); }
        Ok(Self(trimmed.to_lowercase()))
    }
}
```

### Web Security

```rust
// CORS
CorsLayer::new()
    .allow_origin(["https://example.com".parse().unwrap()])
    .allow_methods([Method::GET, Method::POST])
    .allow_credentials(true)

// SQL injection — always parameterize
sqlx::query_as::<_, User>("SELECT * FROM users WHERE email = $1")
    .bind(&email).fetch_optional(&pool).await?;

// XSS — ammonia for HTML sanitization, Askama auto-escapes
let safe = ammonia::clean(user_input);

// Rate limiting — governor crate
let limiter = RateLimiter::direct(Quota::per_minute(NonZeroU32::new(100).unwrap()));
```

### Secret Management

```rust
use secrecy::{ExposeSecret, SecretString};

pub struct DbConfig {
    pub host: String,
    pub password: SecretString,  // Debug shows [REDACTED]
}

fn connect(cfg: &DbConfig) {
    let pw = cfg.password.expose_secret(); // explicit opt-in
}
```

Use `zeroize`/`ZeroizeOnDrop` for custom secret types. Never log secrets.

---

## 7. Things to Avoid

### Integer Overflow

Release mode wraps silently. Fix:

```toml
[profile.release]
overflow-checks = true
```

Or use explicit methods: `checked_add`, `saturating_add`, `wrapping_add`.

### Silent `as` Truncation

```rust
let big: i64 = 300;
// BAD: let small: u8 = big as u8; — silently gives 44
// GOOD:
let small = u8::try_from(big)?;
```

### `unwrap()` in Production

```rust
// BAD
let port: u16 = std::env::var("PORT").unwrap().parse().unwrap();

// GOOD
let port: u16 = std::env::var("PORT")
    .unwrap_or_else(|_| "8080".into())
    .parse()
    .unwrap_or(8080);
```

**Acceptable**: tests, provably valid values (hardcoded regex), prototypes.

### Panicking Indexing

```rust
// BAD: items[5] — panics OOB
// GOOD: items.get(5) — returns Option
```

### Path Traversal via `Path::join`

```rust
// SURPRISE: Path::new("/srv").join("/etc/passwd") → "/etc/passwd"
// FIX: reject absolute paths and ".." from user input
```

### Default Trait Pitfalls

```rust
// BAD: #[derive(Default)] gives port: 0
// GOOD: impl Default manually with meaningful values, or use NonZero types
```

### Other

- Don't `clone()` to appease borrow checker — restructure ownership
- Don't `pub` everything — expose minimal API
- Don't use `Box<dyn Error>` in libraries — use `thiserror`
- Don't use string types for structured data — use enums/newtypes
- Don't skip `// SAFETY:` comments on unsafe blocks
