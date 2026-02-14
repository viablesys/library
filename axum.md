# Axum Reference

Axum is an ergonomic, modular web framework built on **tokio**, **tower**, and **hyper** by the Tokio team.
Current stable: **0.8.x** (0.8.0 released 2025-01-01). Uses `#![forbid(unsafe_code)]`.

---

## 1. Installation

### Cargo.toml

```toml
[dependencies]
axum = "0.8"
tokio = { version = "1", features = ["full"] }
tower = "0.5"
tower-http = { version = "0.6", features = ["cors", "trace", "compression-full", "fs", "timeout", "catch-panic", "set-header", "request-id", "normalize-path", "limit"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"

# Optional — for SSE streams
tokio-stream = "0.1"
futures-util = "0.3"
```

### Feature Flags

| Flag | Default | Purpose |
|------|---------|---------|
| `http1` | yes | hyper HTTP/1.1 support |
| `http2` | no | hyper HTTP/2 support |
| `json` | yes | `Json` extractor/response, serde integration |
| `form` | yes | `Form` extractor |
| `query` | yes | `Query` extractor |
| `matched-path` | yes | `MatchedPath` extractor |
| `original-uri` | yes | `OriginalUri` extractor |
| `multipart` | no | `Multipart` extractor (file uploads) |
| `ws` | no | WebSocket support |
| `macros` | no | `#[debug_handler]` and derive macros |
| `tokio` | yes | tokio runtime + `axum::serve` |
| `tracing` | yes | rejection logging via tracing |
| `tower-log` | yes | tower's log feature |

### Minimal Hello World

```rust
use axum::{routing::get, Router};

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello, World!" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

---

## 2. Routing

### Router Basics

```rust
use axum::{routing::{get, post, put, patch, delete}, Router};

let app = Router::new()
    .route("/", get(root_handler))
    .route("/users", get(list_users).post(create_user))
    .route("/users/{id}", get(get_user).put(update_user).delete(delete_user));
```

### Path Parameter Syntax (v0.8 Change)

v0.8 changed from `:id` / `*rest` to `{id}` / `{*rest}` (aligns with OpenAPI, uses matchit 0.8):

```rust
// Single param
.route("/users/{id}", get(get_user))

// Multiple params
.route("/users/{user_id}/posts/{post_id}", get(get_post))

// Catch-all (greedy)
.route("/files/{*path}", get(serve_file))

// Literal braces — escape with double braces
.route("/literal/{{braces}}", get(handler))
```

### Method Routing Functions

| Function | HTTP Method |
|----------|-------------|
| `get()` | GET |
| `post()` | POST |
| `put()` | PUT |
| `patch()` | PATCH |
| `delete()` | DELETE |
| `head()` | HEAD |
| `options()` | OPTIONS |
| `trace()` | TRACE |
| `any()` | All methods |
| `on(MethodFilter, handler)` | Custom method filter |

Each has a `*_service` variant for routing to Tower services: `get_service()`, `post_service()`, etc.

### Nesting

```rust
let api = Router::new()
    .route("/users", get(list_users))
    .route("/posts", get(list_posts));

let app = Router::new()
    .nest("/api/v1", api);
// Matches: /api/v1/users, /api/v1/posts
```

### Merging Routers

```rust
let user_routes = Router::new().route("/users", get(list_users));
let post_routes = Router::new().route("/posts", get(list_posts));

let app = Router::new()
    .merge(user_routes)
    .merge(post_routes);
```

### Fallback

```rust
let app = Router::new()
    .route("/", get(root))
    .fallback(not_found_handler);

async fn not_found_handler() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "nothing here")
}
```

### route_service — Route to a Tower Service

```rust
use tower_http::services::ServeFile;

let app = Router::new()
    .route_service("/index.html", ServeFile::new("assets/index.html"));
```

### nest_service — Nest a Tower Service

```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .nest_service("/static", ServeDir::new("assets"));
```

---

## 3. Handlers

Handlers are async functions taking 0..16 extractors and returning `impl IntoResponse`.

```rust
// No extractors
async fn root() -> &'static str {
    "Hello"
}

// With extractors
async fn create_user(Json(payload): Json<CreateUser>) -> StatusCode {
    StatusCode::CREATED
}

// Closure handler
.route("/", get(|| async { "inline" }))

// Closure capturing state
let db = Arc::new(pool);
.route("/users", get({
    let db = Arc::clone(&db);
    move || async move { /* use db */ }
}))
```

### Return Types (impl IntoResponse)

```rust
// &'static str — 200 text/plain
async fn a() -> &'static str { "hello" }

// String — 200 text/plain; charset=utf-8
async fn b() -> String { "hello".to_string() }

// StatusCode — status with empty body
async fn c() -> StatusCode { StatusCode::NO_CONTENT }

// Json — 200 application/json
async fn d() -> Json<Value> { Json(json!({"ok": true})) }

// Html — 200 text/html
async fn e() -> Html<&'static str> { Html("<h1>Hi</h1>") }

// Tuples — (StatusCode, body)
async fn f() -> (StatusCode, String) {
    (StatusCode::CREATED, "created".into())
}

