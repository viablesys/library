# SQLite Retrieval Patterns for Structured Memory

Multi-signal retrieval using FTS5 + SQL for small structured datasets (<20K rows).
Complements `fts5.md` (mechanics) with retrieval **strategy** (combining signals).

All examples use the nmem observations table:

```sql
CREATE TABLE observations (
    id INTEGER PRIMARY KEY, timestamp TEXT NOT NULL, session_id TEXT NOT NULL,
    project TEXT NOT NULL, obs_type TEXT NOT NULL, content TEXT NOT NULL,
    file_path TEXT
);
CREATE VIRTUAL TABLE obs_fts USING fts5(
    content, content='observations', content_rowid='id', tokenize='porter unicode61'
);
-- Sync triggers: see fts5.md section 2
```

---

## 1. The Retrieval Problem

nmem answers: "given what I'm doing now, what past observations are relevant?"

Relevance signals: **text match** (BM25), **recency** (exponential decay), **project scope**
(same project scores higher), **type priority** (errors > decisions > reads), **file proximity**
(same file/directory). No single signal suffices. The strategy is composing them.

---

## 2. FTS5 + WHERE Composition

MATCH narrows via inverted index first, then WHERE filters the match set. No full scan.

```rust
// Text match filtered by project and type
let mut stmt = conn.prepare_cached(
    "SELECT o.id, o.content, bm25(obs_fts) AS score
     FROM obs_fts JOIN observations o ON o.id = obs_fts.rowid
     WHERE obs_fts MATCH ?1 AND o.project = ?2
       AND o.obs_type IN ('error', 'decision', 'file_write')
     ORDER BY score LIMIT ?3"
)?;
stmt.query_map(params![query, project, limit], |row| {
    Ok((row.get::<_, i64>(0)?, row.get::<_, String>(1)?, row.get::<_, f64>(2)?))
})?;

// With recency window (?2 = '-7 days', '-30 days')
"... WHERE obs_fts MATCH ?1 AND o.timestamp >= datetime('now', ?2) ORDER BY score LIMIT ?3"

// Exclude low-signal types
"... WHERE obs_fts MATCH ?1 AND o.obs_type != 'file_read' ORDER BY score LIMIT ?2"
```

---

## 3. Recency Weighting

Exponential decay via `julianday()` difference. Half-life of 7 days: score = 0.5 at 7d,
0.25 at 14d, 0.125 at 21d.

```sql
-- pow() available in bundled SQLite 3.46+
SELECT id, pow(0.5, (julianday('now') - julianday(timestamp)) / 7.0) AS recency
FROM observations;
```

Alternative: register custom function for older SQLite:

```rust
conn.create_scalar_function(
    "exp_decay", 2, FunctionFlags::SQLITE_DETERMINISTIC,
    |ctx| {
        let age: f64 = ctx.get(0)?;
        let half_life: f64 = ctx.get(1)?;
        Ok((-0.693 * age / half_life).exp())
    },
)?;
```

### BM25 * recency

`bm25()` returns negative values (lower = better). Multiplying by recency (0..1) preserves
ordering: good match (-5.0) * high recency (0.9) = -4.5, stays more negative than poor
match (-1.0 * 0.9 = -0.9).

```sql
SELECT o.id, o.content,
    bm25(obs_fts) * pow(0.5, (julianday('now') - julianday(o.timestamp)) / 7.0) AS score
FROM obs_fts JOIN observations o ON o.id = obs_fts.rowid
WHERE obs_fts MATCH ?1
ORDER BY score LIMIT 20;  -- ASC (more negative = better)
```

---

## 4. Composite Scoring

Four signals normalized and weighted:

| Signal | Raw | Normalized |
|--------|-----|------------|
| BM25 | negative floats | `-bm25()` (positive, higher = better) |
| Recency | 0..1 decay | already normalized |
| Type | discrete | error=3, decision=2, file_write=1, command=0.8, file_read=0.5 |
| Project | boolean | same=1.0, different=0.3 |

