# FTS5 Advanced Reference (rusqlite)

Advanced FTS5 patterns for rusqlite. Basic usage (CREATE, MATCH, bm25, highlight,
snippet, external content, sync triggers) is in `rusqlite.md` section 9.

---

## 1. Tokenizers

| Tokenizer | Behavior | Best For |
|-----------|----------|----------|
| `unicode61` | Unicode word splitting, case-folding, diacritic removal | General text (default) |
| `ascii` | ASCII-only word splitting, case-folding | English-only, faster |
| `porter` | Porter stemming (running->run, connected->connect) | Natural language search |
| `trigram` | 3-char sliding window, substring matching | LIKE '%term%' replacement, code, paths |

### Tokenizer Chaining

Order matters: `"porter unicode61"` stems after unicode61 tokenization.

```rust
// Prose: porter stemming catches word variants
conn.execute_batch(
    "CREATE VIRTUAL TABLE notes_fts USING fts5(title, body, tokenize='porter unicode61');"
)?;

// Code/paths: trigram enables substring matching without LIKE
conn.execute_batch(
    "CREATE VIRTUAL TABLE code_fts USING fts5(filepath, content, tokenize='trigram');"
)?;
// "parse_json" matches "fn parse_json_value()" via substring
let mut stmt = conn.prepare("SELECT filepath FROM code_fts WHERE code_fts MATCH ?1")?;
```

### nmem Recommendation

- **Prose fields** (summaries, decisions): `porter unicode61`
- **Code/path fields** (file paths, snippets): `trigram`
- Consider two FTS tables if content types differ significantly

---

## 2. External Content Tables

### Contentless (`content=''`)

Stores only the inverted index. No original text retained.

```rust
conn.execute_batch("CREATE VIRTUAL TABLE idx USING fts5(title, body, content='');")?;

// Must supply rowid explicitly
conn.execute(
    "INSERT INTO idx(rowid, title, body) VALUES (?1, ?2, ?3)",
    params![42, "Title", "Body text"],
)?;
```

**Trade-offs:** saves ~50-100% disk, but cannot use `highlight()`/`snippet()` (no
original text). Must join back to source table for display. Deletes require supplying
the original column values.

### External Content (`content='table_name'`)

Index points to a regular table. FTS reads content on demand.

```rust
conn.execute_batch("
    CREATE TABLE observations (
        id INTEGER PRIMARY KEY, title TEXT NOT NULL, body TEXT NOT NULL,
        metadata TEXT  -- JSON, not indexed
    );
    CREATE VIRTUAL TABLE obs_fts USING fts5(
        title, body, content='observations', content_rowid='id'
    );
")?;
```

Sync triggers (INSERT/DELETE/UPDATE) are required -- see `rusqlite.md` section 9 for
the full trigger pattern. Rebuild after bulk loads: `INSERT INTO obs_fts(obs_fts) VALUES('rebuild');`

---

## 3. Column Configuration

### Column Weights in bm25()

Higher weight = more important. Positional, matching column order.

```rust
// title 10x more important than body
let mut stmt = conn.prepare(
    "SELECT rowid, bm25(obs_fts, 10.0, 1.0) AS score
     FROM obs_fts WHERE obs_fts MATCH ?1 ORDER BY score"  // ASC: lower = better
)?;
```

### Unindexed Columns

Stored but not searchable. Avoids JOIN for metadata retrieval.

```rust
conn.execute_batch(
    "CREATE VIRTUAL TABLE notes_fts USING fts5(
        title, body, category UNINDEXED, created_at UNINDEXED
    );"
)?;
// category is returned in results but MATCH won't search it
```

### Prefix Indexes

Speed up prefix queries (`term*`) at cost of larger index.

```rust
conn.execute_batch("CREATE VIRTUAL TABLE notes_fts USING fts5(title, body, prefix='2,3');")?;
// `ru*` and `rus*` are now fast; without this, prefix queries scan the token list
```

### Detail Level

Controls positional data stored. Smaller index, fewer features.

