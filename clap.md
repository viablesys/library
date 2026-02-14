# clap Reference

Command line argument parser for Rust using derive macros. Current version: **4.5.x**.

Docs: <https://docs.rs/clap/latest/clap/>

---

## 1. Installation

```toml
[dependencies]
clap = { version = "4.5", features = ["derive", "env", "wrap_help"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
anyhow = "1"

[dev-dependencies]
assert_cmd = "2"
predicates = "3"
```

| Feature | Purpose |
|---------|---------|
| `derive` | `#[derive(Parser)]`, `#[derive(Subcommand)]`, `#[derive(Args)]`, `#[derive(ValueEnum)]` |
| `env` | `#[arg(env = "VAR")]` — populate args from env vars |
| `wrap_help` | Wrap help text to terminal width |
| `cargo` | `command!()` macro — pull name/version/about from `Cargo.toml` |
| `color` | Colored error/help output (on by default) |

---

## 2. Derive API Basics

```rust
use clap::Parser;
use std::path::PathBuf;

#[derive(Parser)]
#[command(name = "nmem", version, about = "Cross-session memory for Claude Code")]
struct Cli {
    input: String,                                          // required positional
    output: Option<String>,                                 // optional positional
    #[arg(short, long)]
    force: bool,                                            // flag
    #[arg(short, long, default_value = "~/.nmem/nmem.db")]
    db: PathBuf,                                            // option with default
    #[arg(long)]
    tag: Vec<String>,                                       // repeatable
    #[arg(short, long, action = clap::ArgAction::Count)]
    verbose: u8,                                            // -v, -vv, -vvv
}
```

### `#[arg]` attributes

| Attribute | Effect |
|-----------|--------|
| `short` / `short = 'D'` | Single-char flag, inferred or explicit |
| `long` / `long = "database"` | Long flag, inferred or explicit |
| `default_value = "x"` | String default |
| `default_value_t = 8080` | Typed default |
| `env = "NMEM_DB"` | Read from env var (requires `env` feature) |
| `value_name = "PATH"` | Placeholder in help text |
| `value_parser` | Custom parsing function |
| `required`, `hide`, `global` | Must provide / omit from help / share with subcommands |
| `action = ArgAction::Count` | Count occurrences (`-vvv` = 3) |
| `action = ArgAction::SetTrue` | Boolean flag |

---

## 3. Subcommands

```rust
use clap::{Parser, Subcommand};

#[derive(Parser)]
#[command(name = "nmem", version, about)]
struct Cli {
    #[command(subcommand)]
    command: Commands,
    #[arg(long, env = "NMEM_DB", global = true)]
    db: Option<PathBuf>,
}

#[derive(Subcommand)]
enum Commands {
    /// Record an observation from a hook event
    Record { #[arg(short, long)] obs_type: String },
    /// Query past observations
    Query { query: String, #[arg(short = 'n', long, default_value_t = 10)] limit: usize },
    /// Run maintenance tasks (vacuum, prune, reindex)
    Maintain { #[arg(long)] task: Option<String> },
    /// Show system status
    Status,
}

fn main() -> anyhow::Result<()> {
    let cli = Cli::parse();
    let db_path = cli.db.unwrap_or_else(default_db_path);
    match cli.command {
        Commands::Record { obs_type } => handle_record(&db_path, &obs_type),
        Commands::Query { query, limit } => handle_query(&db_path, &query, limit),
        Commands::Maintain { task } => handle_maintain(&db_path, task.as_deref()),
        Commands::Status => handle_status(&db_path),
    }
}
```

When a subcommand has many arguments, extract to a struct with `#[derive(clap::Args)]`:

```rust
#[derive(clap::Args)]
struct RecordArgs {
    #[arg(short, long)]
    obs_type: String,
    #[arg(short, long)]
    session_id: String,
}

// Then use: Record(RecordArgs) in the enum
```

---

## 4. Stdin/Stdout Patterns

### Reading JSON from stdin

```rust
use std::io::{self, Read, IsTerminal};

fn read_stdin_json<T: serde::de::DeserializeOwned>() -> anyhow::Result<T> {
    if io::stdin().is_terminal() {
        anyhow::bail!("expected JSON on stdin (pipe input)");
    }
    let mut buf = String::new();
    io::stdin().read_to_string(&mut buf)?;
    Ok(serde_json::from_str(&buf)?)
}
```

`IsTerminal` is stable since Rust 1.70 — no `atty` crate needed.

### Writing structured output

```rust
fn write_stdout_json<T: serde::Serialize>(value: &T) -> anyhow::Result<()> {
    serde_json::to_writer(io::stdout().lock(), value)?;
    println!();
    Ok(())
}
```

### Stderr for diagnostics (critical for hooks where stdout is captured)

```rust
eprintln!("nmem: recording observation {}", obs_id);  // progress
eprintln!("nmem: error: {err:#}");                     // errors
```

---

## 5. Exit Codes

| Code | Meaning | Claude Code hook semantics |
|------|---------|---------------------------|
| 0 | Success | Hook passes, output used |
| 1 | General error | Hook fails, session continues |
| 2 | Usage/blocking error | Hook blocks the action |

### `std::process::ExitCode`