```rust
fn search(conn: &Connection, query: &str, project: &str, limit: i64)
    -> rusqlite::Result<Vec<ScoredObs>>
{
    let q = sanitize_fts_query(query)?;
    let mut stmt = conn.prepare_cached("
        WITH scored AS (
            SELECT o.id, o.content, o.timestamp, o.obs_type, o.file_path,
                -bm25(obs_fts) AS text_score,
                pow(0.5, (julianday('now') - julianday(o.timestamp)) / 7.0) AS recency,
                CASE o.obs_type
                    WHEN 'error' THEN 3.0 WHEN 'decision' THEN 2.0
                    WHEN 'file_write' THEN 1.0 WHEN 'command' THEN 0.8 ELSE 0.5
                END AS type_w,
                CASE WHEN o.project = ?2 THEN 1.0 ELSE 0.3 END AS proj
            FROM obs_fts JOIN observations o ON o.id = obs_fts.rowid
            WHERE obs_fts MATCH ?1
        )
        SELECT id, content, timestamp, obs_type, file_path,
            (text_score * 0.4 + recency * 0.3 + type_w * 0.2 + proj * 0.1) AS score
        FROM scored ORDER BY score DESC LIMIT ?3
    ")?;
    stmt.query_map(params![q, project, limit], |row| Ok(ScoredObs {
        id: row.get(0)?, content: row.get(1)?, timestamp: row.get(2)?,
        obs_type: row.get(3)?, file_path: row.get(4)?, score: row.get(5)?,
    }))?.collect()
}
```

Weights (0.4/0.3/0.2/0.1) are starting points. Tune by logging which retrievals led to
`get_observations` follow-ups (implicit usefulness signal).

---

## 5. Recency-Only Retrieval

For session start context injection -- no search query, just "recent + relevant."

```rust
// Last N observations for a project
"SELECT id, timestamp, obs_type, content, file_path
 FROM observations WHERE project = ?1 ORDER BY timestamp DESC LIMIT ?2"

// Last N per type (most recent error, decision, write, etc.)
"WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY obs_type ORDER BY timestamp DESC) AS rn
    FROM observations WHERE project = ?1
)
SELECT id, timestamp, obs_type, content, file_path
FROM ranked WHERE rn <= ?2 ORDER BY timestamp DESC"

// Last N per project (cross-project context)
"WITH ranked AS (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY project ORDER BY timestamp DESC) AS rn
    FROM observations
)
SELECT id, timestamp, project, obs_type, content FROM ranked WHERE rn <= ?1
ORDER BY timestamp DESC"

// Last session's observations
"SELECT id, timestamp, obs_type, content, file_path FROM observations
 WHERE session_id = (
    SELECT session_id FROM observations WHERE project = ?1
    ORDER BY timestamp DESC LIMIT 1
 ) ORDER BY timestamp ASC"
```

---

## 6. File-Based Retrieval

No text search needed -- pure SQL on `file_path`.

```rust
// Exact path
"SELECT id, timestamp, obs_type, content FROM observations
 WHERE file_path = ?1 ORDER BY timestamp DESC LIMIT ?2"

// Directory prefix (uses index, no leading %)
"SELECT id, timestamp, obs_type, file_path, content FROM observations
 WHERE file_path LIKE ?1 ORDER BY timestamp DESC LIMIT ?2"
// ?1 = '/home/bpd/dev/viablesys/nmem/src/%'

// Basename match (any directory -- leading % prevents index, fine at <20K)
// ?1 = '%/schema.rs'

// Cross-session file history
"SELECT session_id, obs_type, MIN(timestamp) AS first_seen,
    MAX(timestamp) AS last_seen, COUNT(*) AS touch_count
 FROM observations WHERE file_path = ?1
 GROUP BY session_id, obs_type ORDER BY last_seen DESC"
```

---

## 7. BM25 at Small Scale

