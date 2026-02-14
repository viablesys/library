# rusqlite_migration Reference

Schema migration for rusqlite via SQLite's `user_version` PRAGMA. Version: **2.3.0**.

Docs: <https://docs.rs/rusqlite_migration/latest/rusqlite_migration/>

> See `rusqlite.md` section 8 for the basics. This doc goes deeper.

---

## 1. Installation

```toml
[dependencies]
rusqlite_migration = "2.3"
rusqlite = { version = "0.38", features = ["bundled"] }
```

Optional features: `from-directory` (load `*.sql` files via `include_dir!`), `alpha-async-tokio-rusqlite` (deprecated -- see section 7).

---

## 2. How It Works

- Version stored in `user_version` PRAGMA -- 4-byte integer at fixed file offset, no extra tables
- Migrations run sequentially; each in its own transaction (auto-rollback on failure)
- Version number = count of applied migrations (0 = fresh database)
- **Warning:** if anything else writes `user_version`, behavior is undefined

---

## 3. Defining Migrations

```rust
use rusqlite_migration::{Migrations, M};

const MIGRATIONS: Migrations<'static> = Migrations::from_slice(&[
    M::up("CREATE TABLE observations (
        id INTEGER PRIMARY KEY, session TEXT NOT NULL,
        content TEXT NOT NULL, created INTEGER NOT NULL DEFAULT (unixepoch())
    );"),
    M::up("ALTER TABLE observations ADD COLUMN obs_type TEXT DEFAULT 'discovery';"),
    M::up("CREATE TABLE sessions (id TEXT PRIMARY KEY, started INTEGER NOT NULL, summary TEXT);"),
]);
```

**From files:** `M::up(include_str!("../migrations/001_observations.sql"))`

**With rollback:** `M::up("CREATE TABLE sessions (...);").down("DROP TABLE sessions;")`

**With comment:** `M::up("CREATE INDEX ...;").comment("speed up session lookups")`

**Foreign key check:** `M::up("ALTER TABLE ... REFERENCES ...;").foreign_key_check()` -- runs `PRAGMA foreign_key_check` before commit.

**Rust hook** (data transformation during migration):
```rust
M::up_with_hook(
    "ALTER TABLE observations ADD COLUMN embedding BLOB;",
    |tx: &rusqlite::Transaction| {
        let mut stmt = tx.prepare("SELECT id, content FROM observations WHERE embedding IS NULL")?;
        let rows: Vec<(i64, String)> = stmt.query_map([], |row| {
            Ok((row.get(0)?, row.get(1)?))
        })?.collect::<Result<_, _>>()?;
        let mut update = tx.prepare("UPDATE observations SET embedding = ?1 WHERE id = ?2")?;
        for (id, content) in &rows {
            let emb = compute_embedding(content);
            update.execute(rusqlite::params![emb.as_slice(), id])?;
        }
        Ok(())
    },
),
```

**Rules:** Append-only (never modify/reorder). SQL must end with `;`. Avoid PRAGMAs in SQL (use hooks). `down` must exactly reverse `up`.

---

## 4. Running Migrations

```rust
fn open_db(path: &str) -> rusqlite::Result<Connection> {
    let mut conn = Connection::open(path)?;
    conn.pragma_update(None, "journal_mode", "WAL")?;
    MIGRATIONS.to_latest(&mut conn).expect("migration failed");
    Ok(conn)
}
```

| Method | Purpose |
|--------|---------|
| `to_latest(&mut conn)` | Apply all pending migrations |
| `to_version(&mut conn, 2)` | Migrate to version 2 (up or down) |
| `to_version(&mut conn, 0)` | Roll back everything |
| `current_version(&conn)` | Returns `SchemaVersion` enum |
| `pending_migrations(&conn)` | Count of unapplied (negative = db ahead of code) |

### SchemaVersion

```rust
use rusqlite_migration::SchemaVersion;
match MIGRATIONS.current_version(&conn)? {
    SchemaVersion::NoneSet => println!("fresh database"),
    SchemaVersion::Inside(v) => println!("version {v}"),
    SchemaVersion::Outside(v) => println!("version {v} -- ahead of code!"),
}
```

---

## 5. Validation

```rust
#[test]
fn migrations_valid() { assert!(MIGRATIONS.validate().is_ok()); }
```

`validate()` runs all up-migrations on a temp in-memory DB. Catches SQL errors, conflicting changes, hook panics. Does **not** validate down migrations.

---

## 6. From Directory (`from-directory` feature)

```toml
rusqlite_migration = { version = "2.3", features = ["from-directory"] }
include_dir = "0.7"
```

Directory pattern: `{id}-{name}/` with `up.sql` (required) and `down.sql` (optional).

```
migrations/
  01-create_observations/  up.sql  down.sql
  02-add_obs_type/         up.sql
  03-create_sessions/      up.sql  down.sql
```

```rust
use rusqlite_migration::Migrations;
use include_dir::{Dir, include_dir};
static MIGRATION_DIR: Dir = include_dir!("$CARGO_MANIFEST_DIR/migrations");
let migrations = Migrations::from_directory(&MIGRATION_DIR).expect("load failed");
```

Files embedded at compile time -- no filesystem access at runtime.

**MigrationsBuilder** -- add hooks to file-based migrations:
```rust
use rusqlite_migration::MigrationsBuilder;
let migrations = MigrationsBuilder::from_directory(&MIGRATION_DIR).unwrap()
    .edit(3, |m| m.set_up_hook(|tx| {
        tx.execute("INSERT INTO defaults VALUES ('nmem', '1.0')", [])?;
        Ok(())
    }))
    .finalize();
```