| Level | Size | Column Filters | NEAR/Phrase | highlight/snippet |
|-------|------|----------------|-------------|-------------------|
| `full` (default) | Largest | Yes | Yes | Yes |
| `column` | Medium | Yes | No | No |
| `none` | Smallest | No | No | No |

```rust
conn.execute_batch("CREATE VIRTUAL TABLE logs_fts USING fts5(message, detail='none');")?;
```

---

## 4. Query Syntax (Advanced)

```rust
// Column filter: search only title
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH 'title:refactor'"

// Column filter with phrase
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH 'title:\"error handling\"'"

// NEAR: terms within N tokens (requires detail=full)
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH 'NEAR(rust async, 5)'"

// Initial token: match at start of column
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH '^bug'"

// Boolean + grouping
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH '(rust OR golang) AND NOT python'"

// Phrase + boolean
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH '\"error handling\" AND async'"

// Prefix
"SELECT rowid FROM obs_fts WHERE obs_fts MATCH 'tokio*'"
```

---

## 5. Auxiliary Functions

### Custom Default Ranking

Set once, applies to all `ORDER BY rank` queries.

```rust
conn.execute_batch(
    "INSERT INTO obs_fts(obs_fts, rank) VALUES('rank', 'bm25(10.0, 1.0)');"
)?;

// Now ORDER BY rank uses configured bm25 weights automatically
let mut stmt = conn.prepare(
    "SELECT rowid, rank FROM obs_fts WHERE obs_fts MATCH ?1 ORDER BY rank"
)?;

// Equivalent to explicit: bm25(obs_fts, 10.0, 1.0) AS score ... ORDER BY score
```

---

## 6. Maintenance

All maintenance uses the `INSERT INTO fts(fts) VALUES(...)` command pattern:

| Command | Purpose | When |
|---------|---------|------|
| `'optimize'` | Merge index segments into one | Daily or after batch inserts |
| `'rebuild'` | Rebuild index from content table | After bulk loads, index drift |
| `'integrity-check'` | Verify index consistency | Startup |

```rust
fn run_fts_maintenance(conn: &Connection) -> rusqlite::Result<()> {
    conn.execute_batch("INSERT INTO obs_fts(obs_fts) VALUES('integrity-check');")?;
    conn.execute_batch("INSERT INTO obs_fts(obs_fts) VALUES('optimize');")?;
    Ok(())
}

// Automerge: higher = fewer background merges (default 4, range 0-16)
conn.execute_batch("INSERT INTO obs_fts(obs_fts, rank) VALUES('automerge', 8);")?;
```

---

## 7. FTS5 + JSON Pattern

Index searchable text extracted from JSON columns via triggers.

```rust
conn.execute_batch("
    CREATE TABLE events (
        id INTEGER PRIMARY KEY,
        data TEXT NOT NULL  -- JSON: {title, tags[], body}
    );
    CREATE VIRTUAL TABLE events_fts USING fts5(
        title, tags, body, content='events', content_rowid='id'
    );
")?;
```

INSERT trigger extracts JSON fields. Arrays use `json_each` + `group_concat`:

```rust
conn.execute_batch("
    CREATE TRIGGER events_fts_ai AFTER INSERT ON events BEGIN
        INSERT INTO events_fts(rowid, title, tags, body) VALUES (
            new.id,
            json_extract(new.data, '$.title'),
            COALESCE((SELECT group_concat(je.value, ' ')
                FROM json_each(json_extract(new.data, '$.tags')) AS je), ''),
            json_extract(new.data, '$.body')
        );
    END;
")?;
// DELETE and UPDATE triggers follow the same pattern -- use json_extract(old.data, ...)
// for the delete command values. See rusqlite.md section 9 for the trigger structure.
```

```rust
conn.execute(
    "INSERT INTO events (data) VALUES (?1)",
    [r#"{"title":"Bug fix","tags":["rust","async"],"body":"Fixed lifetime issue"}"#],
)?;

let mut stmt = conn.prepare(
    "SELECT e.data FROM events e
     JOIN events_fts f ON e.id = f.rowid
     WHERE events_fts MATCH ?1 ORDER BY f.rank"
)?;
```

