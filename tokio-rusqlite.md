# tokio-rusqlite Reference

Async wrapper for rusqlite. Spawns a dedicated background thread per connection, sends closures via channel. Version: **0.6.0**.

Docs: <https://docs.rs/tokio-rusqlite/0.6.0/tokio_rusqlite/>

---

## 1. Installation

```toml
[dependencies]
tokio-rusqlite = "0.6"
tokio = { version = "1", features = ["rt-multi-thread", "macros"] }
rusqlite = { version = "0.38", features = ["bundled"] }
```

---

## 2. Why tokio-rusqlite

- rusqlite `Connection` is `Send` but **not** `Sync` (wraps `RefCell`)
- SQLite operations are blocking I/O -- starves the async executor if called directly
- tokio-rusqlite spawns a dedicated OS thread, sends `FnOnce` closures via channel
- The async handle is `Clone + Send + Sync` -- safe to share across tasks
- Alternative: `tokio::task::spawn_blocking` per task (no connection reuse, higher overhead)

---

## 3. Opening a Connection

```rust
use tokio_rusqlite::Connection;

let conn = Connection::open("nmem.db").await?;             // read-write, creates if missing
let conn = Connection::open_in_memory().await?;             // testing
let conn = Connection::open_with_flags(                     // read-only
    "nmem.db",
    rusqlite::OpenFlags::SQLITE_OPEN_READ_ONLY | rusqlite::OpenFlags::SQLITE_OPEN_NO_MUTEX,
).await?;
```

The handle is lightweight. Cloning shares the same background connection.

---

## 4. The `.call()` Pattern

Closure receives `&mut rusqlite::Connection`. Return `R` must be `Send + 'static`.

```rust
// Query
let count: i64 = conn.call(|conn| {
    conn.query_row("SELECT count(*) FROM observations", [], |row| row.get(0))
        .map_err(|e| e.into())
}).await?;

// Insert + get rowid
let id = conn.call(|conn| {
    conn.execute(
        "INSERT INTO observations (session_id, obs_type, content) VALUES (?1, ?2, ?3)",
        rusqlite::params![sid, otype, content],
    )?;
    Ok(conn.last_insert_rowid())
}).await?;

// Collect results (data must not borrow the connection)
let rows: Vec<(i64, String)> = conn.call(|conn| {
    let mut stmt = conn.prepare_cached(
        "SELECT id, content FROM observations WHERE obs_type = ?1 LIMIT ?2"
    )?;
    stmt.query_map(rusqlite::params![otype, limit], |row| {
        Ok((row.get(0)?, row.get(1)?))
    })?.collect::<Result<Vec<_>, _>>().map_err(|e| e.into())
}).await?;

// Transaction
conn.call(|conn| {
    let tx = conn.transaction()?;
    tx.execute("INSERT INTO sessions (started_at) VALUES (unixepoch())", [])?;
    let sid = tx.last_insert_rowid();
    tx.execute("INSERT INTO observations (session_id, content) VALUES (?1, ?2)",
        rusqlite::params![sid, "init"])?;
    tx.commit()?;
    Ok(sid)
}).await?;
```

`call_unwrap` skips error wrapping but panics if the connection is closed. Avoid in production.

---

## 5. Connection Initialization

PRAGMAs + schema in the first `.call()`:

```rust
async fn open_db(path: &str) -> tokio_rusqlite::Result<Connection> {
    let conn = Connection::open(path).await?;
    conn.call(|conn| {
        conn.execute_batch("
            PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL;
            PRAGMA busy_timeout=5000; PRAGMA foreign_keys=ON; PRAGMA cache_size=-64000;
            CREATE TABLE IF NOT EXISTS sessions (
                id INTEGER PRIMARY KEY, started_at INTEGER NOT NULL DEFAULT (unixepoch()), summary TEXT);
            CREATE TABLE IF NOT EXISTS observations (
                id INTEGER PRIMARY KEY, session_id INTEGER REFERENCES sessions(id),
                obs_type TEXT NOT NULL, content TEXT NOT NULL, created_at INTEGER DEFAULT (unixepoch()));
            CREATE INDEX IF NOT EXISTS idx_obs_session ON observations(session_id);
        ")?; Ok(())
    }).await?;
    Ok(conn)
}
```

---

## 6. Multiple Connections (Reader/Writer Pattern)

One writer + N readers via separate `Connection` handles. WAL required. Both are `Clone` for passing into tasks.

```rust
// Writer: read-write with WAL
let writer = Connection::open(path).await?;
writer.call(|c| {
    c.execute_batch("PRAGMA journal_mode=WAL; PRAGMA synchronous=NORMAL; PRAGMA busy_timeout=5000;")?;
    Ok(())
}).await?;

// Reader: read-only (won't block writer, won't trigger SQLITE_BUSY on reads)
let reader = Connection::open_with_flags(
    path, rusqlite::OpenFlags::SQLITE_OPEN_READ_ONLY | rusqlite::OpenFlags::SQLITE_OPEN_NO_MUTEX,
).await?;
reader.call(|c| { c.execute_batch("PRAGMA busy_timeout=5000;")?; Ok(()) }).await?;
```

---

## 7. Error Handling

```rust
#[non_exhaustive]
pub enum Error {
    ConnectionClosed,                         // call() after close
    Close((Connection, rusqlite::Error)),      // close() failed (retryable)
    Rusqlite(rusqlite::Error),                // forwarded from closure
    Other(Box<dyn std::error::Error + Send + Sync>),
}
```

