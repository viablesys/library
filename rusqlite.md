# rusqlite Reference

Ergonomic Rust bindings for SQLite. Current version: **0.38.0** (bundled SQLite 3.51.1).

Docs: <https://docs.rs/rusqlite/latest/rusqlite/>
Repo: <https://github.com/rusqlite/rusqlite>

---

## 1. Installation

### Minimal

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled"] }
```

### Full-featured

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled-full"] }
```

`bundled-full` = `bundled` + `modern-full` (enables almost everything).

### Feature Flags (45 total, 2 default: `cache`, `hashlink`)

| Feature | Purpose | Implies |
|---------|---------|---------|
| `bundled` | Bundle SQLite source, compile from C | `modern_sqlite` |
| `bundled-full` | Bundle + all optional features | `bundled`, `modern-full` |
| `modern_sqlite` | Use bundled bindings for latest SQLite API | `libsqlite3-sys/bundled_bindings` |
| `modern-full` | Enable 24+ optional features without bundling | array, backup, blob, chrono, collation, column_decltype, csvtab, extra_check, functions, hooks, i128_blob, jiff, limits, load_extension, modern_sqlite, serde_json, serialize, series, time, trace, unlock_notify, url, uuid, vtab, window |
| `backup` | Online backup API (`sqlite3_backup`) | -- |
| `blob` | Incremental BLOB I/O | -- |
| `hooks` | Commit/rollback/update notification callbacks | -- |
| `preupdate_hook` | Pre-update change notifications | `hooks` |
| `session` | Session extension (changesets) | `hooks`, requires `buildtime_bindgen` |
| `limits` | Runtime limit configuration | -- |
| `load_extension` | Load dynamic `.so`/`.dll` extensions | -- |
| `loadable_extension` | Write loadable extensions in Rust | -- |
| `vtab` | Virtual table support (`VTab`, `VTabCursor`) | -- |
| `array` | `rarray()` table-valued function | `vtab` |
| `series` | `generate_series()` TVF | `vtab` |
| `csvtab` | CSV virtual table | `vtab` |
| `column_decltype` | Column declared type info | -- |
| `column_metadata` | Column origin/constraint info | -- |
| `extra_check` | Additional safety checks | -- |
| `unlock_notify` | Unlock notification for concurrent access | -- |
| `window` | Window function support | `functions` |
| `functions` | User-defined SQL functions | -- |
| `collation` | Custom collation sequences | -- |
| `trace` | Query tracing/profiling | -- |
| `serialize` | Serialize/deserialize database to bytes | -- |
| `buildtime_bindgen` | Generate bindings at build time | -- |
| `wasm32-wasi-vfs` | WASM VFS support | -- |
| `serde_json` | `ToSql`/`FromSql` for `serde_json::Value` | -- |
| `chrono` | `ToSql`/`FromSql` for `chrono` types | -- |
| `time` | `ToSql`/`FromSql` for `time` crate types | -- |
| `jiff` | `ToSql`/`FromSql` for `jiff` types | -- |
| `uuid` | `ToSql`/`FromSql` for `uuid::Uuid` | -- |
| `url` | `ToSql`/`FromSql` for `url::Url` | -- |
| `i128_blob` | Store i128 as 16-byte BLOB | -- |
| `fallible_uint` | Strict unsigned integer conversion | -- |
| `rusqlite-macros` | `prepare_and_bind!`, `prepare_cached_and_bind!` | -- |
| `sqlcipher` | Link against SQLCipher instead of SQLite | -- |
| `bundled-sqlcipher` | Bundle SQLCipher source | `bundled` |
| `bundled-sqlcipher-vendored-openssl` | Bundle SQLCipher + vendored OpenSSL | `bundled-sqlcipher` |
| `bundled-windows` | Bundle only on Windows | -- |
| `in_gecko` | Firefox/Gecko integration | `modern_sqlite` |
| `with-asan` | Address sanitizer support | -- |
| `csv` | CSV parsing dependency | -- |
| `cache` | **(default)** Prepared statement LRU cache | `hashlink` |

---

## 2. Connection

### Opening

```rust
use rusqlite::{Connection, OpenFlags, Result};

// File database (creates if missing)
let conn = Connection::open("app.db")?;

// In-memory
let conn = Connection::open_in_memory()?;

// Custom flags
let conn = Connection::open_with_flags(
    "app.db",
    OpenFlags::SQLITE_OPEN_READ_WRITE
        | OpenFlags::SQLITE_OPEN_CREATE
        | OpenFlags::SQLITE_OPEN_NO_MUTEX,
)?;

// Read-only
let conn = Connection::open_with_flags(
    "app.db",
    OpenFlags::SQLITE_OPEN_READ_ONLY | OpenFlags::SQLITE_OPEN_NO_MUTEX,
)?;
```

Default flags: `SQLITE_OPEN_READ_WRITE | SQLITE_OPEN_CREATE | SQLITE_OPEN_URI | SQLITE_OPEN_NO_MUTEX`.

### Thread Safety

`Connection` is `Send` but **not** `Sync`. It wraps `RefCell<InnerConnection>`, so it cannot be shared across threads. Each thread needs its own connection, or use a pool.

### Connection State

```rust
conn.is_autocommit()       // true when no transaction active
conn.last_insert_rowid()   // rowid of most recent INSERT
conn.changes()             // rows modified by last statement
conn.path()                // Some("/path/to/db") or None for in-memory
conn.is_busy()             // true if statements need reset
conn.close()?;             // explicit close (rolls back on error)
```