---

## 8. Performance Characteristics

| Operation | Cost | Notes |
|-----------|------|-------|
| MATCH query | ~constant | Independent of table size |
| INSERT (with triggers) | ~2-3x plain INSERT | Trigger overhead |
| Index size | 50-100% of text | Tokenizer and detail level dependent |
| `optimize` | O(n) | Linear in index size, blocks writes briefly |
| `rebuild` | O(n) | Linear in content table size, blocks writes |
| Prefix query (`term*`) | Fast with `prefix=` | Without it, scans token list |
| `trigram` index size | ~3-5x `unicode61` | Every 3-char sequence indexed |

Sizing: 100K observations at 200 bytes avg (20 MB content) produces ~15-20 MB index
(`unicode61/full`), ~5-8 MB (`unicode61/none`), or ~60-100 MB (`trigram`).

---

## 9. Sanitizing User Input for MATCH

Special chars that break MATCH: `*`, `"`, `(`, `)`, `+`, `-`, `^`, `NEAR`, `:`.

Strategy: strip `"`, wrap each term in double quotes, join with spaces (implicit AND).

```rust
fn sanitize_fts_query(input: &str) -> Option<String> {
    let terms: Vec<String> = input
        .split_whitespace()
        .map(|w| format!("\"{}\"", w.replace('"', "")))
        .collect();
    if terms.is_empty() { None } else { Some(terms.join(" ")) }
}

fn search(conn: &Connection, user_input: &str) -> rusqlite::Result<Vec<(i64, f64)>> {
    let query = match sanitize_fts_query(user_input) {
        Some(q) => q,
        None => return Ok(vec![]),
    };
    let mut stmt = conn.prepare_cached(
        "SELECT rowid, bm25(obs_fts, 10.0, 1.0) AS score
         FROM obs_fts WHERE obs_fts MATCH ?1 ORDER BY score LIMIT 20"
    )?;
    stmt.query_map([&query], |row| Ok((row.get(0)?, row.get(1)?)))?
        .collect::<rusqlite::Result<Vec<_>>>()
}
```

---

## 10. Pitfalls

**Delete command syntax is an INSERT.** Easy to get wrong:

```rust
// CORRECT: first column is the table name
conn.execute(
    "INSERT INTO obs_fts(obs_fts, rowid, title, body) VALUES('delete', ?1, ?2, ?3)",
    params![old_id, old_title, old_body],
)?;
// WRONG: inserts a row with literal text "delete"
conn.execute(
    "INSERT INTO obs_fts(rowid, title, body) VALUES(?1, ?2, ?3)",
    params![old_id, old_title, old_body],
)?;
```

**All three triggers are mandatory.** Missing DELETE = phantom results. Missing UPDATE =
stale results after edits.

**Contentless tables cannot use highlight() or snippet().**

```rust
// Errors with content='': "no such column: obs_fts.title"
conn.prepare("SELECT highlight(obs_fts, 0, '<b>', '</b>') FROM obs_fts WHERE obs_fts MATCH ?1");
```

**MATCH with empty string errors.** Always validate before querying.

```rust
// WRONG: "fts5: syntax error"
conn.execute("SELECT rowid FROM obs_fts WHERE obs_fts MATCH ''", [])?;
// CORRECT: guard
if !query.trim().is_empty() { /* run MATCH */ }
```

**Column filter: single terms vs phrases.**

```rust
"title:error handling"           // WRONG: only "error" filtered to title
"title:\"error handling\""       // CORRECT: phrase in column filter
```

**Stale index after bypassing triggers.** Bulk loads or `.import` cause drift.
Fix: `INSERT INTO obs_fts(obs_fts) VALUES('rebuild');`

**Trigram does not support boolean/phrase queries.** `AND`, `OR`, `NOT`, `"phrase"`
syntax errors with trigram. It only does substring matching.