Match at the boundary:

```rust
match conn.call(|c| { /* ... */ }).await {
    Ok(val) => val,
    Err(tokio_rusqlite::Error::Rusqlite(e)) => {
        if let rusqlite::Error::SqliteFailure(err, _) = &e {
            if err.code == rusqlite::ErrorCode::DatabaseBusy { /* retry */ }
        }
        return Err(e.into());
    }
    Err(tokio_rusqlite::Error::ConnectionClosed) => panic!("db connection lost"),
    Err(e) => return Err(e.into()),
}
```

`Error` implements `std::error::Error` -- works with `anyhow`/`thiserror` via `?`.

---

## 8. Transactions in Async Context

Transactions **must** complete within a single `.call()`. Cannot span multiple calls.

```rust
conn.call(|conn| {
    let tx = conn.transaction_with_behavior(rusqlite::TransactionBehavior::Immediate)?;
    for obs in &batch {
        tx.execute("INSERT INTO observations (session_id, obs_type, content) VALUES (?1, ?2, ?3)",
            rusqlite::params![obs.session_id, obs.obs_type, obs.content])?;
    }
    tx.commit()?;
    Ok(())
}).await?;
```

Collect all data async first, then submit in one `.call()` with a transaction.

---

## 9. Graceful Shutdown

```rust
conn.close().await?;  // waits for pending ops, closes connection

// On failure, error contains the handle for retry
if let Err(tokio_rusqlite::Error::Close((conn, err))) = conn.close().await {
    eprintln!("close failed: {err}");
    conn.close().await?;
}
```

Prefer explicit `close().await` over drop in daemon shutdown to ensure WAL checkpoint.

---

## 10. Alternative: spawn_blocking + r2d2

```rust
let manager = r2d2_sqlite::SqliteConnectionManager::file("nmem.db")
    .with_init(|c| c.execute_batch("PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;"));
let pool = r2d2::Pool::builder().max_size(4).build(manager)?;

let p = pool.clone();
let count = tokio::task::spawn_blocking(move || {
    p.get()?.query_row("SELECT count(*) FROM observations", [], |r| r.get::<_, i64>(0))
}).await??;
```

Trade-offs: r2d2 adds pool overhead + thread-per-call, better for web servers with many concurrent handlers. tokio-rusqlite reuses a single thread per connection -- better for daemons like nmem.

---

## 11. Patterns for nmem

Daemon pattern: one write `Connection` for ingestion, one read-only `Connection` for MCP queries.

### Write batching with mpsc channel

```rust
use tokio_rusqlite::Connection;
use tokio::sync::mpsc;

struct Obs { session_id: i64, obs_type: String, content: String }

async fn write_loop(conn: Connection, mut rx: mpsc::Receiver<Obs>) -> anyhow::Result<()> {
    let mut batch = Vec::with_capacity(64);
    let mut tick = tokio::time::interval(std::time::Duration::from_millis(500));
    loop {
        tokio::select! {
            Some(obs) = rx.recv() => {
                batch.push(obs);
                if batch.len() >= 50 { flush(&conn, &mut batch).await?; }
            }
            _ = tick.tick() => {
                if !batch.is_empty() { flush(&conn, &mut batch).await?; }
            }
        }
    }
}

async fn flush(conn: &Connection, batch: &mut Vec<Obs>) -> anyhow::Result<()> {
    let items: Vec<Obs> = batch.drain(..).collect();
    conn.call(move |c| {
        let tx = c.transaction_with_behavior(rusqlite::TransactionBehavior::Immediate)?;
        { let mut s = tx.prepare_cached(
            "INSERT INTO observations (session_id, obs_type, content) VALUES (?1, ?2, ?3)")?;
          for o in &items { s.execute(rusqlite::params![o.session_id, o.obs_type, o.content])?; }
        }
        tx.commit()?; Ok(())
    }).await?; Ok(())
}
```

### Health check + read query

```rust
async fn health(conn: &Connection) -> bool {
    conn.call(|c| { c.execute_batch("SELECT 1")?; Ok(()) }).await.is_ok()
}

async fn search(reader: &Connection, q: String, limit: i64) -> anyhow::Result<Vec<(i64, String)>> {
    reader.call(move |c| {
        let mut s = c.prepare_cached(
            "SELECT id, content FROM observations WHERE content LIKE '%'||?1||'%' ORDER BY created_at DESC LIMIT ?2")?;
        s.query_map(rusqlite::params![q, limit], |r| Ok((r.get(0)?, r.get(1)?)))?.
            collect::<Result<Vec<_>,_>>().map_err(|e| e.into())
    }).await.map_err(|e| e.into())
}
```

---

## Quick Reference

| Operation | Pattern |
|---|---|
| Open | `Connection::open(path).await?` |
| Query | `conn.call(\|c\| c.query_row(...)).await?` |
| Insert | `conn.call(\|c\| { c.execute(...)?; Ok(c.last_insert_rowid()) }).await?` |
| Transaction | `conn.call(\|c\| { let tx = c.transaction()?; ...; tx.commit()?; Ok(()) }).await?` |
| Batch exec | `conn.call(\|c\| c.execute_batch(...)).await?` |
| Close | `conn.close().await?` |
| Health | `conn.call(\|c\| c.execute_batch("SELECT 1")).await.is_ok()` |