### Interrupt Handle

```rust
let handle = conn.interrupt_handle();
// From another thread:
handle.interrupt();  // cancels long-running query
```

---

## 3. WAL Mode

Write-Ahead Logging enables concurrent readers alongside a single writer. Changes go to a WAL file first, then checkpoint to the main database.

### Enable WAL

```rust
// Best done immediately after opening
conn.pragma_update(None, "journal_mode", "WAL")?;
conn.pragma_update(None, "synchronous", "NORMAL")?;  // safe in WAL mode
conn.pragma_update(None, "busy_timeout", "5000")?;    // 5s retry on SQLITE_BUSY
```

WAL mode is **persistent** -- it survives connection close/reopen.

### Advantages

- Readers never block writers, writers never block readers
- Multiple concurrent readers
- Faster writes (sequential WAL appends vs random journal writes)
- `synchronous=NORMAL` is corruption-safe in WAL (not in DELETE journal mode)

### Checkpoint

Transfers WAL content back to the main database file.

```rust
// Auto-checkpoint threshold (default: 1000 pages)
conn.pragma_update(None, "wal_autocheckpoint", "1000")?;

// Manual checkpoint modes
conn.execute_batch("PRAGMA wal_checkpoint(PASSIVE)")?;   // non-blocking
conn.execute_batch("PRAGMA wal_checkpoint(FULL)")?;       // waits for readers
conn.execute_batch("PRAGMA wal_checkpoint(RESTART)")?;    // resets WAL file
conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE)")?;   // truncates WAL to zero
```

### WAL Hook (requires `hooks` feature)

```rust
conn.wal_hook(Some(|_db_name: &str, page_count: i32| {
    if page_count > 1000 {
        // trigger checkpoint
    }
}));
```

### Journal Size Limit

```rust
conn.pragma_update(None, "journal_size_limit", "67108864")?;  // 64 MB
```

### Pitfalls

- Long-running read transactions prevent checkpoint progress, causing WAL growth
- Only one writer at a time (SQLITE_BUSY if contended)
- WAL does not work over network filesystems
- Ensure periodic "reader gaps" for checkpointing

---

## 4. Executing SQL

### execute -- single statement, no rows returned

```rust
// Returns number of rows modified
let changes = conn.execute(
    "INSERT INTO users (name, email) VALUES (?1, ?2)",
    ("Alice", "alice@example.com"),
)?;
```

### execute_batch -- multiple statements, no parameters

```rust
conn.execute_batch(
    "CREATE TABLE IF NOT EXISTS users (
        id    INTEGER PRIMARY KEY,
        name  TEXT NOT NULL,
        email TEXT UNIQUE
    );
    CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);"
)?;
```

### prepare -- reusable prepared statement

```rust
let mut stmt = conn.prepare("SELECT id, name FROM users WHERE email = ?1")?;
```

### prepare_cached -- LRU-cached prepared statement

```rust
// Reuses previously compiled statement if available
let mut stmt = conn.prepare_cached("SELECT id, name FROM users WHERE email = ?1")?;
// Statement returned to cache on drop

// Configure cache size (default: 16)
conn.set_prepared_statement_cache_capacity(64);
conn.flush_prepared_statement_cache();
```

### query_map -- iterate rows

```rust
let mut stmt = conn.prepare("SELECT id, name, email FROM users")?;
let users = stmt.query_map([], |row| {
    Ok(User {
        id: row.get(0)?,
        name: row.get(1)?,
        email: row.get(2)?,
    })
})?;

for user in users {
    let user = user?;
    println!("{}: {}", user.id, user.name);
}
```

### query_row -- exactly one row

```rust
let name: String = conn.query_row(
    "SELECT name FROM users WHERE id = ?1",
    [42],
    |row| row.get(0),
)?;
// Returns Err(QueryReturnedNoRows) if no match
```

### Positional Parameters (?N)

```rust
conn.execute(
    "INSERT INTO users (name, email) VALUES (?1, ?2)",
    params!["Alice", "alice@example.com"],
)?;

// Tuple syntax (no macro needed)
conn.execute(
    "UPDATE users SET name = ?1 WHERE id = ?2",
    ("Bob", 5),
)?;

// Slice syntax
conn.execute("DELETE FROM users WHERE id = ?1", [42])?;
```

### Named Parameters (:name)

```rust
use rusqlite::named_params;

conn.execute(
    "INSERT INTO users (name, email) VALUES (:name, :email)",
    named_params! {
        ":name": "Alice",
        ":email": "alice@example.com",
    },
)?;
```

---

## 5. Mapping Rows

### Row Methods

```rust
// By column index (0-based)
let id: i64 = row.get(0)?;
let name: String = row.get(1)?;
let data: Option<Vec<u8>> = row.get(2)?;  // NULL-safe

// By column name
let name: String = row.get("name")?;

// Unwrap variant (panics on error -- avoid in production)
let id: i64 = row.get_unwrap(0);
```

### Custom Mapping Closure

```rust
struct User {
    id: i64,
    name: String,
    email: Option<String>,
}

let mut stmt = conn.prepare("SELECT id, name, email FROM users")?;
let users: Vec<User> = stmt
    .query_map([], |row| {
        Ok(User {
            id: row.get(0)?,
            name: row.get(1)?,
            email: row.get(2)?,
        })
    })?
    .collect::<Result<Vec<_>, _>>()?;
```

### query_and_then -- fallible mapping