// Tuples — (StatusCode, headers, body)
async fn g() -> (StatusCode, [(HeaderName, &'static str); 1], String) {
    (StatusCode::OK, [(header::CONTENT_TYPE, "text/plain")], "hi".into())
}

// Result — Ok or error response
async fn h() -> Result<Json<User>, StatusCode> {
    Ok(Json(user))
}

// Bytes / Vec<u8> — application/octet-stream
async fn i() -> Vec<u8> { vec![0, 1, 2] }

// () — 200 with empty body
async fn j() {}

// Response — full control
async fn k() -> Response {
    Response::builder()
        .status(StatusCode::NOT_FOUND)
        .header("x-custom", "value")
        .body(Body::from("not found"))
        .unwrap()
}

// Redirect
async fn l() -> Redirect {
    Redirect::to("/other")
}
```

### Multiple Return Types

Use `impl IntoResponse` with `into_response()` on each branch:

```rust
async fn handler() -> impl IntoResponse {
    if condition {
        (StatusCode::OK, "success").into_response()
    } else {
        StatusCode::INTERNAL_SERVER_ERROR.into_response()
    }
}
```

Or use `Result<T, E>` where both T and E implement `IntoResponse`:

```rust
async fn handler() -> Result<Json<Data>, (StatusCode, String)> {
    let data = fetch().map_err(|e| (StatusCode::INTERNAL_SERVER_ERROR, e.to_string()))?;
    Ok(Json(data))
}
```

---

## 4. Extractors

Extractors implement `FromRequest` (consumes body) or `FromRequestParts` (no body access).

**Ordering rule**: Body-consuming extractors must be the **last** parameter. Only **one** body extractor allowed.

### Built-in Extractors

| Extractor | Source | Consumes Body |
|-----------|--------|---------------|
| `Path<T>` | URL path segments | No |
| `Query<T>` | Query string `?key=val` | No |
| `Json<T>` | JSON request body | Yes |
| `Form<T>` | URL-encoded form body | Yes |
| `State<S>` | Application state | No |
| `Extension<T>` | Request extensions | No |
| `HeaderMap` | All request headers | No |
| `ConnectInfo<T>` | Client connection info | No |
| `Multipart` | multipart/form-data | Yes |
| `OriginalUri` | Original URI before nesting | No |
| `MatchedPath` | Matched route pattern | No |
| `Host` | Host header | No |
| `NestedPath` | Path at nesting level | No |
| `RawQuery` | Raw query string | No |
| `RawForm` | Raw form data | Yes |
| `Request` | Full `http::Request<Body>` | Yes |
| `Bytes` | Raw body bytes | Yes |
| `String` | Body as UTF-8 string | Yes |
| `Body` | Body stream | Yes |
| `WebSocketUpgrade` | WebSocket upgrade | No |

### Usage Examples

```rust
use axum::extract::{Path, Query, Json, State, ConnectInfo, MatchedPath, OriginalUri};
use std::collections::HashMap;
use std::net::SocketAddr;

// Path params
async fn get_user(Path(id): Path<u64>) -> String {
    format!("User {id}")
}

// Multiple path params
async fn get_post(Path((user_id, post_id)): Path<(u64, u64)>) -> String {
    format!("User {user_id}, Post {post_id}")
}

// Path as struct (requires serde::Deserialize)
#[derive(Deserialize)]
struct PostPath { user_id: u64, post_id: u64 }
async fn get_post2(Path(p): Path<PostPath>) -> String {
    format!("User {}, Post {}", p.user_id, p.post_id)
}

// Query params
#[derive(Deserialize)]
struct Pagination { page: u32, per_page: u32 }
async fn list(Query(p): Query<Pagination>) -> String {
    format!("page={} per_page={}", p.page, p.per_page)
}

// JSON body
#[derive(Deserialize)]
struct CreateUser { name: String }
async fn create(Json(payload): Json<CreateUser>) -> StatusCode {
    StatusCode::CREATED
}

// Headers
async fn headers(headers: HeaderMap) -> String {
    let ua = headers.get("user-agent").and_then(|v| v.to_str().ok()).unwrap_or("unknown");
    format!("UA: {ua}")
}

// ConnectInfo (requires into_make_service_with_connect_info)
async fn client_addr(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    format!("Connected from: {addr}")
}

// MatchedPath
async fn matched(MatchedPath(path): MatchedPath) -> String {
    format!("Matched: {path}")
}

// OriginalUri (useful inside nest())
async fn original(OriginalUri(uri): OriginalUri) -> String {
    format!("Original: {uri}")
}
```

### Optional and Result Extractors

```rust
// Option — None if extraction fails (extractor must impl OptionalFromRequestParts)
async fn handler(user_agent: Option<TypedHeader<UserAgent>>) { }

// Result — get the rejection error
async fn handler(payload: Result<Json<Value>, JsonRejection>) {
    match payload {
        Ok(Json(v)) => { /* use v */ }
        Err(JsonRejection::MissingJsonContentType(_)) => { /* 415 */ }
        Err(JsonRejection::JsonDataError(e)) => { /* 422 */ }
        Err(JsonRejection::JsonSyntaxError(e)) => { /* 400 */ }
        Err(JsonRejection::BytesRejection(e)) => { /* 413 etc */ }
        Err(e) => { /* other */ }
    }
}
```

### Body Size Limit

Default max body: **2 MB** (affects `Bytes`, `String`, `Json`, `Form`). Override:

```rust
use axum::extract::DefaultBodyLimit;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(DefaultBodyLimit::max(1024 * 1024 * 50)); // 50 MB
```

### Rejection Logging

Enable tracing of extraction failures:

```bash
RUST_LOG=info,axum::rejection=trace cargo run
```

---

## 5. Custom Extractors

### FromRequestParts (no body access)

```rust
use axum::extract::FromRequestParts;
use http::request::Parts;

struct ExtractUserAgent(String);

impl<S> FromRequestParts<S> for ExtractUserAgent
where
    S: Send + Sync,
{
    type Rejection = (StatusCode, &'static str);

    async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
        let ua = parts.headers
            .get(http::header::USER_AGENT)
            .and_then(|v| v.to_str().ok())
            .map(|s| s.to_string())
            .ok_or((StatusCode::BAD_REQUEST, "Missing User-Agent"))?;
        Ok(ExtractUserAgent(ua))
    }
}
```

### FromRequest (consumes body)

```rust
use axum::extract::FromRequest;

struct ValidatedJson<T>(T);

impl<S, T> FromRequest<S> for ValidatedJson<T>
where
    Json<T>: FromRequest<S>,
    T: Validate,
    S: Send + Sync,
{
    type Rejection = Response;

    async fn from_request(req: Request, state: &S) -> Result<Self, Self::Rejection> {
        let Json(value) = Json::<T>::from_request(req, state)
            .await
            .map_err(IntoResponse::into_response)?;
        value.validate().map_err(|e| {
            (StatusCode::UNPROCESSABLE_ENTITY, e.to_string()).into_response()
        })?;
        Ok(ValidatedJson(value))
    }
}
```

### Custom Rejection Types

```rust
enum ApiError {
    BadRequest(String),
    NotFound,
}

impl IntoResponse for ApiError {
    fn into_response(self) -> Response {
        let (status, msg) = match self {
            ApiError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
            ApiError::NotFound => (StatusCode::NOT_FOUND, "not found".into()),
        };
        (status, Json(json!({"error": msg}))).into_response()
    }
}
```

### Derive Macros (requires `macros` feature)

```rust
#[derive(FromRequest)]       // auto-impl FromRequest
#[derive(FromRequestParts)]  // auto-impl FromRequestParts
#[derive(FromRef)]           // auto-impl FromRef for substates
```

### Accessing Other Extractors Inside Custom Extractors

```rust
use axum::RequestPartsExt;

async fn from_request_parts(parts: &mut Parts, _state: &S) -> Result<Self, Self::Rejection> {
    let Extension(pool) = parts.extract::<Extension<PgPool>>()
        .await
        .map_err(|_| (StatusCode::INTERNAL_SERVER_ERROR, "missing pool"))?;
    // use pool...
    Ok(Self { /* ... */ })
}
```

---

## 6. State

### State Extractor (Preferred)

```rust
use axum::extract::State;
use std::sync::Arc;

#[derive(Clone)]
struct AppState {
    db: PgPool,
    redis: RedisPool,
}

#[tokio::main]
async fn main() {
    let state = AppState { db: pool, redis: redis };

    let app = Router::new()
        .route("/users", get(list_users))
        .with_state(state);

    // ...
}

async fn list_users(State(state): State<AppState>) -> Json<Vec<User>> {
    let users = sqlx::query_as("SELECT * FROM users")
        .fetch_all(&state.db).await.unwrap();
    Json(users)
}
```

### Arc Pattern (for non-Clone state or avoiding clones)

```rust
use std::sync::Arc;

struct AppState {
    db: PgPool, // PgPool is already Clone, but this avoids cloning the whole struct
}

let shared = Arc::new(AppState { db: pool });

let app = Router::new()
    .route("/", get(handler))
    .with_state(shared);

async fn handler(State(state): State<Arc<AppState>>) { }
```

### Shared Mutable State

```rust
use std::sync::Arc;
use tokio::sync::Mutex;

#[derive(Clone)]
struct AppState {
    counter: Arc<Mutex<u64>>,
}

async fn increment(State(state): State<AppState>) -> String {
    let mut count = state.counter.lock().await;
    *count += 1;
    format!("Count: {count}")
}
```

Use `tokio::sync::Mutex` (not `std::sync::Mutex`) when holding the lock across `.await` points. Use `std::sync::Mutex` when the critical section has no awaits (lower overhead).

### Substates with FromRef

```rust
use axum::extract::FromRef;

#[derive(Clone)]
struct AppState {
    db: PgPool,
    api_key: String,
}

// Derive approach
#[derive(Clone, FromRef)]
struct AppState2 {
    db: PgPool,
    api_key: String,
}

// Manual approach
impl FromRef<AppState> for PgPool {
    fn from_ref(state: &AppState) -> Self {
        state.db.clone()
    }
}

// Handler can extract substate directly
async fn handler(State(pool): State<PgPool>) -> impl IntoResponse {
    // pool extracted via FromRef
}
```

### Nested Routers with Different State

```rust
let api_routes = Router::new()
    .route("/data", get(api_handler))
    .with_state(ApiState { /* ... */ });

let app = Router::new()
    .nest("/api", api_routes)
    .with_state(AppState { /* ... */ });
```

---

## 7. Response Types

### IntoResponse Implementations

Everything that implements `IntoResponse` can be returned from handlers:

```rust
// Tuple patterns
(StatusCode, impl IntoResponse)
(HeaderMap, impl IntoResponse)
(StatusCode, HeaderMap, impl IntoResponse)
(StatusCode, [(HeaderName, &str); N], impl IntoResponse)
([(HeaderName, &str); N], impl IntoResponse)

// Parts tuple (up to many IntoResponseParts + body)
(Part1, Part2, ..., impl IntoResponse)
```

### Custom IntoResponse

```rust
struct ApiResponse<T: Serialize> {
    status: StatusCode,
    data: T,
}

impl<T: Serialize> IntoResponse for ApiResponse<T> {
    fn into_response(self) -> Response {
        (self.status, Json(json!({ "data": self.data }))).into_response()
    }
}
```

### Typed Headers (via axum-extra)

```rust
// Cargo.toml: axum-extra = { version = "0.10", features = ["typed-header"] }
use axum_extra::TypedHeader;
use headers::UserAgent;

async fn handler(TypedHeader(ua): TypedHeader<UserAgent>) -> String {
    ua.to_string()
}
```

### Streaming Responses

```rust
use axum::body::Body;
use tokio_stream::StreamExt;
use futures_util::stream;

async fn stream_handler() -> Body {
    let stream = stream::iter(vec!["hello ", "world"])
        .map(|s| Ok::<_, std::io::Error>(s));
    Body::from_stream(stream)
}
```

### AppendHeaders

```rust
use axum::response::AppendHeaders;

async fn handler() -> (AppendHeaders<[(HeaderName, &'static str); 2]>, &'static str) {
    (
        AppendHeaders([
            (header::SET_COOKIE, "a=1"),
            (header::SET_COOKIE, "b=2"),
        ]),
        "body",
    )
}
```

### Redirect

```rust
use axum::response::Redirect;

async fn temp() -> Redirect { Redirect::temporary("/new") }
async fn perm() -> Redirect { Redirect::permanent("/new") }
async fn see_other() -> Redirect { Redirect::to("/new") } // 303
```

---

## 8. Middleware

Axum uses Tower middleware. No custom middleware system.

### Execution Order

```rust
// .layer() wraps — last added runs first (outermost):
Router::new()
    .route("/", get(handler))
    .layer(layer_a)  // runs 2nd
    .layer(layer_b); // runs 1st (outermost)

// ServiceBuilder runs top-to-bottom (more intuitive):
use tower::ServiceBuilder;

Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(layer_a)  // runs 1st (outermost)
            .layer(layer_b)  // runs 2nd
    );
```

### from_fn Middleware

```rust
use axum::middleware::{self, Next};

async fn my_middleware(request: Request, next: Next) -> Response {
    // Before handler
    let start = std::time::Instant::now();

    let response = next.run(request).await;

    // After handler
    let duration = start.elapsed();
    println!("Request took {duration:?}");

    response
}

let app = Router::new()
    .route("/", get(handler))
    .layer(middleware::from_fn(my_middleware));
```

### from_fn with State

```rust
async fn auth_middleware(
    State(state): State<AppState>,
    request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = request.headers()
        .get(header::AUTHORIZATION)
        .and_then(|v| v.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    if !state.validate_token(token) {
        return Err(StatusCode::UNAUTHORIZED);
    }

    Ok(next.run(request).await)
}

let app = Router::new()
    .route("/protected", get(protected_handler))
    .layer(middleware::from_fn_with_state(state.clone(), auth_middleware));
```

### from_extractor Middleware

```rust
use axum::middleware::from_extractor;

// Any extractor can be used as middleware — request is rejected if extraction fails
let app = Router::new()
    .route("/", get(handler))
    .layer(from_extractor::<RequireAuth>());
```

### map_request / map_response

```rust
use axum::middleware::{map_request, map_response};

async fn add_request_id(mut req: Request) -> Request {
    req.extensions_mut().insert(RequestId::new());
    req
}

async fn add_server_header(mut res: Response) -> Response {
    res.headers_mut().insert("x-server", "axum".parse().unwrap());
    res
}

let app = Router::new()
    .route("/", get(handler))
    .layer(map_request(add_request_id))
    .layer(map_response(add_server_header));
```

### Route-Level vs Router-Level Middleware

```rust
// Router-level — runs on ALL requests including 404s
Router::new()
    .route("/", get(handler))
    .layer(middleware);

// Route-level — runs ONLY on matched routes
Router::new()
    .route("/", get(handler))
    .route_layer(middleware);
```

### Built-in tower-http Layers

#### TraceLayer

```rust
use tower_http::trace::TraceLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(TraceLayer::new_for_http());
```

#### CorsLayer

```rust
use tower_http::cors::{CorsLayer, Any};
use http::Method;

// Permissive (dev)
let cors = CorsLayer::permissive();

// Restrictive (prod)
let cors = CorsLayer::new()
    .allow_origin("https://example.com".parse::<HeaderValue>().unwrap())
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::DELETE])
    .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION])
    .allow_credentials(true)
    .max_age(Duration::from_secs(3600));