```rust
use std::process::ExitCode;

fn main() -> ExitCode {
    match run() {
        Ok(()) => ExitCode::SUCCESS,
        Err(e) => {
            eprintln!("nmem: {e:#}");
            ExitCode::from(1)
        }
    }
}
```

Map domain errors to codes. Avoid `process::exit()` in library code (skips destructors):

```rust
enum CliError {
    InvalidInput(String),      // exit 2
    Database(rusqlite::Error), // exit 1
    Blocking(String),          // exit 2
}

impl CliError {
    fn exit_code(&self) -> u8 {
        match self {
            Self::InvalidInput(_) | Self::Blocking(_) => 2,
            Self::Database(_) => 1,
        }
    }
}
```

---

## 6. Error Handling

### `anyhow` for CLI tools

```rust
use anyhow::{Context, Result, bail};

fn handle_record(db_path: &Path, obs_type: &str) -> Result<()> {
    let event: HookEvent = read_stdin_json()
        .context("failed to parse hook event from stdin")?;
    let conn = open_db(db_path)
        .context("failed to open database")?;
    if obs_type.is_empty() {
        bail!("observation type must not be empty");
    }
    conn.execute(
        "INSERT INTO observations (type, data) VALUES (?1, ?2)",
        (obs_type, &event.data),
    ).context("failed to insert observation")?;
    Ok(())
}
```

`{e:#}` gives the full chain on one line: `"failed to open database: no such file: /tmp/nmem.db"`.

Never `.unwrap()` in CLI tools — always `.context("msg")?` instead.

---

## 7. Fast Startup

For binaries invoked hundreds of times per session. Measure with `hyperfine --warmup 5 'echo "{}" | ./target/release/nmem record'`. Target: <10ms parse-and-exit, <50ms with SQLite write.

**Principles:**
- Clap parsing is ~microseconds, no concern there
- Don't open DB or read stdin until the subcommand actually needs it
- Skip `tracing` subscriber init unless `-v` is passed
- Skip config file parsing unless the subcommand requires it
- Prefer `rusqlite` with `bundled` (avoids dlopen overhead variance)
- Smaller binaries load faster from cold cache (see Section 10)

---

## 8. Environment Variables

```rust
#[derive(Parser)]
struct Cli {
    #[arg(long, env = "NMEM_DB", default_value = "~/.nmem/nmem.db")]
    db: PathBuf,
    #[arg(long, env = "NMEM_LOG", default_value = "warn")]
    log_level: String,
    #[command(subcommand)]
    command: Commands,
}
```

**Precedence:** CLI args > env vars > defaults (automatic). Help shows: `--db <PATH> [env: NMEM_DB=] [default: ...]`

---

## 9. Testing CLI Binaries

### Unit testing with `try_parse_from`

```rust
#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn parse_record_subcommand() {
        let cli = Cli::try_parse_from([
            "nmem", "--db", "/tmp/test.db", "record", "--obs-type", "decision",
        ]).unwrap();
        assert!(matches!(cli.command, Commands::Record { .. }));
    }

    #[test]
    fn missing_required_arg_fails() {
        assert!(Cli::try_parse_from(["nmem", "record"]).is_err());
    }
}
```

### Integration testing with `assert_cmd`

```rust
// tests/cli.rs
use assert_cmd::Command;
use predicates::prelude::*;

#[test]
fn status_exits_zero() {
    Command::cargo_bin("nmem").unwrap()
        .arg("status").assert().success();
}

#[test]
fn record_reads_stdin_json() {
    Command::cargo_bin("nmem").unwrap()
        .args(["record", "--obs-type", "decision"])
        .write_stdin(r#"{"event":"tool_use","tool":"Read","data":{}}"#)
        .assert().success();
}

#[test]
fn record_rejects_invalid_json() {
    Command::cargo_bin("nmem").unwrap()
        .args(["record", "--obs-type", "decision"])
        .write_stdin("not json").assert().failure()
        .stderr(predicate::str::contains("invalid JSON"));
}

#[test]
fn blocking_error_exits_2() {
    Command::cargo_bin("nmem").unwrap()
        .args(["record", "--obs-type", ""])
        .write_stdin("{}").assert().code(2);
}
```

---

## 10. Cargo Profile for CLI Tools

```toml
[profile.release]
opt-level = "z"       # Optimize for size (faster cold-cache startup)
lto = true            # Link-time optimization
codegen-units = 1     # Better optimization, slower compile
strip = true          # Strip debug symbols
panic = "abort"       # No unwind tables
```

| Setting | Benefit | Cost |
|---------|---------|------|
| `opt-level = "z"` | Smallest binary | Slightly slower hot-path vs `3` |
| `lto = true` | Smaller + faster | Longer compile |
| `codegen-units = 1` | Better optimization | Longer compile |
| `strip = true` | Smaller binary | No debug symbols in crashes |
| `panic = "abort"` | Smaller binary | No `catch_unwind` |

Typical sizes: dev ~15MB, release defaults ~4MB, optimized release ~1.5MB.

For hook handlers, prefer `opt-level = "z"` since cold-cache load time dominates. For compute-heavy CLIs, use `opt-level = 3`.
