# serde + serde_json Reference

Rust serialization framework and JSON data format. Current versions: **serde 1.x**, **serde_json 1.x**.

Docs: <https://serde.rs/> | <https://docs.rs/serde_json/latest/serde_json/>

---

## 1. Installation

```toml
[dependencies]
serde = { version = "1", features = ["derive"] }
serde_json = "1"
```

### Feature Flags

**serde:**

| Feature | Purpose |
|---------|---------|
| `derive` | `#[derive(Serialize, Deserialize)]` proc macros |
| `std` | **(default)** Standard library support |
| `alloc` | `no_std` with allocator (enables String, Vec, etc.) |
| `rc` | Serialize/deserialize `Rc<T>` and `Arc<T>` |

**serde_json:**

| Feature | Purpose |
|---------|---------|
| `std` | **(default)** Standard library, includes `from_reader`/`to_writer` |
| `preserve_order` | `Map<String, Value>` uses `IndexMap` (insertion order) |
| `arbitrary_precision` | Numbers stored as strings internally, no precision loss |
| `raw_value` | `RawValue` type for deferred parsing |
| `unbounded_depth` | Disable recursion limit (default: 128) |

---

## 2. Derive Macros

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Debug)]
struct Observation {
    id: i64,
    content: String,
    timestamp: i64,
}
```

### Container Attributes

```rust
#[derive(Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]       // sessionId, startTime, etc.
#[serde(deny_unknown_fields)]            // reject unknown keys on deserialize
struct SessionSummary {
    session_id: String,
    start_time: i64,
}
```

`rename_all` options: `"camelCase"`, `"snake_case"`, `"PascalCase"`, `"SCREAMING_SNAKE_CASE"`, `"kebab-case"`.

### Field Attributes

```rust
#[derive(Serialize, Deserialize)]
struct Observation {
    id: i64,
    #[serde(rename = "type")]              // different JSON key
    obs_type: String,
    #[serde(default)]                      // Default::default() if missing
    score: f64,
    #[serde(skip)]                         // excluded entirely
    internal_cache: Vec<u8>,
    #[serde(skip_serializing)]             // deserialize only
    raw_input: String,
    #[serde(skip_deserializing)]           // serialize only
    computed_hash: String,
    #[serde(skip_serializing_if = "Option::is_none")]  // omit None
    parent_id: Option<i64>,
    #[serde(flatten)]                      // inline nested struct
    metadata: Metadata,
    #[serde(with = "timestamp_secs")]      // custom ser/de module
    created_at: std::time::SystemTime,
}
```

### Variant Attributes

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum ObsType {
    #[serde(rename = "bug_fix")]    // custom discriminator
    BugFix { description: String },
    Feature { description: String },
    #[serde(other)]                 // catch-all (must be unit variant)
    Unknown,
}
```

---

## 3. Enum Representations

### Externally Tagged (default)

```rust
#[derive(Serialize, Deserialize)]
enum Event {
    Insert { table: String, rowid: i64 },
}
// {"Insert": {"table": "obs", "rowid": 1}}
```

### Internally Tagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "type")]
enum Observation {
    Decision { content: String, rationale: String },
    BugFix { content: String, file: String },
}
// {"type": "Decision", "content": "...", "rationale": "..."}
```

Cannot contain tuple variants. Tag is embedded in the data.

### Adjacently Tagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(tag = "t", content = "c")]
enum Message {
    Text(String),
    Data { key: String, value: serde_json::Value },
}
// {"t": "Text", "c": "hello"}
```

### Untagged

```rust
#[derive(Serialize, Deserialize)]
#[serde(untagged)]
enum FlexValue { Int(i64), Float(f64), Text(String) }
// 42  or  3.14  or  "hello"
```