let app = Router::new()
    .route("/api", get(handler))
    .layer(cors);
```

#### CompressionLayer

```rust
use tower_http::compression::CompressionLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(CompressionLayer::new());
```

#### TimeoutLayer

```rust
use tower_http::timeout::TimeoutLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(TimeoutLayer::new(Duration::from_secs(30)));
```

Note: `TimeoutLayer` returns an error, not a response. Wrap with `HandleErrorLayer`:

```rust
use axum::error_handling::HandleErrorLayer;
use tower::ServiceBuilder;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(HandleErrorLayer::new(|_: BoxError| async {
                StatusCode::REQUEST_TIMEOUT
            }))
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
    );
```

#### CatchPanicLayer

```rust
use tower_http::catch_panic::CatchPanicLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(CatchPanicLayer::new());
```

#### SetResponseHeaderLayer

```rust
use tower_http::set_header::SetResponseHeaderLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(SetResponseHeaderLayer::overriding(
        header::SERVER,
        HeaderValue::from_static("my-server"),
    ));
```

#### RequestBodyLimitLayer

```rust
use tower_http::limit::RequestBodyLimitLayer;

let app = Router::new()
    .route("/upload", post(upload))
    .layer(RequestBodyLimitLayer::new(1024 * 1024 * 10)); // 10 MB