```rust
let users = stmt.query_and_then([], |row| -> anyhow::Result<User> {
    Ok(User {
        id: row.get(0)?,
        name: row.get::<_, String>(1)?,
        email: validate_email(row.get(2)?)?,  // can return non-rusqlite errors
    })
})?;
```

---

## 6. Transactions

### Basic Transaction

```rust
let tx = conn.transaction()?;  // BEGIN DEFERRED
tx.execute("INSERT INTO users (name) VALUES (?1)", ["Alice"])?;
tx.execute("INSERT INTO users (name) VALUES (?1)", ["Bob"])?;
tx.commit()?;
// If tx is dropped without commit(), it rolls back automatically
```

### Transaction Behavior

```rust
use rusqlite::TransactionBehavior;

// DEFERRED -- acquire locks lazily (default)
let tx = conn.transaction()?;

// IMMEDIATE -- acquire RESERVED lock now (prevents other writers)
let tx = conn.transaction_with_behavior(TransactionBehavior::Immediate)?;

// EXCLUSIVE -- acquire EXCLUSIVE lock now (prevents readers too in non-WAL)
let tx = conn.transaction_with_behavior(TransactionBehavior::Exclusive)?;
```

Use `Immediate` for write transactions in multi-connection scenarios to fail fast on contention.

### Savepoints (Nested Transactions)

```rust
let tx = conn.transaction()?;
tx.execute("INSERT INTO users (name) VALUES (?1)", ["Alice"])?;

{
    let sp = tx.savepoint()?;
    sp.execute("INSERT INTO users (name) VALUES (?1)", ["Bob"])?;
    // sp dropped here -- rolls back Bob's insert
}

tx.commit()?;  // only Alice committed
```

Named savepoints:

```rust
let sp = tx.savepoint_with_name("retry_point")?;
```

### Drop Behavior

```rust
use rusqlite::DropBehavior;

let mut tx = conn.transaction()?;
tx.set_drop_behavior(DropBehavior::Commit);  // auto-commit on drop
// Default is DropBehavior::Rollback
```

### Pattern: Retry on SQLITE_BUSY

```rust
fn with_retry<F, T>(conn: &mut Connection, f: F) -> Result<T>
where
    F: Fn(&Transaction) -> Result<T>,
{
    loop {
        let tx = conn.transaction_with_behavior(TransactionBehavior::Immediate)?;
        match f(&tx) {
            Ok(val) => {
                tx.commit()?;
                return Ok(val);
            }
            Err(rusqlite::Error::SqliteFailure(err, _))
                if err.code == rusqlite::ErrorCode::DatabaseBusy =>
            {
                std::thread::sleep(std::time::Duration::from_millis(10));
                continue;
            }
            Err(e) => return Err(e),
        }
    }
}
```

---

## 7. Types

### Rust to SQLite Mapping

| Rust Type | SQLite Type | Notes |
|-----------|-------------|-------|
| `bool` | INTEGER | 0/1 |
| `i8`, `i16`, `i32`, `i64` | INTEGER | Stored as i64 |
| `u8`, `u16`, `u32` | INTEGER | Stored as i64 |
| `f32`, `f64` | REAL | Stored as f64 |
| `String`, `&str` | TEXT | UTF-8 |
| `Vec<u8>`, `&[u8]` | BLOB | Raw bytes |
| `Option<T>` | NULL or T | None maps to NULL |
| `()` | NULL | Unit type |
| `serde_json::Value` | TEXT/INTEGER/REAL/NULL | With `serde_json` feature |
| `chrono::NaiveDateTime` | TEXT | With `chrono` feature |
| `chrono::DateTime<Utc>` | TEXT | With `chrono` feature |
| `time::OffsetDateTime` | TEXT | With `time` feature |
| `uuid::Uuid` | BLOB (16 bytes) | With `uuid` feature |
| `url::Url` | TEXT | With `url` feature |
| `i128` | BLOB (16 bytes) | With `i128_blob` feature |

### Number Conversion Rules

- INTEGER to smaller int: `IntegralValueOutOfRange` if value does not fit
- REAL to int: always `InvalidColumnType` (no implicit truncation)
- INTEGER to float: `as` cast (never fails)
- REAL to float: `as` cast (never fails)

### ToSql Trait

```rust
use rusqlite::types::{ToSql, ToSqlOutput};

impl ToSql for MyType {
    fn to_sql(&self) -> rusqlite::Result<ToSqlOutput<'_>> {
        Ok(ToSqlOutput::from(self.0.to_string()))
    }
}
```

### FromSql Trait

```rust
use rusqlite::types::{FromSql, FromSqlError, FromSqlResult, ValueRef};

impl FromSql for MyType {
    fn column_result(value: ValueRef<'_>) -> FromSqlResult<Self> {
        match value {
            ValueRef::Text(s) => {
                let s = std::str::from_utf8(s).map_err(|e| FromSqlError::Other(e.into()))?;
                s.parse().map(MyType).map_err(|e| FromSqlError::Other(e.into()))
            }
            _ => Err(FromSqlError::InvalidType),
        }
    }
}
```

### Custom Type Example: Unix Timestamp

```rust
use rusqlite::types::{FromSql, FromSqlResult, ToSql, ToSqlOutput, ValueRef};
use std::time::{SystemTime, UNIX_EPOCH};

struct Timestamp(SystemTime);

impl ToSql for Timestamp {
    fn to_sql(&self) -> rusqlite::Result<ToSqlOutput<'_>> {
        let secs = self.0.duration_since(UNIX_EPOCH)
            .map_err(|e| rusqlite::Error::ToSqlConversionFailure(e.into()))?
            .as_secs();
        Ok(ToSqlOutput::from(secs as i64))
    }
}

impl FromSql for Timestamp {
    fn column_result(value: ValueRef<'_>) -> FromSqlResult<Self> {
        let secs = i64::column_result(value)? as u64;
        Ok(Timestamp(UNIX_EPOCH + std::time::Duration::from_secs(secs)))
    }
}
```