Tries variants in order. No tag in output. Poor error messages (reports last variant's failure).

### When to Use Each

| Representation | Use case |
|----------------|----------|
| Externally tagged | Simple enums, Rust-to-Rust |
| **Internally tagged** | **APIs, stored data (nmem observations) -- human-readable, self-describing** |
| Adjacently tagged | Mixed tuple + struct variants |
| Untagged | Accepting multiple input formats, union types |

### Sibling-Field Dispatch (Heterogeneous Payloads)

When the discriminant is a sibling field rather than inside the variant data — common in hook payloads, API responses, and event streams where `tool_name` determines the shape of `tool_input`:

```json
{"tool_name": "Bash", "tool_input": {"command": "ls"}}
{"tool_name": "Read", "tool_input": {"file_path": "/foo.rs"}}
{"tool_name": "Edit", "tool_input": {"file_path": "/foo.rs", "old_string": "x", "new_string": "y"}}
```

**Approach 1: Deserialize to Value, dispatch manually** (simplest)

```rust
#[derive(Deserialize)]
struct HookPayload {
    tool_name: String,
    tool_input: serde_json::Value,
    tool_response: Option<String>,
}

fn extract(payload: &HookPayload) -> Option<Observation> {
    match payload.tool_name.as_str() {
        "Bash" => {
            let cmd = payload.tool_input.get("command")?.as_str()?;
            Some(Observation::command(cmd))
        }
        "Read" => {
            let path = payload.tool_input.get("file_path")?.as_str()?;
            Some(Observation::file_read(path))
        }
        "Write" | "Edit" => {
            let path = payload.tool_input.get("file_path")?.as_str()?;
            Some(Observation::file_write(path))
        }
        _ => None, // Unknown tool — skip
    }
}
```

**Approach 2: Typed variants with `from_value`** (type-safe per tool)

```rust
#[derive(Deserialize)]
struct BashInput { command: String }

#[derive(Deserialize)]
struct ReadInput { file_path: String }

#[derive(Deserialize)]
struct EditInput { file_path: String, old_string: String, new_string: String }

fn extract_typed(payload: &HookPayload) -> Result<Observation, serde_json::Error> {
    match payload.tool_name.as_str() {
        "Bash" => {
            let input: BashInput = serde_json::from_value(payload.tool_input.clone())?;
            Ok(Observation::command(&input.command))
        }
        "Read" => {
            let input: ReadInput = serde_json::from_value(payload.tool_input.clone())?;
            Ok(Observation::file_read(&input.file_path))
        }
        // ...
    }
}
```

Approach 2 gives better error messages (field-level type mismatches) but clones the `Value`. For small payloads like hook events, the clone cost is negligible. Use approach 1 when you only need one or two fields; approach 2 when you need validated, complete input.

**Approach 3: `#[serde(flatten)]` with untagged enum** (fully automatic, fragile)

```rust
#[derive(Deserialize)]
struct HookEvent {
    tool_name: String,
    #[serde(flatten)]
    input: ToolInput,
}

#[derive(Deserialize)]
#[serde(untagged)]
enum ToolInput {
    Bash { command: String },
    Read { file_path: String },
    // Ambiguous: Edit also has file_path — untagged tries in order
}
```

Avoid approach 3. Untagged enums try variants in declaration order, so overlapping fields (both `Read` and `Edit` have `file_path`) cause mismatches. Error messages are poor ("data did not match any variant"). Use approaches 1 or 2 instead.

---

## 4. serde_json::Value

```rust
use serde_json::Value;

let v: Value = serde_json::from_str(r#"{"name": "Alice", "score": 42}"#)?;

// Indexing -- returns Value::Null for missing keys (never panics)
let name = &v["name"];
let nested = &v["a"]["b"]["c"];  // Value::Null if any level missing

// Type checking and extraction
v["name"].as_str();      // Option<&str>
v["score"].as_i64();     // Option<i64>
v["score"].as_f64();     // Option<f64>
v["active"].as_bool();   // Option<bool>
v["tags"].as_array();    // Option<&Vec<Value>>
v["meta"].as_object();   // Option<&Map<String, Value>>
v["name"].is_string();   // bool
v["score"].is_null();    // bool
```

### json! Macro

```rust
use serde_json::json;

let obs_type = "decision";
let val = json!({
    "type": obs_type,
    "tags": ["architecture", "rust"],
    "metadata": {"session": 42, "reviewed": false},
    "parent": null
});
```

### Mutating

```rust
let mut v = json!({"tags": ["rust"], "count": 0});

if let Some(obj) = v.as_object_mut() {
    obj.insert("new_key".to_string(), json!("value"));
    obj.remove("count");
}
if let Some(arr) = v["tags"].as_array_mut() {
    arr.push(json!("sqlite"));
}
v["count"] = json!(5);  // direct index assignment
```

---

## 5. Serialization / Deserialization

```rust
let obs = Observation { id: 1, content: "found bug".into(), score: 0.9 };

// Serialize
let s: String = serde_json::to_string(&obs)?;          // compact
let s: String = serde_json::to_string_pretty(&obs)?;    // indented
let b: Vec<u8> = serde_json::to_vec(&obs)?;             // bytes
serde_json::to_writer(&mut buf, &obs)?;                  // to Write impl

// Deserialize
let obs: Observation = serde_json::from_str(&s)?;
let obs: Observation = serde_json::from_slice(&b)?;
let obs: Observation = serde_json::from_reader(std::io::BufReader::new(file))?;

// Value conversions (no intermediate string)
let val: serde_json::Value = serde_json::to_value(&obs)?;  // struct -> Value
let obs: Observation = serde_json::from_value(val)?;        // Value -> struct
```

### Streaming Deserializer

Newline-delimited JSON without loading all into memory.

```rust
let data = r#"{"id":1,"content":"a","score":0.1}
{"id":2,"content":"b","score":0.2}"#;

let stream = serde_json::Deserializer::from_str(data).into_iter::<Observation>();
for result in stream {
    let obs = result?;
    println!("{}: {}", obs.id, obs.content);
}
// Also: Deserializer::from_reader(reader).into_iter::<T>()
```

---

## 6. Custom Serialization

### `#[serde(with = "module")]` -- DateTime as Timestamp

Module must expose `serialize` and `deserialize` with specific signatures.

```rust
use serde::{self, Serialize, Deserialize, Serializer, Deserializer};
use std::time::{SystemTime, UNIX_EPOCH};

mod timestamp_secs {
    use super::*;
    pub fn serialize<S: Serializer>(time: &SystemTime, ser: S) -> Result<S::Ok, S::Error> {
        let secs = time.duration_since(UNIX_EPOCH)
            .map_err(serde::ser::Error::custom)?.as_secs();
        ser.serialize_u64(secs)
    }
    pub fn deserialize<'de, D: Deserializer<'de>>(de: D) -> Result<SystemTime, D::Error> {
        let secs = u64::deserialize(de)?;
        Ok(UNIX_EPOCH + std::time::Duration::from_secs(secs))
    }
}

#[derive(Serialize, Deserialize)]
struct Observation {
    id: i64,
    #[serde(with = "timestamp_secs")]
    created_at: SystemTime,  // {"id": 1, "created_at": 1700000000}
}
```

For `Option<SystemTime>`, use a similar module handling `None` via `serialize_none()` / `Option::<u64>::deserialize`.

### Manual Impl (brief)

Implement `Serialize`/`Deserialize` directly when derive + attributes are insufficient. Use `serializer.serialize_struct()` for struct-like output, `serializer.serialize_seq()` for arrays. Prefer `#[serde(with)]` when possible.

---

## 7. Error Handling

| Method | Cause |
|--------|-------|
| `is_syntax()` | Invalid JSON (missing comma, bad escape) |
| `is_data()` | Valid JSON, wrong shape (missing field, type mismatch) |
| `is_eof()` | Input ended unexpectedly |
| `is_io()` | Underlying I/O error (`from_reader`/`to_writer` only) |

`e.line()` and `e.column()` pinpoint the failure position.

```rust
match serde_json::from_str::<Observation>(input) {
    Ok(obs) => { /* use obs */ }
    Err(e) => {
        eprintln!("error at {}:{}: {e}", e.line(), e.column());
        // e.is_syntax(), e.is_data(), e.is_eof(), e.is_io()
    }
}

// With anyhow -- include position in context
let config: Config = serde_json::from_str(&text)
    .map_err(|e| anyhow::anyhow!("parse {path} at {}:{}: {e}", e.line(), e.column()))?;
```

---

## 8. Patterns for SQLite + JSON

### Storing serde_json::Value in SQLite

Requires rusqlite `serde_json` feature for `ToSql`/`FromSql` on `Value`.

```toml
rusqlite = { version = "0.38", features = ["bundled", "serde_json"] }
```

```rust
let metadata = json!({"file": "main.rs", "line": 42});
conn.execute("INSERT INTO obs (content, metadata) VALUES (?1, ?2)",
    params!["found memory leak", &metadata])?;

let meta: serde_json::Value = conn.query_row(
    "SELECT metadata FROM obs WHERE id = ?1", [1], |row| row.get(0))?;
```

### Flexible Metadata: Typed Struct + JSON Extras

`#[serde(flatten)]` captures known fields, unknown fields overflow into a map.

```rust
#[derive(Serialize, Deserialize, Debug)]
struct Observation {
    id: i64,
    #[serde(rename = "type")]
    obs_type: String,
    content: String,
    #[serde(default, skip_serializing_if = "HashMap::is_empty", flatten)]
    extra: HashMap<String, Value>,  // catches "file", "severity", etc.
}
```

### Observation Pattern: Fixed Schema + JSON Column

Structured fields as SQL columns (queryable, indexable), variable data as JSON TEXT.

```rust
// Table: id INTEGER PRIMARY KEY, session_id INTEGER, obs_type TEXT,
//        content TEXT, metadata TEXT (JSON), created_at INTEGER

// Internally tagged enum for observation types
#[derive(Serialize, Deserialize, Debug)]
#[serde(tag = "type")]
enum ObsType {
    #[serde(rename = "decision")]  Decision { rationale: String },
    #[serde(rename = "bugfix")]    BugFix { file: String, line: Option<u32> },
    #[serde(other)]                Unknown,
}

fn insert_obs(conn: &Connection, session_id: i64, obs_type: &str,
              content: &str, metadata: &Value) -> rusqlite::Result<i64> {
    conn.execute("INSERT INTO observations (session_id, obs_type, content, metadata)
         VALUES (?1, ?2, ?3, ?4)", params![session_id, obs_type, content, metadata])?;
    Ok(conn.last_insert_rowid())
}

// Query JSON fields with json_extract
fn find_by_file(conn: &Connection, file: &str) -> rusqlite::Result<Vec<(i64, String)>> {
    let mut stmt = conn.prepare(
        "SELECT id, content FROM observations WHERE json_extract(metadata, '$.file') = ?1")?;
    stmt.query_map([file], |row| Ok((row.get(0)?, row.get(1)?)))?.collect()
}
```

### Full Struct as JSON Column

```rust
fn insert_typed(conn: &Connection, obs: &Observation) -> rusqlite::Result<i64> {
    let json = serde_json::to_string(obs)
        .map_err(|e| rusqlite::Error::ToSqlConversionFailure(e.into()))?;
    conn.execute("INSERT INTO obs_raw (data) VALUES (?1)", [&json])?;
    Ok(conn.last_insert_rowid())
}

fn load_typed(conn: &Connection, id: i64) -> rusqlite::Result<Observation> {
    conn.query_row("SELECT data FROM obs_raw WHERE id = ?1", [id], |row| {
        let s: String = row.get(0)?;
        serde_json::from_str(&s).map_err(|e|
            rusqlite::Error::FromSqlConversionFailure(0, rusqlite::types::Type::Text, e.into()))
    })
}
```
