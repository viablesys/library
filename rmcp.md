# rmcp Reference

Rust SDK for the Model Context Protocol. Build MCP servers that expose tools to Claude Code sessions.
Current version: **0.15.x**. Repo: <https://github.com/modelcontextprotocol/rust-sdk>

For plugin architecture (hooks, .mcp.json config, discovery): see `claude-code-plugins.md`.
This doc covers rmcp implementation -- tool definitions, server state, stdio transport, error handling, testing.

---

## 1. Installation

```toml
[dependencies]
rmcp = { version = "0.15", features = ["server", "transport-io"] }
tokio = { version = "1", features = ["macros", "rt-multi-thread", "io-std"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
schemars = "0.8"
```

| Feature | Purpose |
|---------|---------|
| `server` | Server handler traits, `#[tool]` macro, `ToolRouter` |
| `transport-io` | Async stdin/stdout for stdio servers |
| `transport-streamable-http-server` | HTTP streaming transport |
| `transport-child-process` | Spawn server as child process (client-side, for tests) |
| `schemars` | JSON Schema generation for tool parameters |

For a session-scoped stdio MCP server, `server` + `transport-io` is sufficient.

---

## 2. Server Setup

```rust
use rmcp::{ServerHandler, ServiceExt, model::*, transport::stdio};

#[derive(Clone)]
struct NmemServer;

impl ServerHandler for NmemServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            instructions: Some("nmem observation search and retrieval".into()),
            capabilities: ServerCapabilities::builder()
                .enable_tools()
                .build(),
            ..Default::default()
        }
    }
}

#[tokio::main]
async fn main() -> anyhow::Result<()> {
    eprintln!("nmem-mcp: starting");  // stderr only -- stdout is MCP protocol
    let service = NmemServer.serve(stdio()).await?;
    service.waiting().await?;  // blocks until stdin closes
    Ok(())
}
```

**Lifecycle:** Claude Code spawns the binary (stdin/stdout pipes) -> `serve(stdio())` performs MCP handshake -> `waiting()` blocks on stdin -> stdin closes when session ends -> process exits.

### Graceful Shutdown

Use `serve_server_with_ct(server, stdio(), ct)` with a `tokio_util::sync::CancellationToken` for signal-driven shutdown. For session-scoped servers this is optional -- stdin close handles it.

---

## 3. Defining Tools

Three macros work together: `#[tool_router]` on the impl block collects tools into a `ToolRouter`, `#[tool]` on methods marks each as an MCP tool, `#[tool_handler]` on the `ServerHandler` impl wires the router to `call_tool`/`list_tools`.

```rust
use rmcp::{
    ServerHandler, handler::server::{tool::ToolRouter, wrapper::Parameters},
    model::*, tool, tool_handler, tool_router, ErrorData,
};
use schemars::JsonSchema;
use serde::Deserialize;

type DbHandle = Arc<Mutex<rusqlite::Connection>>;

#[derive(Clone)]
struct NmemServer { db: DbHandle, tool_router: ToolRouter<Self> }

#[derive(Deserialize, JsonSchema)]
struct SearchParams {
    /// Text query to search observations
    query: String,
    /// Max results (default 20)
    #[serde(default)]
    limit: Option<usize>,
}

#[derive(Deserialize, JsonSchema)]
struct GetObsParams {
    /// Observation IDs to fetch
    ids: Vec<i64>,
}

#[tool_router]
impl NmemServer {
    fn new(db: DbHandle) -> Self {
        Self { db, tool_router: Self::tool_router() }
    }

    #[tool(description = "Search observations by query. Returns ranked index with IDs.")]
    async fn search(&self, #[tool(aggr)] p: Parameters<SearchParams>) -> Result<CallToolResult, ErrorData> {
        let db = self.db.lock().await;
        // ... FTS5 query, collect results, serialize ...
        Ok(CallToolResult::success(vec![Content::text(json_string)]))
    }

    #[tool(description = "Get full observation details by IDs.")]
    async fn get_observations(&self, #[tool(aggr)] p: Parameters<GetObsParams>) -> Result<CallToolResult, ErrorData> {
        if p.0.ids.is_empty() {
            return Ok(CallToolResult::error(vec![Content::text("ids must not be empty")]));
        }
        // ... SELECT WHERE id IN (...), collect, serialize ...
        Ok(CallToolResult::success(vec![Content::text(json_string)]))
    }

    #[tool(description = "Recent observations for session context. No parameters.")]
    async fn recent_context(&self) -> Result<CallToolResult, ErrorData> {
        // ... query last 50 observations ...
        Ok(CallToolResult::success(vec![Content::text(json_string)]))
    }
}

#[tool_handler]
impl ServerHandler for NmemServer {
    fn get_info(&self) -> ServerInfo {
        ServerInfo {
            instructions: Some("nmem: cross-session observation search and retrieval".into()),
            capabilities: ServerCapabilities::builder().enable_tools().build(),
            ..Default::default()
        }
    }
}
```