### serde_json::Value Mapping (with `serde_json` feature)

```rust
// ToSql: Null->NULL, Number(i64)->INTEGER, Number(f64)->REAL, other->TEXT (serialized)
// FromSql: Text/Blob->deserialized, Integer->Number, Real->Number, Null->Null
let val = serde_json::json!({"key": "value"});
conn.execute("INSERT INTO data (json_col) VALUES (?1)", [&val])?;

let result: serde_json::Value = conn.query_row(
    "SELECT json_col FROM data WHERE id = ?1", [1], |row| row.get(0),
)?;
```

---

## 8. Migrations

### rusqlite_migration

```toml
[dependencies]
rusqlite_migration = "2.3"
rusqlite = { version = "0.38.0", features = ["bundled"] }
```

Tracks schema version via SQLite's `user_version` (integer at fixed file offset, no extra tables).

### Inline Migrations

```rust
use rusqlite_migration::{Migrations, M};

const MIGRATIONS: Migrations<'static> = Migrations::from_slice(&[
    M::up("CREATE TABLE users (
        id    INTEGER PRIMARY KEY,
        name  TEXT NOT NULL,
        email TEXT UNIQUE
    );"),
    M::up("ALTER TABLE users ADD COLUMN created_at TEXT DEFAULT (datetime('now'));"),
    M::up("CREATE TABLE posts (
        id      INTEGER PRIMARY KEY,
        user_id INTEGER NOT NULL REFERENCES users(id),
        title   TEXT NOT NULL,
        body    TEXT NOT NULL
    );"),
]);

fn main() -> rusqlite::Result<()> {
    let mut conn = rusqlite::Connection::open("app.db")?;
    MIGRATIONS.to_latest(&mut conn).unwrap();
    Ok(())
}
```

### With include_str!

```rust
const MIGRATIONS: Migrations<'static> = Migrations::from_slice(&[
    M::up(include_str!("../migrations/001_users.sql")),
    M::up(include_str!("../migrations/002_posts.sql")),
]);
```

### From Directory (requires `from-directory` feature)

```toml
rusqlite_migration = { version = "2.3", features = ["from-directory"] }
```

Loads `*.sql` files from a directory.

### Downward Migrations

```rust
M::up("CREATE TABLE users (id INTEGER PRIMARY KEY, name TEXT);")
    .down("DROP TABLE users;"),
```

### Testing

```rust
#[test]
fn migrations_valid() {
    assert!(MIGRATIONS.validate().is_ok());
}
```

### Alternative: refinery

```toml
[dependencies]
refinery = { version = "0.8", features = ["rusqlite"] }
```

File-based migrations with CLI support (`refinery_cli`).

---

## 9. FTS5 (Full-Text Search)

Requires `bundled` feature (FTS5 is compiled into bundled SQLite).

### Create FTS5 Table

```rust
conn.execute_batch(
    "CREATE VIRTUAL TABLE docs_fts USING fts5(title, body);"
)?;
```

No types, constraints, or PRIMARY KEY allowed on FTS5 columns.

### Insert Data

```rust
conn.execute(
    "INSERT INTO docs_fts (title, body) VALUES (?1, ?2)",
    params!["Rust Guide", "Rust is a systems programming language focused on safety"],
)?;
```

### MATCH Queries

```rust
let mut stmt = conn.prepare(
    "SELECT title, body FROM docs_fts WHERE docs_fts MATCH ?1 ORDER BY rank"
)?;
let results = stmt.query_map(["rust AND safety"], |row| {
    Ok((row.get::<_, String>(0)?, row.get::<_, String>(1)?))
})?;
```

Boolean operators: `AND`, `OR`, `NOT`, phrase: `"exact phrase"`, prefix: `rust*`, column filter: `title:rust`.

### Ranking with bm25()

```rust
// Lower bm25 score = better match (order ASC)
let mut stmt = conn.prepare(
    "SELECT title, bm25(docs_fts) AS score
     FROM docs_fts
     WHERE docs_fts MATCH ?1
     ORDER BY score"
)?;

// Weight columns: bm25(table, w0, w1, ...) -- higher weight = more important
let mut stmt = conn.prepare(
    "SELECT title, bm25(docs_fts, 10.0, 1.0) AS score
     FROM docs_fts
     WHERE docs_fts MATCH ?1
     ORDER BY score"
)?;
```

### highlight()

Returns full column value with matched terms wrapped in markers.

```rust
let mut stmt = conn.prepare(
    "SELECT highlight(docs_fts, 1, '<b>', '</b>')
     FROM docs_fts
     WHERE docs_fts MATCH ?1"
)?;
```

### snippet()

Returns a fragment (~32 words) around the match with highlighting.

```rust
let mut stmt = conn.prepare(
    "SELECT snippet(docs_fts, 1, '<b>', '</b>', '...', 32)
     FROM docs_fts
     WHERE docs_fts MATCH ?1"
)?;
```

Args: `snippet(table, column_idx, open_marker, close_marker, ellipsis, max_tokens)`.

### External Content Tables

Index a regular table without duplicating data:

```rust
conn.execute_batch("
    CREATE TABLE articles (
        id    INTEGER PRIMARY KEY,
        title TEXT NOT NULL,
        body  TEXT NOT NULL
    );

    CREATE VIRTUAL TABLE articles_fts USING fts5(
        title, body,
        content='articles',
        content_rowid='id'
    );
")?;
```

### Sync Triggers

Keep the FTS index in sync with the content table:

```rust
conn.execute_batch("
    CREATE TRIGGER articles_ai AFTER INSERT ON articles BEGIN
        INSERT INTO articles_fts(rowid, title, body)
        VALUES (new.id, new.title, new.body);
    END;

    CREATE TRIGGER articles_ad AFTER DELETE ON articles BEGIN
        INSERT INTO articles_fts(articles_fts, rowid, title, body)
        VALUES ('delete', old.id, old.title, old.body);
    END;

    CREATE TRIGGER articles_au AFTER UPDATE ON articles BEGIN
        INSERT INTO articles_fts(articles_fts, rowid, title, body)
        VALUES ('delete', old.id, old.title, old.body);
        INSERT INTO articles_fts(rowid, title, body)
        VALUES (new.id, new.title, new.body);
    END;
")?;
```

Always add INSERT, DELETE, and UPDATE triggers together. Missing delete triggers cause stale search results.

### Rebuild Index

```rust
conn.execute_batch("INSERT INTO articles_fts(articles_fts) VALUES('rebuild');")?;
```

### Safety: Sanitize User Input

Raw user input in MATCH can cause errors on special characters (`*`, `"`, `(`, etc.). Escape or validate before passing to MATCH.

---

## 10. JSON

SQLite JSON functions work via standard SQL. Requires `bundled` (JSON1 included) or system SQLite with JSON support.

### Setup

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled", "serde_json"] }
serde_json = "1"
```

### json_extract()

```rust
let name: String = conn.query_row(
    "SELECT json_extract(data, '$.name') FROM items WHERE id = ?1",
    [1],
    |row| row.get(0),
)?;

// Multiple paths return a JSON array
let result: String = conn.query_row(
    "SELECT json_extract(data, '$.name', '$.age') FROM items WHERE id = ?1",
    [1],
    |row| row.get(0),
)?;
```

### json_each() -- table-valued function

```rust
let mut stmt = conn.prepare(
    "SELECT je.value
     FROM items, json_each(json_extract(items.data, '$.tags')) AS je
     WHERE items.id = ?1"
)?;
let tags: Vec<String> = stmt
    .query_map([1], |row| row.get(0))?
    .collect::<Result<_, _>>()?;
```

### json_group_array() -- aggregate into JSON array

```rust
let result: String = conn.query_row(
    "SELECT json_group_array(json_extract(data, '$.name'))
     FROM items
     WHERE json_extract(data, '$.active') = 1",
    [],
    |row| row.get(0),
)?;
```

### Storing serde_json::Value Directly

```rust
use serde_json::json;

let data = json!({"name": "Alice", "tags": ["rust", "sqlite"]});
conn.execute("INSERT INTO items (data) VALUES (?1)", [&data])?;