| | Inline (`from_slice`) | Directory |
|-|----------------------|-----------|
| `const`-compatible | Yes | No |
| Hooks | Direct | Via `MigrationsBuilder::edit` |
| SQL editor support | No | Yes |
| Extra deps | None | `include_dir` |

---

## 7. Async Support

Deprecated feature. Define migrations normally, run inside `tokio-rusqlite`:

```rust
let conn = tokio_rusqlite::Connection::open("nmem.db").await?;
conn.call(|conn| { MIGRATIONS.to_latest(conn)?; Ok(()) }).await?;
```

---

## 8. Patterns for Schema Evolution

**Add column** -- always provide DEFAULT for existing rows:
```rust
M::up("ALTER TABLE observations ADD COLUMN obs_type TEXT DEFAULT 'discovery';"),
```

**Add table / index:**
```rust
M::up("CREATE TABLE IF NOT EXISTS sessions (id TEXT PRIMARY KEY, started INTEGER NOT NULL);"),
M::up("CREATE INDEX IF NOT EXISTS idx_obs_created ON observations(created);"),
```

**Rename column** (SQLite 3.25+, bundled 0.38 ships 3.51):
```rust
M::up("ALTER TABLE observations RENAME COLUMN content TO body;"),
```

**FTS5 index** -- split into schema + rebuild migrations:
```rust
M::up("
    CREATE VIRTUAL TABLE obs_fts USING fts5(body, content='observations', content_rowid='id');
    CREATE TRIGGER obs_fts_ai AFTER INSERT ON observations BEGIN
        INSERT INTO obs_fts(rowid, body) VALUES (new.id, new.body);
    END;
    CREATE TRIGGER obs_fts_ad AFTER DELETE ON observations BEGIN
        INSERT INTO obs_fts(obs_fts, rowid, body) VALUES('delete', old.id, old.body);
    END;
    CREATE TRIGGER obs_fts_au AFTER UPDATE ON observations BEGIN
        INSERT INTO obs_fts(obs_fts, rowid, body) VALUES('delete', old.id, old.body);
        INSERT INTO obs_fts(rowid, body) VALUES (new.id, new.body);
    END;
"),
M::up("INSERT INTO obs_fts(obs_fts) VALUES('rebuild');"),
```

**Major schema change** (recreate table pattern):
```rust
M::up_with_hook(
    "CREATE TABLE observations_new (
        id INTEGER PRIMARY KEY, session TEXT NOT NULL, body TEXT NOT NULL,
        obs_type TEXT NOT NULL DEFAULT 'discovery', created INTEGER NOT NULL DEFAULT (unixepoch())
    );",
    |tx: &rusqlite::Transaction| {
        tx.execute_batch("
            INSERT INTO observations_new SELECT id, session, content, COALESCE(obs_type,'discovery'), created FROM observations;
            DROP TABLE observations;
            ALTER TABLE observations_new RENAME TO observations;
        ")?;
        Ok(())
    },
),
```

**Foreign keys during migration:**
```rust
M::up_with_hook("ALTER TABLE observations ADD COLUMN session_id TEXT;", |tx| {
    tx.pragma_update(None, "foreign_keys", "OFF")?;
    tx.execute_batch("UPDATE observations SET session_id = session;")?;
    tx.pragma_update(None, "foreign_keys", "ON")?;
    Ok(())
}),
```

---

## 9. Testing Migrations

```rust
#[test]
fn migrations_apply() {
    let mut conn = rusqlite::Connection::open_in_memory().unwrap();
    MIGRATIONS.to_latest(&mut conn).unwrap();
}

#[test]
fn migration_v2_adds_obs_type() {
    let mut conn = rusqlite::Connection::open_in_memory().unwrap();
    MIGRATIONS.to_version(&mut conn, 1).unwrap();
    conn.execute("INSERT INTO observations (session, content) VALUES ('s1', 'test')", []).unwrap();
    MIGRATIONS.to_version(&mut conn, 2).unwrap();
    let t: String = conn.query_row(
        "SELECT obs_type FROM observations WHERE session = 's1'", [], |r| r.get(0),
    ).unwrap();
    assert_eq!(t, "discovery");
}

#[test]
fn migration_v3_rollback() {
    let mut conn = rusqlite::Connection::open_in_memory().unwrap();
    MIGRATIONS.to_version(&mut conn, 3).unwrap();
    MIGRATIONS.to_version(&mut conn, 2).unwrap();
    let exists: bool = conn.query_row(
        "SELECT count(*)>0 FROM sqlite_master WHERE type='table' AND name='sessions'",
        [], |r| r.get(0),
    ).unwrap();
    assert!(!exists);
}
```

---

## 10. Pitfalls

- **Version = slice position.** Ordered by position in `from_slice`, not filename. `user_version` increments per applied migration.
- **ALTER TABLE limits.** No `DROP COLUMN` before SQLite 3.35. Never change types or add constraints to existing columns. Workaround: create new table, copy, drop old, rename (section 8).
- **`execute_batch` errors.** Multiple statements in one migration -- if one fails, you get one error with no indication which statement failed. Keep migrations focused.
- **Foreign keys.** `PRAGMA foreign_keys` is per-connection, defaults OFF. Enable before migrations and ALTER TABLE may fail. Use `.foreign_key_check()` or hooks.
- **Hooks break `const`.** `up_with_hook`/`down_with_hook` require `Migrations::new(vec![...])` instead of `from_slice`.
- **No partial state.** Failed migration rolls back entirely -- cannot resume mid-batch.
- **`user_version` conflicts.** Only one migration system per database file.

---

## 11. Version Inspection

```rust
let version: usize = MIGRATIONS.current_version(&conn)?.into();
```

From CLI: `sqlite3 nmem.db "PRAGMA user_version;"`

Uses: backup verification, debugging schema drift, manual recovery (`PRAGMA user_version = N;` -- dangerous, recovery only).