### Parameter Patterns

```rust
// 1. Aggregate (recommended) -- single struct via Parameters<T>
async fn search(&self, #[tool(aggr)] p: Parameters<SearchParams>) -> Result<CallToolResult, ErrorData>

// 2. Individual -- each annotated with #[tool(param)]
async fn get_by_id(&self, #[tool(param)] #[schemars(description = "Observation ID")] id: i64) -> ...

// 3. No params
async fn recent_context(&self) -> Result<CallToolResult, ErrorData>
```

### Return Types

```rust
Ok(CallToolResult::success(vec![Content::text("result")]))     // success
Ok(CallToolResult::error(vec![Content::text("not found")]))     // tool error (LLM sees it)
Err(ErrorData::new(ErrorCode::INTERNAL_ERROR, "bug", None::<()>)) // protocol error
```

**`Json<T>` wrapper:** auto-serializes response. `Err(String)` produces a tool error.

```rust
use rmcp::handler::server::wrapper::Json;

#[tool(description = "Search")]
async fn search(&self, #[tool(aggr)] p: Parameters<SearchParams>) -> Result<Json<Vec<SearchResult>>, String> {
    Ok(Json(self.db.search(&p.0.query)?))
}
```

---

## 4. Tool Parameters and Validation

Parameter structs derive `Deserialize` + `JsonSchema`. Doc comments become JSON Schema descriptions. Optional fields use `Option<T>`, defaults via `#[serde(default)]`, enums via `#[serde(rename_all)]`:

```rust
#[derive(Deserialize, JsonSchema)]
struct SearchParams {
    /// Text query to search observations
    query: String,
    /// Maximum results (default: 20)
    #[serde(default = "default_limit")]
    limit: Option<usize>,
    /// Filter by observation type
    obs_type: Option<ObsType>,
}
fn default_limit() -> Option<usize> { Some(20) }

#[derive(Deserialize, JsonSchema)]
#[serde(rename_all = "snake_case")]
enum ObsType { Decision, Bugfix, Feature, Refactor, Discovery }
```

Validate inside the handler. Return tool errors for bad input (not protocol errors). See the `get_observations` example in Section 3.

---

## 5. Server State

The server struct must be `Clone + Send + Sync + 'static` (rmcp requirement for tokio).

### Read-Only SQLite Connection

```rust
type DbHandle = Arc<Mutex<rusqlite::Connection>>;

impl NmemServer {
    fn new(db_path: &str) -> rusqlite::Result<Self> {
        let conn = rusqlite::Connection::open_with_flags(
            db_path,
            rusqlite::OpenFlags::SQLITE_OPEN_READ_ONLY | rusqlite::OpenFlags::SQLITE_OPEN_NO_MUTEX,
        )?;
        conn.pragma_update(None, "journal_mode", "wal")?;
        Ok(Self { db: Arc::new(Mutex::new(conn)), tool_router: Self::tool_router() })
    }
}
```

Access in tools: `let db = self.db.lock().await;`

**Alternative:** `tokio-rusqlite` offloads queries to a dedicated thread, avoiding Mutex across awaits. Use `self.db.call(move |conn| { ... }).await` pattern. See `tokio-rusqlite.md`.

---

## 6. Stdio Transport

Stdin carries JSON-RPC requests from Claude Code, stdout carries responses. Flow: `initialize` handshake -> `tools/list` -> `tools/call` (repeated) -> stdin closes -> process exits.