let result: serde_json::Value = conn.query_row(
    "SELECT data FROM items WHERE id = ?1", [1], |row| row.get(0),
)?;
```

### Other JSON Functions

```sql
json_insert(json, path, value)       -- insert if not exists
json_replace(json, path, value)      -- replace if exists
json_set(json, path, value)          -- insert or replace
json_remove(json, path)              -- remove key
json_type(json, path)                -- returns type name
json_valid(json)                     -- returns 1 if valid JSON
json_array(val1, val2, ...)          -- create JSON array
json_object(key1, val1, ...)         -- create JSON object
json_group_object(key, value)        -- aggregate into JSON object
json_patch(target, patch)            -- RFC 7396 merge patch
```

### JSONB (SQLite 3.45+)

Binary JSON format, more efficient for manipulation. Use `jsonb()` to convert TEXT to JSONB, `json()` to convert back.

For Rust-native JSONB handling, see the `serde-sqlite-jsonb` crate.

---

## 11. Performance

### Recommended PRAGMAs

```rust
fn configure_performance(conn: &Connection) -> rusqlite::Result<()> {
    conn.execute_batch("
        PRAGMA journal_mode = WAL;
        PRAGMA synchronous = NORMAL;
        PRAGMA cache_size = -64000;        -- 64 MB (negative = KB)
        PRAGMA mmap_size = 30000000000;    -- ~30 GB virtual memory mapping
        PRAGMA temp_store = MEMORY;
        PRAGMA foreign_keys = ON;
    ")?;
    Ok(())
}
```

| PRAGMA | Default | Recommended | Notes |
|--------|---------|-------------|-------|
| `journal_mode` | DELETE | WAL | Concurrent readers, faster writes |
| `synchronous` | FULL | NORMAL | Safe in WAL, 1 fsync per checkpoint instead of every write |
| `cache_size` | -2000 (2 MB) | -64000 (64 MB) | Negative = KB. Boost for I/O heavy workloads |
| `mmap_size` | 0 | Match DB size | OS manages pages, fewer syscalls. Best for read-heavy |
| `temp_store` | DEFAULT | MEMORY | Temp tables/indices in RAM |
| `busy_timeout` | 0 | 5000 | Milliseconds to retry on SQLITE_BUSY |
| `foreign_keys` | OFF | ON | Must set per-connection |
| `auto_vacuum` | NONE | INCREMENTAL | Reclaim space without full VACUUM |

### Query Optimization

```rust
// Run periodically to update statistics
conn.execute_batch("PRAGMA analysis_limit = 400; PRAGMA optimize;")?;
```

### Prepared Statement Cache

```rust
// Default capacity: 16
conn.set_prepared_statement_cache_capacity(128);

// Use prepare_cached instead of prepare for hot queries
let mut stmt = conn.prepare_cached("SELECT * FROM users WHERE id = ?1")?;
// Statement returned to LRU cache on drop

// Flush all cached statements
conn.flush_prepared_statement_cache();
```

### Batch Inserts

```rust
fn bulk_insert(conn: &mut Connection, items: &[(String, i64)]) -> rusqlite::Result<()> {
    let tx = conn.transaction()?;
    {
        let mut stmt = tx.prepare_cached(
            "INSERT INTO items (name, value) VALUES (?1, ?2)"
        )?;
        for (name, value) in items {
            stmt.execute(params![name, value])?;
        }
    }
    tx.commit()
}
```

Wrapping in a transaction avoids per-statement fsync. 100x+ speedup for large inserts.

### Bulk Loading Tips

- Disable `synchronous` temporarily: `PRAGMA synchronous = OFF` (risk: corruption on crash)
- Drop indices before bulk load, recreate after
- Use `INSERT OR IGNORE` / `INSERT OR REPLACE` to handle conflicts
- Increase `cache_size` temporarily for index creation

---

## 12. Backup

Requires `backup` feature.

### Simple Backup

```rust
use rusqlite::{backup, Connection};
use std::path::Path;
use std::time::Duration;

fn backup_db(src: &Connection, dst_path: &Path) -> rusqlite::Result<()> {
    let mut dst = Connection::open(dst_path)?;
    let backup = backup::Backup::new(src, &mut dst)?;
    backup.run_to_completion(5, Duration::from_millis(250), None)
}
```

### With Progress Callback

```rust
fn backup_with_progress(src: &Connection, dst_path: &Path) -> rusqlite::Result<()> {
    let mut dst = Connection::open(dst_path)?;
    let backup = backup::Backup::new(src, &mut dst)?;
    backup.run_to_completion(
        100,  // pages per step
        Duration::from_millis(100),  // sleep between steps
        Some(|progress: backup::Progress| {
            let pct = (progress.pagecount - progress.remaining) as f64
                / progress.pagecount as f64 * 100.0;
            println!("Backup: {:.1}%", pct);
        }),
    )
}
```

### Incremental Backup

```rust
let backup = backup::Backup::new(src, &mut dst)?;
loop {
    match backup.step(50)? {
        backup::StepResult::Done => break,
        backup::StepResult::More => {
            let progress = backup.progress();
            println!("{} of {} pages remaining", progress.remaining, progress.pagecount);
            std::thread::sleep(Duration::from_millis(100));
        }
        backup::StepResult::Busy => {
            std::thread::sleep(Duration::from_millis(500));
        }
    }
}
```

### Alternative: VACUUM INTO

```rust
conn.execute("VACUUM INTO ?1", ["/backup/app.db.bak"])?;
```

Creates a standalone copy. Simpler but locks the database during the operation.

---

## 13. Hooks

Requires `hooks` feature.

### update_hook -- row change notifications

```rust
conn.update_hook(Some(|action, db_name: &str, table: &str, rowid| {
    println!("{:?} on {}.{} rowid={}", action, db_name, table, rowid);
    // action: Action::SQLITE_INSERT | SQLITE_UPDATE | SQLITE_DELETE
}));

// Remove hook
conn.update_hook(None::<fn(rusqlite::hooks::Action, &str, &str, i64)>);
```

### commit_hook -- pre-commit notification

```rust
conn.commit_hook(Some(|| {
    println!("About to commit");
    false  // return true to convert commit to rollback
}));
```

### rollback_hook

```rust
conn.rollback_hook(Some(|| {
    println!("Transaction rolled back");
}));
```

### authorizer -- statement permission control

```rust
use rusqlite::hooks::{Authorization, AuthAction, AuthContext};

conn.authorizer(Some(|ctx: AuthContext<'_>| {
    match ctx.action {
        AuthAction::Delete { table_name } if table_name == "audit_log" => {
            Authorization::Deny  // prevent deleting audit records
        }
        _ => Authorization::Allow,
    }
}));
```

Return values: `Authorization::Allow`, `Authorization::Deny`, `Authorization::Ignore`.

### progress_handler -- periodic callback during queries

```rust
conn.progress_handler(1000, Some(|| {
    // Called every ~1000 VM instructions
    // Return true to interrupt the operation
    false
}));
```

### preupdate_hook (requires `preupdate_hook` feature)

Notifies before row changes with access to old/new values.

---

## 14. Common Patterns

### Repository Pattern

```rust
use rusqlite::{params, Connection, Result};

pub struct UserRepo {
    conn: Connection,
}