```

#### NormalizePathLayer

```rust
use tower_http::normalize_path::NormalizePathLayer;

// Must wrap entire router (before routing decisions)
let app = NormalizePathLayer::trim_trailing_slash()
    .layer(Router::new().route("/foo", get(handler)));
```

#### RequestIdLayer

```rust
use tower_http::request_id::{SetRequestIdLayer, PropagateRequestIdLayer, MakeRequestUuid};

let app = Router::new()
    .route("/", get(handler))
    .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
    .layer(PropagateRequestIdLayer::x_request_id());
```

### Recommended ServiceBuilder Stack

```rust
use tower::ServiceBuilder;
use tower_http::{
    trace::TraceLayer,
    compression::CompressionLayer,
    cors::CorsLayer,
    timeout::TimeoutLayer,
    catch_panic::CatchPanicLayer,
    request_id::{SetRequestIdLayer, PropagateRequestIdLayer, MakeRequestUuid},
};

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
            .layer(PropagateRequestIdLayer::x_request_id())
            .layer(TraceLayer::new_for_http())
            .layer(CatchPanicLayer::new())
            .layer(HandleErrorLayer::new(|_: BoxError| async {
                StatusCode::REQUEST_TIMEOUT
            }))
            .layer(TimeoutLayer::new(Duration::from_secs(30)))
            .layer(CompressionLayer::new())
            .layer(CorsLayer::permissive())
    );
