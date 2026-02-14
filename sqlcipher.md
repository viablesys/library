# SQLCipher + rusqlite Reference

SQLCipher: standalone SQLite fork with 256-bit AES encryption. Transparent page-level encryption with HMAC tamper detection.

Docs: <https://www.zetetic.net/sqlcipher/sqlcipher-api/> | Repo: <https://github.com/sqlcipher/sqlcipher>

---

## 1. Installation

### Bundled SQLCipher (uses system OpenSSL)

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled-sqlcipher"] }
```

Requires OpenSSL dev headers: `sudo dnf install openssl-devel` (Fedora).

### Fully Vendored (no system crypto dependency)

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["bundled-sqlcipher-vendored-openssl"] }
```

Compiles OpenSSL from source. Slower build, zero system deps.

### Feature Exclusivity

`bundled-sqlcipher` implies `bundled`. You **cannot** use both `bundled` and `bundled-sqlcipher` -- they are the same underlying SQLite source, but `bundled-sqlcipher` compiles with encryption enabled. If another dependency pulls in `rusqlite` with `bundled`, you'll get a conflict. Unify features in `[dependencies]` or use `default-features = false`.

### System-installed SQLCipher

```toml
[dependencies]
rusqlite = { version = "0.38.0", features = ["sqlcipher"] }
```

Links against system `libsqlcipher`. Set `SQLCIPHER_STATIC=1` for static linking.

---

## 2. Opening an Encrypted Database

`PRAGMA key` **must** be the first statement after opening. Any other operation first will treat the file as unencrypted.

### Passphrase Key (PBKDF2-derived)

```rust
use rusqlite::{Connection, Result};

fn open_encrypted(path: &str, passphrase: &str) -> Result<Connection> {
    let conn = Connection::open(path)?;
    conn.pragma_update(None, "key", passphrase)?;

    // Verify the key works -- wrong key fails here, not on open
    conn.query_row("SELECT count(*) FROM sqlite_master", [], |_| Ok(()))?;
    Ok(conn)
}
```

The passphrase is run through PBKDF2 (256000 iterations by default) to derive the encryption key.

### Raw Hex Key (no KDF)

```rust
// 64 hex chars = 32 bytes = 256-bit key, bypass PBKDF2
conn.execute_batch("PRAGMA key = \"x'2DD29CA851E7B56E4697B0E1F08507293D761A05CE4D1B628663F411A8086D99'\";")?;
```

Use raw keys when you manage key derivation externally (e.g., from a hardware keyring).

---

## 3. Configuration PRAGMAs

**Order matters**: set cipher configuration PRAGMAs *before* `PRAGMA key`.

```rust
fn open_configured(path: &str, passphrase: &str) -> Result<Connection> {
    let conn = Connection::open(path)?;

    // Configuration BEFORE key
    conn.pragma_update(None, "cipher_page_size", "4096")?;
    conn.pragma_update(None, "kdf_iter", "256000")?;
    conn.pragma_update(None, "cipher_use_hmac", "ON")?;
    conn.pragma_update(None, "cipher_hmac_algorithm", "HMAC_SHA512")?;
    conn.pragma_update(None, "cipher_kdf_algorithm", "PBKDF2_HMAC_SHA512")?;

    // Key AFTER configuration
    conn.pragma_update(None, "key", passphrase)?;
    conn.query_row("SELECT count(*) FROM sqlite_master", [], |_| Ok(()))?;
    Ok(conn)
}
```

| PRAGMA | Default | Notes |
|--------|---------|-------|
| `cipher_page_size` | 4096 | Must be power of 2 (512-65536). Match SQLite page size. |
| `kdf_iter` | 256000 | Higher = slower open, same query speed. |
| `cipher_use_hmac` | ON | Per-page HMAC for tamper detection. Disabling saves ~20% space. |
| `cipher_hmac_algorithm` | HMAC_SHA512 | Options: HMAC_SHA1, HMAC_SHA256, HMAC_SHA512 |
| `cipher_kdf_algorithm` | PBKDF2_HMAC_SHA512 | Options: PBKDF2_HMAC_SHA1, PBKDF2_HMAC_SHA256, PBKDF2_HMAC_SHA512 |
| `cipher_plaintext_header_size` | 0 | Bytes of unencrypted header. Set to 32 for WAL-mode databases if the VFS needs to read the header (unusual). |

---

## 4. Key Management

### Re-key (change passphrase)

```rust
fn rekey_database(conn: &Connection, new_passphrase: &str) -> Result<()> {
    conn.pragma_update(None, "rekey", new_passphrase)?;
    Ok(())
}
```

Re-encrypts the entire database. Proportional to database size. Must already be unlocked with the current key.

### sqlite3_key API vs PRAGMA

`PRAGMA key` and the C-level `sqlite3_key()` are equivalent. rusqlite's `pragma_update` wraps the PRAGMA approach. No functional difference.

### Where to Store the Key (local developer tool)

For nmem / local-first tools, ranked by practicality:

1. **Environment variable** (`NMEM_DB_KEY`) -- simplest, works in systemd units and shells
2. **OS keyring** via `keyring` crate -- survives reboots, no plaintext on disk
3. **Prompt at startup** -- most secure, worst UX for a background service
4. **Config file with restricted perms** (`chmod 600`) -- acceptable for single-user dev tools

Never hardcode the key. Never commit it.

---

## 5. Encrypting an Existing Database

### Using sqlcipher_export (recommended)