BM25 ranks by: term frequency (with saturation), inverse document frequency (rare terms
score higher), document length normalization.

**<100 rows:** BM25 is nearly useless. IDF has no discrimination -- 5 matches in 80 docs
produces mild penalties. Use recency-only or exact match.

**100-1K rows:** BM25 begins differentiating. Common terms get downweighted, rare terms
(specific error messages) surface correctly. FTS5 starts earning its keep.

**1K-20K rows:** BM25 works well. Sweet spot for FTS5 without vector search.

| Situation | Better approach |
|-----------|----------------|
| <100 observations | Recency-only or `LIKE '%term%'` |
| No text query | Recency-only (section 5) |
| File-specific | Path match (section 6) |
| Substring in paths/code | Trigram FTS or LIKE |

Strategy switch for bootstrapping:

```rust
fn choose_strategy(conn: &Connection, project: &str) -> rusqlite::Result<Strategy> {
    let count: i64 = conn.query_row(
        "SELECT COUNT(*) FROM observations WHERE project = ?1",
        [project], |row| row.get(0),
    )?;
    Ok(if count < 100 { Strategy::RecencyOnly } else { Strategy::Composite })
}
```

---

## 8. Rust Patterns

### Result struct

```rust
struct ScoredObs {
    id: i64, content: String, timestamp: String,
    obs_type: String, file_path: Option<String>, score: f64,
}
```

### Input sanitization (see also fts5.md section 9)

```rust
fn sanitize_fts_query(input: &str) -> rusqlite::Result<String> {
    let terms: Vec<String> = input.split_whitespace()
        .filter(|w| !w.is_empty())
        .map(|w| format!("\"{}\"", w.replace('"', "")))
        .collect();
    if terms.is_empty() {
        return Err(rusqlite::Error::QueryReturnedNoRows);
    }
    Ok(terms.join(" "))
}
```

### Recency-only context injection

```rust
fn recent_context(conn: &Connection, project: &str, per_type: i64)
    -> rusqlite::Result<Vec<ScoredObs>>
{
    let mut stmt = conn.prepare_cached(
        "WITH ranked AS (
            SELECT *, ROW_NUMBER() OVER (
                PARTITION BY obs_type ORDER BY timestamp DESC
            ) AS rn FROM observations WHERE project = ?1
        )
        SELECT id, content, timestamp, obs_type, file_path, 0.0 AS score
        FROM ranked WHERE rn <= ?2 ORDER BY timestamp DESC"
    )?;
    stmt.query_map(params![project, per_type], |row| Ok(ScoredObs {
        id: row.get(0)?, content: row.get(1)?, timestamp: row.get(2)?,
        obs_type: row.get(3)?, file_path: row.get(4)?, score: row.get(5)?,
    }))?.collect()
}
```

---

## 9. Performance Notes

At nmem's scale, everything is fast:

| Query | ~1K rows | ~10K rows | ~20K rows |
|-------|----------|-----------|-----------|
| FTS5 MATCH + JOIN | <1ms | <1ms | 1-2ms |
| Composite CTE | <1ms | 1-2ms | 2-5ms |
| Recency-only | <1ms | <1ms | <1ms |
| File path LIKE | <1ms | <1ms | <1ms |

Only optimization that matters: `prepare_cached` to avoid re-parsing SQL.

**At 100K+** (unlikely): compound index `(project, timestamp DESC)` becomes load-bearing,
FTS5 `optimize` command needed periodically, CTE should pre-filter by project/recency
before scoring, window functions need bounded time ranges, consider materialized recency
tiers as a column.

### Indexes worth having from the start

```sql
CREATE INDEX idx_obs_project ON observations(project);
CREATE INDEX idx_obs_timestamp ON observations(timestamp DESC);
CREATE INDEX idx_obs_filepath ON observations(file_path) WHERE file_path IS NOT NULL;
CREATE INDEX idx_obs_session ON observations(session_id);
```

Negligible cost at <20K rows. Correct habits for when scale arrives.