```

---

## 9. Error Handling

Axum requires all services to be infallible (`Error = Infallible`). Errors are modeled as responses.

### Handler Errors via Result

```rust
async fn handler() -> Result<Json<Data>, AppError> {
    let data = fetch_data().await?;
    Ok(Json(data))
}
```

### Custom Error Type

```rust
use axum::response::{IntoResponse, Response};

enum AppError {
    NotFound,
    Internal(anyhow::Error),
    BadRequest(String),
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "not found".to_string()),
            AppError::Internal(e) => {
                tracing::error!("Internal error: {e:?}");
                (StatusCode::INTERNAL_SERVER_ERROR, "internal error".to_string())
            }
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg),
        };
        (status, Json(json!({"error": message}))).into_response()
    }
}

// Enable ? with anyhow
impl From<anyhow::Error> for AppError {
    fn from(e: anyhow::Error) -> Self {
        AppError::Internal(e)
    }
}

// Enable ? with sqlx
impl From<sqlx::Error> for AppError {
    fn from(e: sqlx::Error) -> Self {
        AppError::Internal(e.into())
    }
}
```

### HandleErrorLayer for Fallible Middleware

```rust
use axum::error_handling::HandleErrorLayer;

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(HandleErrorLayer::new(|method: Method, uri: Uri, err: BoxError| async move {
                (
                    StatusCode::INTERNAL_SERVER_ERROR,
                    format!("`{method} {uri}` failed: {err}"),
                )
            }))
            .layer(TimeoutLayer::new(Duration::from_secs(10)))
    );
```

### HandleError for Fallible Services

```rust
use axum::error_handling::HandleError;

let fallible_service = tower::service_fn(|_req| async {
    thing().await?;
    Ok::<_, anyhow::Error>(Response::new(Body::empty()))
});

let app = Router::new().route_service(
    "/",
    HandleError::new(fallible_service, |err: anyhow::Error| async move {
        (StatusCode::INTERNAL_SERVER_ERROR, err.to_string())
    }),
);
```

---

## 10. WebSocket

Requires `ws` feature: `axum = { version = "0.8", features = ["ws"] }`

### Basic WebSocket Handler

```rust
use axum::extract::ws::{Message, WebSocket, WebSocketUpgrade};
use axum::response::Response;

async fn ws_handler(ws: WebSocketUpgrade) -> Response {
    ws.on_upgrade(handle_socket)
}

async fn handle_socket(mut socket: WebSocket) {
    while let Some(msg) = socket.recv().await {
        match msg {
            Ok(Message::Text(text)) => {
                if socket.send(Message::Text(format!("Echo: {text}"))).await.is_err() {
                    break; // client disconnected
                }
            }
            Ok(Message::Binary(data)) => {
                let _ = socket.send(Message::Binary(data)).await;
            }
            Ok(Message::Ping(data)) => {
                let _ = socket.send(Message::Pong(data)).await;
            }
            Ok(Message::Close(_)) => break,
            Err(_) => break,
            _ => {}
        }
    }
}