```rust
fn encrypt_existing(plaintext_path: &str, encrypted_path: &str, passphrase: &str) -> Result<()> {
    let conn = Connection::open(plaintext_path)?;

    // Attach the new encrypted database
    conn.execute(
        "ATTACH DATABASE ?1 AS encrypted",
        [encrypted_path],
    )?;

    // Set encryption key on the attached database
    conn.pragma_update(Some("encrypted"), "key", passphrase)?;

    // Export all schema + data
    conn.query_row("SELECT sqlcipher_export('encrypted')", [], |_| Ok(()))?;

    conn.execute_batch("DETACH DATABASE encrypted")?;
    Ok(())
}
```

### Using ATTACH + manual copy

```rust
fn encrypt_via_attach(plaintext_path: &str, encrypted_path: &str, passphrase: &str) -> Result<()> {
    let conn = Connection::open(plaintext_path)?;
    conn.execute("ATTACH DATABASE ?1 AS enc", [encrypted_path])?;
    conn.pragma_update(Some("enc"), "key", passphrase)?;

    // Copy each table manually
    let tables: Vec<String> = conn
        .prepare("SELECT name FROM sqlite_master WHERE type='table'")?
        .query_map([], |row| row.get(0))?
        .collect::<Result<_>>()?;

    for table in &tables {
        conn.execute_batch(&format!(
            "CREATE TABLE enc.{t} AS SELECT * FROM main.{t}",
            t = table
        ))?;
    }

    conn.execute_batch("DETACH DATABASE enc")?;
    Ok(())
}
```

`sqlcipher_export` is preferred -- it copies indices, triggers, and views automatically.

---

## 6. Compatibility

| Concern | Status |
|---------|--------|
| Standard `sqlite3` CLI | Cannot read encrypted files. Use `sqlcipher` CLI. |
| WAL mode | Works. Set `PRAGMA journal_mode = WAL` after `PRAGMA key`. |
| FTS5 | Works. Bundled in `bundled-sqlcipher`. |
| `sqlite-vec` extension | Works with `load_extension` feature. Extensions operate on decrypted pages in memory. |
| `rusqlite_migration` | Works. Set key before calling `to_latest()`. |
| Litestream | Replicates encrypted bytes. Restore produces an encrypted copy (needs key to read). |

### sqlcipher CLI

```bash
# Fedora: not packaged, build from source or use container
sqlcipher encrypted.db
sqlite> PRAGMA key = 'passphrase';
sqlite> .tables
```

---

## 7. Performance

| Aspect | Impact |
|--------|--------|
| Overall throughput | ~5-15% overhead vs unencrypted SQLite |
| Database open | Dominated by KDF (256000 PBKDF2 iterations). ~50-200ms depending on hardware. |
| Query execution | Negligible. AES-256-CBC per page is fast on modern CPUs. |
| Page encryption | Each 4096-byte page encrypted/decrypted independently. |
| SQLite page cache | Caches **decrypted** pages. Cache effectiveness unchanged. |
| File size | ~same with HMAC on (HMAC stored in reserved bytes per page). |

To reduce open latency for CLI tools, lower `kdf_iter` (tradeoff: weaker brute-force resistance) or use a raw hex key (skips KDF entirely).

---

## 8. Secure Delete

```rust
conn.pragma_update(None, "secure_delete", "ON")?;
```

| Setting | Behavior |
|---------|----------|
| `secure_delete = ON` | Zero-fills freed pages. ~5-10% slower deletes. |
| `secure_delete = FAST` | Zero-fills only without increasing I/O (skips overflow pages). |
| `secure_delete = OFF` | Default. Freed pages retain old content until overwritten. |

### WAL Caveat

Deleted data may persist in WAL frames until checkpoint. After purging sensitive data:

```rust
conn.pragma_update(None, "secure_delete", "ON")?;
conn.execute("DELETE FROM secrets WHERE id = ?1", [id])?;
conn.execute_batch("PRAGMA wal_checkpoint(TRUNCATE)")?;
```

### Relevance to nmem

If a secret is accidentally stored as an observation, `secure_delete + TRUNCATE checkpoint + VACUUM` is the best-effort purge. Encryption makes the on-disk residue unreadable without the key regardless.

---

## 9. Pitfalls

**Forgetting PRAGMA key**: Database appears empty or corrupt. Every connection must set the key before any other operation.

**Wrong key detection**: `PRAGMA key` always succeeds. The error (`SQLITE_NOTADB`) surfaces on the first real query, not on open.

```rust
// This does NOT fail with wrong key:
conn.pragma_update(None, "key", "wrong_password")?;
// This DOES fail:
conn.query_row("SELECT count(*) FROM sqlite_master", [], |_| Ok(()))?;
// Error: SqliteFailure(... SQLITE_NOTADB ...)
```

**Feature conflict**: `bundled-sqlcipher` and `bundled` both provide `libsqlite3-sys`. If a transitive dependency enables `bundled`, add a `[patch]` or unify features.

**PRAGMA order**: Cipher config PRAGMAs (`kdf_iter`, `cipher_page_size`, etc.) must come *before* `PRAGMA key`. After key is set, they are read-only.

**integrity_check**: Requires key to be set first. Without it, returns false positives.

**r2d2 pool init**: Set key in the `with_init` callback, not after `pool.get()`.

```rust
let manager = SqliteConnectionManager::file("encrypted.db")
    .with_init(|conn| {
        conn.execute_batch("PRAGMA key = 'passphrase';")?;
        conn.execute_batch("PRAGMA journal_mode = WAL;")?;
        Ok(())
    });
```

**Testing**: Use a test helper that creates an in-memory encrypted database:

```rust
#[cfg(test)]
fn test_conn() -> Connection {
    let conn = Connection::open_in_memory().unwrap();
    conn.pragma_update(None, "key", "test-key").unwrap();
    conn
}
```