### Critical: stdout is MCP-only

Never write to stdout. All logging goes to stderr:

```rust
eprintln!("nmem-mcp: query took {}ms", elapsed);  // Good
// println!("debug");  // Bad -- corrupts MCP protocol

// With tracing:
tracing_subscriber::fmt()
    .with_writer(std::io::stderr)
    .with_env_filter("nmem_mcp=debug")
    .init();
```

**Lifecycle:** No daemon, no port, no PID file. One server per session. Claude Code spawns it, it serves queries, stdin closes when the session ends, process exits.

---

## 7. Error Handling

| Level | Type | Visible To | Use When |
|-------|------|------------|----------|
| Tool error | `CallToolResult::error(...)` | LLM sees error text | Bad input, not found, query failure |
| Protocol error | `Err(ErrorData { .. })` | MCP framework | Server bug, unrecoverable |

### Mapping rusqlite Errors

```rust
fn db_err(e: &impl std::fmt::Display) -> ErrorData {
    ErrorData::new(rmcp::model::ErrorCode::INTERNAL_ERROR, format!("db: {e}"), None::<()>)
}

// Use throughout: .map_err(|e| db_err(&e))?
// For expected failures like "not found", return CallToolResult::error instead
// For QueryReturnedNoRows, return tool error. For other rusqlite errors, return protocol error.
```

---

## 8. Testing

### Unit Testing Tool Handlers

Call tool methods directly with an in-memory DB -- no MCP transport needed:

```rust
#[cfg(test)]
mod tests {
    use super::*;

    fn test_db() -> DbHandle {
        let conn = rusqlite::Connection::open_in_memory().unwrap();
        conn.execute_batch("
            CREATE TABLE observations (id INTEGER PRIMARY KEY, obs_type TEXT, title TEXT, content TEXT);
            INSERT INTO observations VALUES (1, 'decision', 'Use rmcp', 'Chose rmcp');
        ").unwrap();
        Arc::new(Mutex::new(conn))
    }

    #[tokio::test]
    async fn search_finds_matching() {
        let server = NmemServer { db: test_db(), tool_router: NmemServer::tool_router() };
        let result = server.search(Parameters(SearchParams { query: "rmcp".into(), limit: Some(10) })).await.unwrap();
        assert!(!result.is_error.unwrap_or(false));
    }
}
```

### Integration Testing with Child Process

Spawn the binary and test via MCP client. Add `rmcp` with `client` + `transport-child-process` to `[dev-dependencies]`. Use `TokioChildProcess::new(child)` as transport, `().serve(transport)` to create a client, then `client.list_tools()` / `client.call_tool()` to verify.

---

## 9. Integration with Claude Code

### Plugin .mcp.json

```json
{
  "mcpServers": {
    "nmem-search": {
      "type": "stdio",
      "command": "${CLAUDE_PLUGIN_ROOT}/bin/nmem-mcp",
      "args": ["--db", "${HOME}/.nmem/nmem.db"],
      "env": { "RUST_LOG": "nmem_mcp=info" }
    }
  }
}
```

### CLI Registration

```bash
claude mcp add --transport stdio nmem-search -- /path/to/nmem-mcp --db ~/.nmem/nmem.db
```

### Tool Naming and Permissions

Tools appear as `mcp__nmem-search__<tool>` (e.g., `mcp__nmem-search__search`). Auto-allow in settings.json:

```json
{ "permissions": { "allow": ["mcp__nmem-search__search", "mcp__nmem-search__get_observations", "mcp__nmem-search__recent_context"] } }
```

---

## Quick Reference

**Macros:** `#[tool_router]` (impl block) -> `#[tool(description="...")]` (methods) -> `#[tool_handler]` (ServerHandler impl). Params: `#[tool(aggr)]` for struct, `#[tool(param)]` for individual.

**Key types:** `CallToolResult` / `Content` / `ErrorData` from `rmcp::model`, `Parameters<T>` / `Json<T>` from `rmcp::handler::server::wrapper`, `ToolRouter<S>` from `rmcp::handler::server::tool`.

**Annotations:** `#[tool(description = "...", annotations(read_only_hint = true, open_world_hint = false))]`