let app = Router::new().route("/ws", get(ws_handler));
```

### Message Types

| Variant | Description |
|---------|-------------|
| `Message::Text(String)` | UTF-8 text frame |
| `Message::Binary(Vec<u8>)` | Binary data frame |
| `Message::Ping(Vec<u8>)` | Ping heartbeat |
| `Message::Pong(Vec<u8>)` | Pong response |
| `Message::Close(Option<CloseFrame>)` | Close frame |

### Split for Concurrent Read/Write

```rust
use futures_util::{SinkExt, StreamExt};

async fn handle_socket(socket: WebSocket) {
    let (mut sender, mut receiver) = socket.split();

    // Spawn a task for sending
    let mut send_task = tokio::spawn(async move {
        loop {
            tokio::time::sleep(Duration::from_secs(1)).await;
            if sender.send(Message::Text("tick".into())).await.is_err() {
                break;
            }
        }
    });

    // Receive in main task
    let mut recv_task = tokio::spawn(async move {
        while let Some(Ok(msg)) = receiver.next().await {
            match msg {
                Message::Text(text) => println!("Received: {text}"),
                Message::Close(_) => break,
                _ => {}
            }
        }
    });

    // Wait for either task to finish
    tokio::select! {
        _ = &mut send_task => recv_task.abort(),
        _ = &mut recv_task => send_task.abort(),
    };
}
```

### WebSocket with State

```rust
async fn ws_handler(
    ws: WebSocketUpgrade,
    State(state): State<AppState>,
) -> Response {
    ws.on_upgrade(move |socket| handle_socket(socket, state))
}

async fn handle_socket(socket: WebSocket, state: AppState) {
    // use state...
}
```

---

## 11. SSE (Server-Sent Events)

### Basic SSE Handler

```rust
use axum::response::sse::{Event, KeepAlive, Sse};
use futures_util::stream::{self, Stream};
use std::convert::Infallible;
use tokio_stream::StreamExt as _;

async fn sse_handler() -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let stream = stream::repeat_with(|| {
        Event::default().data("hello")
    })
    .map(Ok)
    .throttle(Duration::from_secs(1));

    Sse::new(stream).keep_alive(KeepAlive::default())
}

let app = Router::new().route("/sse", get(sse_handler));
```

### Event Builder

```rust
Event::default()
    .data("message body")           // data field
    .event("custom-event")          // event type
    .id("msg-123")                  // event ID
    .retry(Duration::from_secs(5))  // retry interval
    .comment("a comment")           // comment line
```

### SSE with Channel (Dynamic Events)

```rust
use tokio::sync::broadcast;
use tokio_stream::wrappers::BroadcastStream;

async fn sse_handler(
    State(tx): State<broadcast::Sender<String>>,
) -> Sse<impl Stream<Item = Result<Event, Infallible>>> {
    let rx = tx.subscribe();
    let stream = BroadcastStream::new(rx)
        .filter_map(|result| result.ok())
        .map(|msg| Ok(Event::default().data(msg)));

    Sse::new(stream).keep_alive(
        KeepAlive::new()
            .interval(Duration::from_secs(15))
            .text("keep-alive"),
    )
}
```

### Caveat: Compression

Do NOT apply `CompressionLayer` to SSE routes. Compression buffers the response, preventing incremental event delivery. Exclude SSE routes or apply compression selectively.

---

## 12. Static Files

### ServeDir — Serve a Directory

```rust
use tower_http::services::ServeDir;

let app = Router::new()
    .nest_service("/static", ServeDir::new("assets"));
// GET /static/style.css -> reads assets/style.css
```

### ServeFile — Serve a Single File

```rust
use tower_http::services::ServeFile;

let app = Router::new()
    .route_service("/robots.txt", ServeFile::new("static/robots.txt"));
```

### SPA Fallback (index.html for missing files)

```rust
let serve_dir = ServeDir::new("dist")
    .not_found_service(ServeFile::new("dist/index.html"));

let app = Router::new()
    .route("/api/health", get(|| async { "ok" }))
    .fallback_service(serve_dir);
```

### Serve from Root

```rust
let app = Router::new()
    .route("/api/data", get(api_handler))
    .fallback_service(ServeDir::new("public"));
// GET /index.html -> reads public/index.html
```

### Custom 404 Handler

```rust
async fn handle_404() -> (StatusCode, &'static str) {
    (StatusCode::NOT_FOUND, "File not found")
}

let serve_dir = ServeDir::new("assets")
    .not_found_service(handle_404.into_service());

let app = Router::new().fallback_service(serve_dir);
```

---

## 13. Testing

### Approach 1: tower::ServiceExt::oneshot (No Extra Deps)

```rust
#[cfg(test)]
mod tests {
    use super::*;
    use axum::{body::Body, http::{Request, StatusCode}};
    use http_body_util::BodyExt;
    use tower::ServiceExt; // for oneshot
    use serde_json::{json, Value};

    #[tokio::test]
    async fn test_hello() {
        let app = app();

        let response = app
            .oneshot(
                Request::builder()
                    .uri("/")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::OK);
        let body = response.into_body().collect().await.unwrap().to_bytes();
        assert_eq!(&body[..], b"Hello, World!");
    }

    #[tokio::test]
    async fn test_json_post() {
        let app = app();

        let response = app
            .oneshot(
                Request::builder()
                    .method("POST")
                    .uri("/users")
                    .header("content-type", "application/json")
                    .body(Body::from(serde_json::to_string(&json!({
                        "name": "alice"
                    })).unwrap()))
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::CREATED);
    }