impl UserRepo {
    pub fn new(path: &str) -> Result<Self> {
        let conn = Connection::open(path)?;
        conn.execute_batch("
            PRAGMA journal_mode = WAL;
            PRAGMA foreign_keys = ON;
        ")?;
        Ok(Self { conn })
    }

    pub fn create(&self, name: &str, email: &str) -> Result<i64> {
        self.conn.execute(
            "INSERT INTO users (name, email) VALUES (?1, ?2)",
            params![name, email],
        )?;
        Ok(self.conn.last_insert_rowid())
    }

    pub fn find_by_id(&self, id: i64) -> Result<Option<User>> {
        self.conn
            .query_row(
                "SELECT id, name, email FROM users WHERE id = ?1",
                [id],
                |row| Ok(User {
                    id: row.get(0)?,
                    name: row.get(1)?,
                    email: row.get(2)?,
                }),
            )
            .optional()  // converts QueryReturnedNoRows to Ok(None)
    }

    pub fn delete(&self, id: i64) -> Result<bool> {
        let changes = self.conn.execute("DELETE FROM users WHERE id = ?1", [id])?;
        Ok(changes > 0)
    }
}
```

### Connection Pooling with r2d2

```toml
[dependencies]
r2d2 = "0.8"
r2d2_sqlite = "0.25"
rusqlite = { version = "0.38.0", features = ["bundled"] }
```

```rust
use r2d2::Pool;
use r2d2_sqlite::SqliteConnectionManager;

let manager = SqliteConnectionManager::file("app.db")
    .with_init(|conn| {
        conn.execute_batch("
            PRAGMA journal_mode = WAL;
            PRAGMA foreign_keys = ON;
            PRAGMA busy_timeout = 5000;
        ")
    });

let pool = Pool::builder()
    .max_size(4)  // concurrent readers
    .build(manager)?;

// Get a connection from the pool
let conn = pool.get()?;
conn.execute("SELECT 1", [])?;
```

### Async Wrapper with tokio-rusqlite

```toml
[dependencies]
tokio-rusqlite = "0.6"
tokio = { version = "1", features = ["full"] }
```

```rust
use tokio_rusqlite::Connection;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let conn = Connection::open("app.db").await?;

    conn.call(|conn| {
        conn.execute_batch("PRAGMA journal_mode = WAL")?;
        Ok(())
    }).await?;

    let users = conn.call(|conn| {
        let mut stmt = conn.prepare("SELECT id, name FROM users")?;
        let users = stmt.query_map([], |row| {
            Ok((row.get::<_, i64>(0)?, row.get::<_, String>(1)?))
        })?.collect::<Result<Vec<_>, _>>()?;
        Ok(users)
    }).await?;

    Ok(())
}
```

### Manual spawn_blocking with r2d2

```rust
let pool = pool.clone();
let result = tokio::task::spawn_blocking(move || {
    let conn = pool.get()?;
    conn.query_row("SELECT count(*) FROM users", [], |row| row.get::<_, i64>(0))
}).await??;
```

### Key-Value Store

```rust
pub struct KvStore {
    conn: Connection,
}

impl KvStore {
    pub fn new(path: &str) -> Result<Self> {
        let conn = Connection::open(path)?;
        conn.execute_batch("
            PRAGMA journal_mode = WAL;
            CREATE TABLE IF NOT EXISTS kv (
                key   TEXT PRIMARY KEY,
                value BLOB NOT NULL,
                updated_at INTEGER DEFAULT (unixepoch())
            ) WITHOUT ROWID;
        ")?;
        Ok(Self { conn })
    }

    pub fn get(&self, key: &str) -> Result<Option<Vec<u8>>> {
        self.conn
            .query_row("SELECT value FROM kv WHERE key = ?1", [key], |row| row.get(0))
            .optional()
    }

    pub fn set(&self, key: &str, value: &[u8]) -> Result<()> {
        self.conn.execute(
            "INSERT INTO kv (key, value, updated_at) VALUES (?1, ?2, unixepoch())
             ON CONFLICT(key) DO UPDATE SET value = excluded.value, updated_at = excluded.updated_at",
            params![key, value],
        )?;
        Ok(())
    }

    pub fn delete(&self, key: &str) -> Result<bool> {
        Ok(self.conn.execute("DELETE FROM kv WHERE key = ?1", [key])? > 0)
    }
}
```

### Time-Series Storage

```rust
conn.execute_batch("
    CREATE TABLE IF NOT EXISTS metrics (
        name       TEXT NOT NULL,
        timestamp  INTEGER NOT NULL,
        value      REAL NOT NULL,
        tags       TEXT,  -- JSON
        PRIMARY KEY (name, timestamp)
    ) WITHOUT ROWID;

    CREATE INDEX IF NOT EXISTS idx_metrics_ts ON metrics(timestamp);
")?;

// Insert
conn.execute(
    "INSERT INTO metrics (name, timestamp, value, tags) VALUES (?1, ?2, ?3, ?4)",
    params!["cpu_usage", unixepoch, 72.5, r#"{"host":"srv1"}"#],
)?;

// Query range
let mut stmt = conn.prepare(
    "SELECT timestamp, value FROM metrics
     WHERE name = ?1 AND timestamp BETWEEN ?2 AND ?3
     ORDER BY timestamp"
)?;

// Downsample with GROUP BY
let mut stmt = conn.prepare(
    "SELECT (timestamp / 3600) * 3600 AS bucket, AVG(value)
     FROM metrics
     WHERE name = ?1 AND timestamp BETWEEN ?2 AND ?3
     GROUP BY bucket
     ORDER BY bucket"
)?;
```

### Embeddings / Vector Search with sqlite-vec

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled", "load_extension"] }
sqlite-vec = "0.1.7-alpha.2"
zerocopy = { version = "0.8", features = ["derive"] }
```

```rust
use rusqlite::{Connection, ffi::sqlite3_auto_extension, params};
use sqlite_vec::sqlite3_vec_init;

fn main() -> rusqlite::Result<()> {
    // Register sqlite-vec as auto extension
    unsafe {
        sqlite3_auto_extension(Some(std::mem::transmute(
            sqlite3_vec_init as *const (),
        )));
    }

    let conn = Connection::open_in_memory()?;

    // Create vector table (384-dimensional float32 vectors)
    conn.execute_batch("
        CREATE VIRTUAL TABLE embeddings USING vec0(
            id INTEGER PRIMARY KEY,
            vector FLOAT[384]
        );
    ")?;

    // Insert embedding
    let embedding: Vec<f32> = vec![0.1; 384];
    conn.execute(
        "INSERT INTO embeddings (id, vector) VALUES (?1, ?2)",
        params![1, embedding.as_slice().as_bytes()],
    )?;

    // KNN search (find 10 nearest neighbors)
    let query: Vec<f32> = vec![0.1; 384];
    let mut stmt = conn.prepare(
        "SELECT id, distance
         FROM embeddings
         WHERE vector MATCH ?1
         ORDER BY distance
         LIMIT 10"
    )?;

    let results = stmt.query_map(
        [query.as_slice().as_bytes()],
        |row| Ok((row.get::<_, i64>(0)?, row.get::<_, f64>(1)?)),
    )?;

    for r in results {
        let (id, dist) = r?;
        println!("id={}, distance={:.4}", id, dist);
    }

    Ok(())
}
```

Use `zerocopy::AsBytes` for zero-copy `Vec<f32>` to bytes conversion.

---

## 15. Error Handling

### Error Enum (28 variants, non-exhaustive)

Key variants:

| Variant | When |
|---------|------|
| `SqliteFailure(ffi::Error, Option<String>)` | Underlying SQLite error |
| `QueryReturnedNoRows` | `query_row` found no match |
| `QueryReturnedMoreThanOneRow` | `query_one` found multiple |
| `InvalidParameterCount(usize, usize)` | Wrong number of params |
| `InvalidColumnIndex(usize)` | Column index out of range |
| `InvalidColumnName(String)` | Named column not found |
| `InvalidColumnType(usize, String, Type)` | Type mismatch on get() |
| `FromSqlConversionFailure(usize, Type, Box<dyn Error>)` | FromSql failed |
| `ToSqlConversionFailure(Box<dyn Error>)` | ToSql failed |
| `IntegralValueOutOfRange(usize, i64)` | Integer overflow |
| `Utf8Error(Utf8Error)` | Invalid UTF-8 |
| `ExecuteReturnedResults` | execute() returned rows |
| `SqlInputError { msg, sql, offset }` | SQL syntax error with position |
| `SqliteSingleThreadedMode` | Threading mode conflict |

### Matching Constraint Violations

```rust
use rusqlite::{Error, ErrorCode, ffi};

match conn.execute("INSERT INTO users (email) VALUES (?1)", ["dupe@test.com"]) {
    Ok(_) => println!("Inserted"),
    Err(Error::SqliteFailure(err, msg)) => {
        match err.extended_code {
            ffi::SQLITE_CONSTRAINT_UNIQUE => {
                println!("Duplicate: {:?}", msg);
            }
            ffi::SQLITE_CONSTRAINT_FOREIGNKEY => {
                println!("FK violation: {:?}", msg);
            }
            ffi::SQLITE_CONSTRAINT_NOTNULL => {
                println!("NULL violation: {:?}", msg);
            }
            ffi::SQLITE_CONSTRAINT_CHECK => {
                println!("CHECK failed: {:?}", msg);
            }
            ffi::SQLITE_CONSTRAINT_PRIMARYKEY => {
                println!("PK violation: {:?}", msg);
            }
            _ => println!("SQLite error {}: {:?}", err.extended_code, msg),
        }
    }
    Err(e) => println!("Other error: {}", e),
}
```

### Extended Error Code Constants

| Constant | Value | Meaning |
|----------|-------|---------|
| `SQLITE_CONSTRAINT_UNIQUE` | 2067 | UNIQUE violation |
| `SQLITE_CONSTRAINT_PRIMARYKEY` | 1555 | PRIMARY KEY violation |
| `SQLITE_CONSTRAINT_FOREIGNKEY` | 787 | FOREIGN KEY violation |
| `SQLITE_CONSTRAINT_NOTNULL` | 1299 | NOT NULL violation |
| `SQLITE_CONSTRAINT_CHECK` | 275 | CHECK constraint failed |
| `SQLITE_CONSTRAINT_TRIGGER` | 1811 | RAISE in trigger |

### ErrorCode Enum

```rust
match err.code {
    ErrorCode::ConstraintViolation => { /* any constraint */ }
    ErrorCode::DatabaseBusy => { /* SQLITE_BUSY */ }
    ErrorCode::DatabaseLocked => { /* SQLITE_LOCKED */ }
    ErrorCode::CannotOpen => { /* file not found etc */ }
    ErrorCode::ReadOnly => { /* read-only database */ }
    ErrorCode::DiskFull => { /* SQLITE_FULL */ }
    _ => {}
}
```

### Helper Methods

```rust
let err: Error = /* ... */;
err.sqlite_error()       // Option<ffi::Error> -- underlying SQLite error
err.sqlite_error_code()  // Option<ffi::ErrorCode> -- error code enum
```

### Converting QueryReturnedNoRows to Option

```rust
use rusqlite::OptionalExtension;

// Returns Ok(None) instead of Err(QueryReturnedNoRows)
let user: Option<String> = conn
    .query_row("SELECT name FROM users WHERE id = ?1", [99], |row| row.get(0))
    .optional()?;
```