    #[tokio::test]
    async fn test_not_found() {
        let app = app();

        let response = app
            .oneshot(
                Request::builder()
                    .uri("/nonexistent")
                    .body(Body::empty())
                    .unwrap(),
            )
            .await
            .unwrap();

        assert_eq!(response.status(), StatusCode::NOT_FOUND);
    }

    // Multiple requests on the same app — must clone or recreate
    #[tokio::test]
    async fn test_multiple_requests() {
        let mut app = app().into_service();

        let req = Request::builder().uri("/").body(Body::empty()).unwrap();
        let res = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(req)
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::OK);

        let req = Request::builder().uri("/").body(Body::empty()).unwrap();
        let res = ServiceExt::<Request<Body>>::ready(&mut app)
            .await
            .unwrap()
            .call(req)
            .await
            .unwrap();
        assert_eq!(res.status(), StatusCode::OK);
    }
}
```

**Important**: `oneshot` consumes the service. For multiple requests, use `.into_service()` and call `.ready().await?.call(req)`.

**Important**: Router with state must call `.with_state(state)` before `oneshot` (needs `Router<()>`).

### Approach 2: axum-test Crate

```toml
[dev-dependencies]
axum-test = "16"
```

```rust
use axum_test::TestServer;

#[tokio::test]
async fn test_with_server() {
    let app = Router::new().route("/ping", get(|| async { "pong" }));
    let server = TestServer::new(app).unwrap();

    let response = server.get("/ping").await;
    response.assert_status_ok();
    response.assert_text("pong");
}
```

### Testing with ConnectInfo

```rust
use axum::extract::connect_info::MockConnectInfo;

let app = app()
    .layer(MockConnectInfo(SocketAddr::from(([127, 0, 0, 1], 3000))));
```

---

## 14. Graceful Shutdown

### with_graceful_shutdown

```rust
use tokio::signal;

#[tokio::main]
async fn main() {
    let app = Router::new().route("/", get(|| async { "Hello" }));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

async fn shutdown_signal() {
    let ctrl_c = async {
        signal::ctrl_c().await.expect("failed to install Ctrl+C handler");
    };

    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .expect("failed to install signal handler")
            .recv()
            .await;
    };

    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();

    tokio::select! {
        _ = ctrl_c => {},
        _ = terminate => {},
    }
}
```

### With Timeout Protection

```rust
let server = axum::serve(listener, app)
    .with_graceful_shutdown(shutdown_signal());

if let Err(e) = tokio::time::timeout(Duration::from_secs(30), server).await {
    tracing::warn!("Graceful shutdown timed out, forcing exit");
}
```

---

## 15. Common Patterns

### REST API Structure

```rust
mod handlers;
mod models;
mod errors;

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let pool = PgPool::connect(&std::env::var("DATABASE_URL").unwrap())
        .await.unwrap();

    let state = AppState { db: pool };

    let app = Router::new()
        .nest("/api/v1", api_routes())
        .layer(TraceLayer::new_for_http())
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .route("/users", get(handlers::list_users).post(handlers::create_user))
        .route("/users/{id}", get(handlers::get_user).put(handlers::update_user).delete(handlers::delete_user))
        .route("/health", get(|| async { "ok" }))
}
```

### Shared Database Pool

```rust
use sqlx::PgPool;

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

async fn list_users(State(state): State<AppState>) -> Result<Json<Vec<User>>, AppError> {
    let users = sqlx::query_as::<_, User>("SELECT id, name, email FROM users")
        .fetch_all(&state.db)
        .await?;
    Ok(Json(users))
}
```

### Auth Middleware

```rust
use axum::middleware::{self, Next};

async fn require_auth(
    State(state): State<AppState>,
    mut request: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let auth_header = request.headers()
        .get(header::AUTHORIZATION)
        .and_then(|h| h.to_str().ok())
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let token = auth_header
        .strip_prefix("Bearer ")
        .ok_or(StatusCode::UNAUTHORIZED)?;

    let user = state.validate_token(token)
        .await
        .map_err(|_| StatusCode::UNAUTHORIZED)?;

    // Insert user into request extensions for downstream handlers
    request.extensions_mut().insert(user);

    Ok(next.run(request).await)
}

// Apply to specific routes
let protected = Router::new()
    .route("/me", get(get_me))
    .route("/settings", put(update_settings))
    .layer(middleware::from_fn_with_state(state.clone(), require_auth));

let public = Router::new()
    .route("/login", post(login))
    .route("/register", post(register));

let app = Router::new()
    .merge(public)
    .merge(protected)
    .with_state(state);
```

### Request ID

```rust
use tower_http::request_id::{SetRequestIdLayer, PropagateRequestIdLayer, MakeRequestUuid};

let app = Router::new()
    .route("/", get(handler))
    .layer(
        ServiceBuilder::new()
            .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
            .layer(PropagateRequestIdLayer::x_request_id())
            .layer(TraceLayer::new_for_http())
    );
```

### CORS Setup (Production)

```rust
use tower_http::cors::{CorsLayer, AllowOrigin};
use http::{header, Method, HeaderValue};

let origins = [
    "https://app.example.com".parse::<HeaderValue>().unwrap(),
    "https://admin.example.com".parse::<HeaderValue>().unwrap(),
];

let cors = CorsLayer::new()
    .allow_origin(origins)
    .allow_methods([Method::GET, Method::POST, Method::PUT, Method::PATCH, Method::DELETE])
    .allow_headers([header::CONTENT_TYPE, header::AUTHORIZATION, header::ACCEPT])
    .allow_credentials(true)
    .max_age(Duration::from_secs(3600));

let app = Router::new()
    .nest("/api", api_routes())
    .layer(cors);
```

### Rate Limiting (via tower)

```rust
// Using tower::limit::RateLimitLayer
use tower::limit::RateLimitLayer;

let app = Router::new()
    .route("/api/data", get(handler))
    .layer(RateLimitLayer::new(100, Duration::from_secs(60))); // 100 req/min
```

### ConnectInfo (Client IP)

```rust
use axum::extract::ConnectInfo;
use std::net::SocketAddr;

async fn handler(ConnectInfo(addr): ConnectInfo<SocketAddr>) -> String {
    format!("Your IP: {}", addr.ip())
}

// Must use into_make_service_with_connect_info
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(
    listener,
    app.into_make_service_with_connect_info::<SocketAddr>(),
).await.unwrap();
```

### Debug Handler (Better Compiler Errors)

```rust
// Cargo.toml: axum-macros = "0.5"
use axum_macros::debug_handler;

#[debug_handler]
async fn handler(Json(payload): Json<CreateUser>) -> impl IntoResponse {
    // If this handler has type errors, the compiler message will be clear
    StatusCode::CREATED
}
```

### Full Production Skeleton

```rust
use axum::{
    extract::State,
    http::StatusCode,
    middleware,
    response::{IntoResponse, Json},
    routing::{get, post},
    Router,
};
use serde::{Deserialize, Serialize};
use sqlx::PgPool;
use std::sync::Arc;
use tokio::signal;
use tower::ServiceBuilder;
use tower_http::{
    catch_panic::CatchPanicLayer,
    compression::CompressionLayer,
    cors::CorsLayer,
    request_id::{MakeRequestUuid, PropagateRequestIdLayer, SetRequestIdLayer},
    trace::TraceLayer,
};

#[derive(Clone)]
struct AppState {
    db: PgPool,
}

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt()
        .with_env_filter("info,tower_http=debug,axum::rejection=trace")
        .init();

    let db = PgPool::connect(&std::env::var("DATABASE_URL").unwrap())
        .await
        .unwrap();

    let state = AppState { db };

    let app = Router::new()
        .nest("/api", api_routes())
        .fallback(|| async { (StatusCode::NOT_FOUND, "not found") })
        .layer(
            ServiceBuilder::new()
                .layer(SetRequestIdLayer::x_request_id(MakeRequestUuid))
                .layer(PropagateRequestIdLayer::x_request_id())
                .layer(TraceLayer::new_for_http())
                .layer(CatchPanicLayer::new())
                .layer(CompressionLayer::new())
                .layer(CorsLayer::permissive()),
        )
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    tracing::info!("listening on {}", listener.local_addr().unwrap());

    axum::serve(listener, app)
        .with_graceful_shutdown(shutdown_signal())
        .await
        .unwrap();
}

fn api_routes() -> Router<AppState> {
    Router::new()
        .route("/health", get(|| async { "ok" }))
        .route("/users", get(list_users).post(create_user))
        .route("/users/{id}", get(get_user))
}

async fn shutdown_signal() {
    let ctrl_c = async { signal::ctrl_c().await.unwrap() };
    #[cfg(unix)]
    let terminate = async {
        signal::unix::signal(signal::unix::SignalKind::terminate())
            .unwrap().recv().await;
    };
    #[cfg(not(unix))]
    let terminate = std::future::pending::<()>();
    tokio::select! { _ = ctrl_c => {}, _ = terminate => {} }
}
```

---

## Quick Reference

| Need | Solution |
|------|----------|
| JSON API | `Json<T>` extractor + `Json<T>` response |
| Path params | `Path<T>` with `/{name}` syntax |
| Query string | `Query<T>` with `Deserialize` struct |
| Shared state | `State<T>` + `.with_state()` |
| Mutable state | `State<Arc<Mutex<T>>>` |
| Substates | `#[derive(FromRef)]` on state struct |
| Auth middleware | `middleware::from_fn_with_state` |
| Static files | `tower_http::services::ServeDir` |
| SPA fallback | `ServeDir::new("dist").not_found_service(ServeFile::new("dist/index.html"))` |
| WebSocket | `ws` feature + `WebSocketUpgrade` extractor |
| SSE | `Sse::new(stream).keep_alive(KeepAlive::default())` |
| CORS | `tower_http::cors::CorsLayer` |
| Compression | `tower_http::compression::CompressionLayer` |
| Request tracing | `tower_http::trace::TraceLayer` |
| Timeout | `HandleErrorLayer` + `TimeoutLayer` |
| Rate limit | `tower::limit::RateLimitLayer` |
| Body limit | `DefaultBodyLimit::max(bytes)` or `RequestBodyLimitLayer` |
| Testing | `tower::ServiceExt::oneshot` or `axum-test` crate |
| Better errors | `#[debug_handler]` from `axum-macros` |
| Graceful stop | `.with_graceful_shutdown(shutdown_signal())` |
