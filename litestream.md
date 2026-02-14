# Litestream Reference

Streaming replication for SQLite databases. Runs as a standalone background process,
continuously replicating WAL changes to S3, GCS, Azure Blob Storage, SFTP, or local
file replicas. Provides disaster recovery with point-in-time restore.

- Repo: https://github.com/benbjohnson/litestream
- Docs: https://litestream.io
- License: Apache 2.0
- Written in: Go
- Latest: v0.5.6

---

## Table of Contents

1. [How It Works](#how-it-works)
2. [Installation](#installation)
3. [Configuration](#configuration)
4. [S3 Configuration](#s3-configuration)
5. [S3-Compatible Providers](#s3-compatible-providers)
6. [Cloudflare R2](#cloudflare-r2)
7. [Commands](#commands)
8. [Running as a Service](#running-as-a-service)
9. [Restore](#restore)
10. [Docker Patterns](#docker-patterns)
11. [Kubernetes](#kubernetes)
12. [WAL Mode Requirements](#wal-mode-requirements)
13. [Monitoring](#monitoring)
14. [Limitations](#limitations)
15. [Integration with Rust](#integration-with-rust)

---

## How It Works

### WAL Streaming

SQLite WAL mode writes page changes to a separate `-wal` file before checkpointing
them back into the main database. Litestream exploits this by taking over the
checkpointing process:

1. Opens a long-running read transaction to prevent SQLite from checkpointing
2. Continuously copies new WAL pages to a **shadow WAL** (staging area)
3. Manually triggers checkpoints when ready, ensuring no pages are missed
4. Streams shadow WAL segments to configured replica destinations

Default polling interval: every 1 second.

### Shadow WAL

The shadow WAL is a hidden directory next to your database file. For a database at
`/var/lib/my.db`, the shadow WAL lives at `/var/lib/.my.db-litestream`.

Shadow WAL files are sequentially numbered (`00000000.wal`, `00000001.wal`, etc.).
Each checkpoint advances to a new WAL file. Original WAL frames and checksums are
preserved for consistency verification. Files are removed after successful replication.

### Generations

A **generation** is a contiguous set of snapshots + WAL files that can fully restore a
database. Each generation has a random 16-character hex ID.

- New generation created when Litestream first starts replicating
- New generation forced if WAL continuity breaks (e.g., Litestream was stopped)
- Snapshot taken at generation start, WAL files numbered with 8-char incrementing hex
- Two servers accidentally sharing a replica path won't overwrite each other

### Snapshots and Retention

Snapshots are periodic full copies of the database. Restore time is proportional to
WAL files accumulated since the last snapshot.

- `snapshot.interval` (default: `1h`) -- how often to snapshot
- `snapshot.retention` (default: `24h`) -- how long to keep snapshots
- At least one snapshot always retained
- Retention enforcement removes old snapshots and their associated WAL files

### Replication Model

- **Asynchronous** -- changes replicated out-of-band (default: every 1 second)
- **Single-writer** -- only one process writes; Litestream is a separate reader
- **No write lock contention** -- Litestream uses read transactions; application
  writers only briefly blocked during checkpoint (mitigated by `busy_timeout`)

---

## Installation

### Debian/Ubuntu (.deb)

```bash
wget https://github.com/benbjohnson/litestream/releases/download/v0.5.6/litestream-0.5.6-linux-x86_64.deb
sudo dpkg -i litestream-0.5.6-linux-x86_64.deb
```

### Fedora/RHEL (.rpm)

```bash
wget https://github.com/benbjohnson/litestream/releases/download/v0.5.6/litestream-0.5.6-linux-x86_64.rpm
sudo dnf install litestream-0.5.6-linux-x86_64.rpm
```

### Homebrew (macOS/Linux)

```bash
brew tap benbjohnson/litestream
brew install litestream
```

### Binary Download

Download from https://github.com/benbjohnson/litestream/releases for:
- Linux: amd64, arm64, armv6, armv7
- macOS: amd64, arm64
- Windows: amd64 (unofficial, community-supported)

Extract and move to PATH:

```bash
tar -xzf litestream-v0.5.6-linux-amd64.tar.gz
sudo mv litestream /usr/local/bin/
```

### Docker

```bash
docker pull litestream/litestream
docker run litestream/litestream version
```

### Build from Source

Requires Go toolchain (1.24+):

```bash
git clone https://github.com/benbjohnson/litestream.git
cd litestream
go install ./cmd/litestream
# Binary installed to $GOPATH/bin/litestream
```

Optional VFS extension (requires CGO):

```bash
make vfs   # produces dist/litestream-vfs.so
```

### Verify

```bash
litestream version
```

---

## Configuration

Default config path: `/etc/litestream.yml` (must be `.yml`, not `.yaml`).

Override with: `litestream replicate -config /path/to/litestream.yml`

Environment variable `LITESTREAM_CONFIG` also sets the config path.

### Full Structure

```yaml
# --- Global settings ---
access-key-id: ${AWS_ACCESS_KEY_ID}
secret-access-key: ${AWS_SECRET_ACCESS_KEY}

# Prometheus metrics endpoint (disabled by default)
addr: ":9090"

# Logging
logging:
  level: info          # debug, info, warn, error
  type: text           # text or json
  stderr: false

# Snapshot management
snapshot:
  interval: 1h
  retention: 24h

# --- Databases ---
dbs:
  - path: /var/lib/app.db
    monitor-interval: 1s          # how often to check for WAL changes
    checkpoint-interval: 1m       # how often to checkpoint
    busy-timeout: 1s              # SQLite busy timeout
    min-checkpoint-page-count: 1000  # pages before PASSIVE checkpoint
    meta-path: /var/lib/.app.db-litestream  # shadow WAL location

    replica:
      type: s3                    # s3, gs, abs, sftp, file, nats, oss, webdav
      url: s3://mybucket/app
      sync-interval: 1s           # how often to push to replica
      # type-specific fields below...

  # Directory mode: replicate all matching databases
  - dir: /var/lib/tenants
    pattern: "*.db"
    watch: true
    replica:
      url: s3://mybucket/tenants
```

### Environment Variable Expansion

Config files support `$VAR` and `${VAR}` expansion. Use `-no-expand-env` to disable.

### Supported Replica Types

| Type | Field | Description |
|------|-------|-------------|
| `s3` | `bucket`, `path`, `region`, `endpoint`, `force-path-style` | AWS S3 and S3-compatible |
| `gs` | `bucket`, `path` | Google Cloud Storage |
| `abs` | `account-name`, `account-key`, `bucket`, `path` | Azure Blob Storage |
| `sftp` | `host`, `user`, `password`, `key-path`, `path` | SFTP server |
| `file` | `path` | Local filesystem |
| `nats` | `bucket`, `username`, `password`, TLS fields | NATS JetStream |
| `oss` | `bucket`, `region`, `path` | Alibaba Cloud OSS |
| `webdav` | `webdav-url`, `webdav-username`, `webdav-password`, `path` | WebDAV server |

### Global Replica Defaults

Settings at the top level apply to all replicas unless overridden:

```yaml
region: us-west-2
sync-interval: 1s
access-key-id: ${AWS_ACCESS_KEY_ID}
secret-access-key: ${AWS_SECRET_ACCESS_KEY}
```

---

## S3 Configuration

### Minimal (URL format)

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      url: s3://mybucket/app
```

### Expanded

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      type: s3
      bucket: mybucket
      path: app
      region: us-east-1
      access-key-id: ${AWS_ACCESS_KEY_ID}
      secret-access-key: ${AWS_SECRET_ACCESS_KEY}
```

### S3 Replica Fields

| Field | Description | Default |
|-------|-------------|---------|
| `bucket` | S3 bucket name | required |
| `path` | Path prefix within bucket | required |
| `region` | AWS region | auto-detected |
| `endpoint` | Custom endpoint URL (for S3-compatible) | AWS default |
| `force-path-style` | Use path-style URLs instead of virtual-hosted | `false` (auto `true` if `endpoint` set) |
| `skip-verify` | Skip TLS verification (dev only) | `false` |
| `access-key-id` | AWS access key | global / env var |
| `secret-access-key` | AWS secret key | global / env var |
| `part-size` | Multipart upload part size | `5MB` |
| `concurrency` | Parallel upload concurrency | `5` |

### Credential Resolution Order

1. Per-replica `access-key-id` / `secret-access-key`
2. Global config `access-key-id` / `secret-access-key`
3. `LITESTREAM_ACCESS_KEY_ID` / `LITESTREAM_SECRET_ACCESS_KEY`
4. `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`
5. AWS IAM instance profile / role

### IAM Policy (Minimal)

Replication (read-write):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetBucketLocation", "s3:ListBucket"],
      "Resource": "arn:aws:s3:::MYBUCKET"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:PutObject", "s3:DeleteObject", "s3:GetObject"],
      "Resource": "arn:aws:s3:::MYBUCKET/*"
    }
  ]
}
```

Restore (read-only): remove `s3:PutObject` and `s3:DeleteObject`.

---

## S3-Compatible Providers

Use the `endpoint` field to redirect API requests to non-AWS providers. As of v0.5.0,
Litestream auto-detects many providers and configures `force-path-style` etc.
automatically.

### General Pattern

```yaml
replica:
  type: s3
  bucket: mybucket
  path: db
  endpoint: https://provider-endpoint.com
  region: us-east-1
  access-key-id: ${ACCESS_KEY}
  secret-access-key: ${SECRET_KEY}
```

Or with URL syntax:

```yaml
replica:
  url: s3://mybucket/db?endpoint=provider-endpoint.com
```

### Provider Quick Reference

| Provider | Endpoint | Region |
|----------|----------|--------|
| MinIO | `https://minio.example.com:9000` | `us-east-1` |
| Backblaze B2 | `s3.<region>.backblazeb2.com` | bucket region |
| DigitalOcean Spaces | `<region>.digitaloceanspaces.com` | region code |
| Wasabi | `s3.<region>.wasabisys.com` | region code |
| Linode Object Storage | `<region>.linodeobjects.com` | region code |
| Scaleway | `s3.<region>.scw.cloud` | region code |
| Tigris | `fly.storage.tigris.dev` | `auto` |
| Filebase | auto-detected | -- |
| Cloudflare R2 | `<account-id>.r2.cloudflarestorage.com` | `auto` |

### MinIO Example

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      type: s3
      bucket: mybucket
      path: app
      endpoint: http://localhost:9000
      region: us-east-1
      access-key-id: ${MINIO_ACCESS_KEY}
      secret-access-key: ${MINIO_SECRET_KEY}
```

### Backblaze B2 Example

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      type: s3
      bucket: mybucket
      path: app
      endpoint: s3.us-west-004.backblazeb2.com
      region: us-west-004
      access-key-id: ${B2_KEY_ID}
      secret-access-key: ${B2_APP_KEY}
```

---

## Cloudflare R2

R2 is S3-compatible with zero egress fees. Find your Account ID in the Cloudflare
dashboard under R2.

### Create API Token

1. Go to Cloudflare dashboard > R2 > Manage R2 API Tokens
2. Create token with "Object Read & Write" permissions
3. Save the Access Key ID and Secret Access Key

### Configuration

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      type: s3
      bucket: mybucket
      path: app
      endpoint: ${CF_ACCOUNT_ID}.r2.cloudflarestorage.com
      region: auto
      access-key-id: ${R2_ACCESS_KEY_ID}
      secret-access-key: ${R2_SECRET_ACCESS_KEY}
```

### Notes

- Region must be `auto` (or `us-east-1` for compatibility)
- Endpoint format: `<account-id>.r2.cloudflarestorage.com`
- EU data residency: `<account-id>.eu.r2.cloudflarestorage.com`
- v0.5.0+ auto-detects R2 from endpoint URL
- R2 supports both path-style and virtual-hosted style

---

## Commands

### replicate

Start continuous replication. Primary command for production use.

```bash
# Using config file (default: /etc/litestream.yml)
litestream replicate

# Using custom config
litestream replicate -config /path/to/litestream.yml

# Command line (single database)
litestream replicate /path/to/db s3://mybucket/db

# Wrap an application process (Litestream exits when child exits)
litestream replicate -exec "myapp serve" /path/to/db s3://mybucket/db

# Using config file exec (in litestream.yml: exec: "myapp serve")
litestream replicate
```

Flags:
- `-config PATH` -- config file path (default `/etc/litestream.yml`)
- `-exec CMD` -- run subprocess; Litestream exits when child exits
- `-no-expand-env` -- disable env var expansion in config

**Important**: flags must appear before positional arguments.

### restore

Restore a database from a replica.

```bash
# Restore to original path (from config)
litestream restore /path/to/db

# Restore to a different path
litestream restore -o /tmp/restored.db /path/to/db

# Restore from replica URL directly
litestream restore -o /tmp/restored.db s3://mybucket/db

# Point-in-time restore
litestream restore -timestamp 2024-01-15T10:30:00Z /path/to/db

# Restore by transaction ID (v0.5.0+)
litestream restore -txid 0000003a /path/to/db

# Skip if database already exists (useful for init scripts)
litestream restore -if-db-not-exists /path/to/db

# Skip if no replicas available (useful for first deploy)
litestream restore -if-replica-exists /path/to/db
```

Flags:
- `-o PATH` -- output path (default: original database path)
- `-if-db-not-exists` -- exit 0 if database already exists
- `-if-replica-exists` -- exit 0 if no replicas found
- `-timestamp TIMESTAMP` -- restore to specific point in time (RFC 3339)
- `-txid TXID` -- restore to specific transaction ID (hex)
- `-parallelism NUM` -- concurrent WAL downloads (default: 8)
- `-config PATH` -- config file path
- `-no-expand-env` -- disable env var expansion

**Safety**: restore refuses to overwrite an existing database file.

### databases

List configured databases and their replicas.

```bash
litestream databases
litestream databases -config /path/to/litestream.yml
```

Output:

```
path         replicas
/var/lib/db  s3
```

### generations

List available generations for a database.

```bash
litestream generations /path/to/db
litestream generations s3://mybucket/db
```

### snapshots

List available snapshots for a database.

```bash
litestream snapshots /path/to/db
litestream snapshots s3://mybucket/db
```

### wal

List WAL files for a database (deprecated in v0.5.0, replaced by `ltx`).

```bash
litestream wal /path/to/db
litestream wal s3://mybucket/db
```

---

## Running as a Service

### systemd

The `.deb` and `.rpm` packages install a systemd service automatically.

```bash
# Enable and start
sudo systemctl enable litestream
sudo systemctl start litestream

# Check status
sudo systemctl status litestream

# View logs
sudo journalctl -u litestream -f

# Restart after config changes
sudo systemctl restart litestream
```

Config file: `/etc/litestream.yml`

Example `/etc/litestream.yml`:

```yaml
access-key-id: ${LITESTREAM_ACCESS_KEY_ID}
secret-access-key: ${LITESTREAM_SECRET_ACCESS_KEY}

dbs:
  - path: /var/lib/myapp/app.db
    replica:
      url: s3://mybucket/app
```

Set credentials in systemd environment or `/etc/default/litestream`.

### Wrapping Your Application with -exec

The `-exec` flag makes Litestream a simple process supervisor. Litestream starts
replication, then launches your application as a child process. When the child exits,
Litestream performs a final sync and exits.

```bash
litestream replicate -exec "myapp serve --port 8080"
```

In config file:

```yaml
exec: "myapp serve --port 8080"

dbs:
  - path: /var/lib/app.db
    replica:
      url: s3://mybucket/app
```

This is the recommended pattern for single-container deployments (Fly.io, Railway, etc.).

---

## Restore

### Basic Restore

```bash
# Restore using config file (finds replica from config)
litestream restore /var/lib/app.db

# Restore directly from replica URL
litestream restore -o /var/lib/app.db s3://mybucket/app
```

### Point-in-Time Restore

```bash
litestream restore -timestamp "2024-06-15T14:30:00Z" -o /tmp/restored.db /var/lib/app.db
```

Timestamp must be in RFC 3339 format. Litestream finds the snapshot before the
requested time and replays WAL files up to that point.

### Restore to Different Path

```bash
litestream restore -o /tmp/restored.db s3://mybucket/app
```

### Automated Restore (Init Scripts / Docker)

```bash
# Restore if no local database exists, and if a replica is available
litestream restore -if-db-not-exists -if-replica-exists -o /var/lib/app.db s3://mybucket/app
```

- `-if-db-not-exists` -- skip if `/var/lib/app.db` already exists
- `-if-replica-exists` -- skip if no backup found (first deployment)

This combination is safe for automated startup: it does the right thing whether
it's a fresh deploy, a restart with existing data, or a recovery from failure.

### Parallelism

```bash
litestream restore -parallelism 16 -o /var/lib/app.db s3://mybucket/app
```

Downloads up to 16 WAL files concurrently (default: 8).

---

## Docker Patterns

### Pattern 1: Single Container with -exec

Recommended for platforms that run a single container (Fly.io, Railway).

**Dockerfile:**

```dockerfile
FROM golang:1.22 AS builder
WORKDIR /app
COPY . .
RUN go build -o /myapp .

FROM debian:bookworm-slim

# Install Litestream
ADD https://github.com/benbjohnson/litestream/releases/download/v0.5.6/litestream-0.5.6-linux-amd64.tar.gz /tmp/litestream.tar.gz
RUN tar -xzf /tmp/litestream.tar.gz -C /usr/local/bin/ && rm /tmp/litestream.tar.gz

COPY --from=builder /myapp /usr/local/bin/myapp
COPY litestream.yml /etc/litestream.yml
COPY scripts/run.sh /scripts/run.sh

RUN chmod +x /scripts/run.sh
CMD ["/scripts/run.sh"]
```

**scripts/run.sh (entrypoint):**

```bash
#!/bin/bash
set -e

# Restore database from replica if it doesn't exist locally
litestream restore -if-db-not-exists -if-replica-exists -o /var/lib/app.db "${REPLICA_URL}"

# Start Litestream replication, launching the app as a subprocess
exec litestream replicate -exec "myapp serve"
```

**litestream.yml:**

```yaml
dbs:
  - path: /var/lib/app.db
    replica:
      url: ${REPLICA_URL}
```

**Run:**

```bash
docker run \
  -e REPLICA_URL=s3://mybucket/app \
  -e LITESTREAM_ACCESS_KEY_ID=AKIA... \
  -e LITESTREAM_SECRET_ACCESS_KEY=... \
  -p 8080:8080 \
  myapp
```

### Pattern 2: Docker Compose Sidecar

```yaml
version: "3.8"

services:
  app:
    image: myapp:latest
    volumes:
      - data:/var/lib/data
    depends_on:
      litestream:
        condition: service_healthy

  litestream:
    image: litestream/litestream:latest
    volumes:
      - data:/var/lib/data
      - ./litestream.yml:/etc/litestream.yml
    environment:
      - LITESTREAM_ACCESS_KEY_ID
      - LITESTREAM_SECRET_ACCESS_KEY
    command: ["replicate"]
    healthcheck:
      test: ["CMD", "litestream", "restore", "-if-db-not-exists", "-if-replica-exists", "-o", "/var/lib/data/app.db", "s3://mybucket/app"]
      interval: 10s
      timeout: 30s
      retries: 3
      start_period: 10s

volumes:
  data:
```

### Volume Constraints

- Host and container must use the same OS (SQLite locking is OS-specific)
- Local Docker volumes and bind mounts: supported
- Block storage devices: supported
- NFS / SMB network filesystems: NOT supported (broken file locking)
- Docker Desktop on macOS/Windows: NOT supported for production

---

## Kubernetes

### Architecture

Use a **StatefulSet** with `replicas: 1` (Litestream requires single-writer):

1. **Init container** -- restores database from replica before app starts
2. **App container** -- your application
3. **Sidecar container** -- runs `litestream replicate` continuously
4. **Shared PVC** -- database file shared between all containers

### Step 1: Create Secret

```bash
kubectl create secret generic litestream \
  --from-literal=LITESTREAM_ACCESS_KEY_ID="AKIA..." \
  --from-literal=LITESTREAM_SECRET_ACCESS_KEY="..."
```

### Step 2: Create ConfigMap

```yaml
# litestream.yml
dbs:
  - path: /var/lib/myapp/db
    replica:
      url: s3://mybucket/db
```

```bash
kubectl create configmap litestream --from-file=litestream.yml
```

### Step 3: StatefulSet

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: myapp
spec:
  replicas: 1           # MUST be 1 -- Litestream is single-writer
  selector:
    matchLabels:
      app: myapp
  template:
    metadata:
      labels:
        app: myapp
    spec:
      # --- Init container: restore from backup ---
      initContainers:
        - name: init-litestream
          image: litestream/litestream:latest
          args:
            - restore
            - -if-db-not-exists
            - -if-replica-exists
            - /var/lib/myapp/db
          volumeMounts:
            - name: data
              mountPath: /var/lib/myapp
            - name: litestream-config
              mountPath: /etc/litestream.yml
              subPath: litestream.yml
          envFrom:
            - secretRef:
                name: litestream

      containers:
        # --- Application container ---
        - name: myapp
          image: myapp:latest
          volumeMounts:
            - name: data
              mountPath: /var/lib/myapp

        # --- Litestream sidecar ---
        - name: litestream
          image: litestream/litestream:latest
          args:
            - replicate
          ports:
            - name: metrics
              containerPort: 9090
          volumeMounts:
            - name: data
              mountPath: /var/lib/myapp
            - name: litestream-config
              mountPath: /etc/litestream.yml
              subPath: litestream.yml
          envFrom:
            - secretRef:
                name: litestream

      volumes:
        - name: litestream-config
          configMap:
            name: litestream

  # --- PVC for database storage ---
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```

### Alternative: emptyDir (No PVC)

If you trust Litestream to always restore from S3, you can use `emptyDir` instead of a
PVC. The init container restores the database on every pod start. This avoids PVC
management but means every pod start pays the restore cost.

```yaml
volumes:
  - name: data
    emptyDir: {}
```

### Verification

Delete the pod and watch init container logs to confirm restore from S3:

```bash
kubectl delete pod myapp-0
kubectl logs myapp-0 -c init-litestream
```

---

## WAL Mode Requirements

Litestream **only works with SQLite WAL mode**. It will not function with the default
rollback journal mode.

### Enable WAL Mode

```sql
PRAGMA journal_mode=wal;
```

This is a persistent setting -- once enabled, it remains across connections.

### Required PRAGMAs

Set these on every new database connection:

```sql
-- Required: prevent SQLITE_BUSY errors during Litestream checkpoints
PRAGMA busy_timeout = 5000;

-- Recommended: safe in WAL mode and reduces fsync overhead
PRAGMA synchronous = NORMAL;

-- Recommended: enable referential integrity
PRAGMA foreign_keys = ON;
```

### Disable Autocheckpointing (High Write Load)

Under high write throughput (tens of thousands of transactions/second), SQLite's
autocheckpointing can race with Litestream's checkpoints, causing WAL continuity breaks
and forcing expensive full snapshots.

```sql
PRAGMA wal_autocheckpoint = 0;
```

Let Litestream handle all checkpointing via its `checkpoint-interval` setting.

### Connection String Examples

**Rust (rusqlite):**

```rust
let conn = Connection::open("/var/lib/app.db")?;
conn.execute_batch("
    PRAGMA journal_mode=wal;
    PRAGMA busy_timeout=5000;
    PRAGMA synchronous=NORMAL;
    PRAGMA foreign_keys=ON;
")?;
```

**Go (modernc.org/sqlite or mattn/go-sqlite3):**

```go
db, err := sql.Open("sqlite3", "/var/lib/app.db?_journal_mode=wal&_busy_timeout=5000&_synchronous=normal&_foreign_keys=on")
```

**Python (sqlite3):**

```python
conn = sqlite3.connect("/var/lib/app.db")
conn.execute("PRAGMA journal_mode=wal")
conn.execute("PRAGMA busy_timeout=5000")
conn.execute("PRAGMA synchronous=NORMAL")
```

---

## Monitoring

### Prometheus Metrics

Enable the metrics endpoint in config:

```yaml
addr: ":9090"
```

Scrape `http://localhost:9090/metrics` with Prometheus.

In Kubernetes, expose the port on the sidecar container:

```yaml
ports:
  - name: metrics
    containerPort: 9090
```

Add Prometheus annotations:

```yaml
metadata:
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "9090"
```

### Health Checks

Litestream does not expose a dedicated health endpoint. Strategies:

1. **Process check** -- verify the Litestream process is running
2. **Metrics endpoint** -- if `addr` is set, HTTP 200 on `/metrics` indicates alive
3. **File age check** -- monitor the shadow WAL directory timestamp
4. **VFS lag function** (proposed) -- `litestream_lag()` SQL function returning seconds
   since last successful poll (in development)

### Logging

```yaml
logging:
  level: info          # debug for troubleshooting
  type: json           # json for structured log ingestion
  stderr: false
```

---

## Limitations

### Single-Writer Only

Litestream requires exactly one process writing to the database. Multiple concurrent
writers will corrupt the replica. This is a fundamental constraint of SQLite itself
when used with Litestream's WAL-based replication.

### No Multi-Node Write Replication

Litestream does not support multi-primary replication. It is a disaster recovery tool,
not a distributed database. All writes must go to one node.

### Read Replicas are Read-Only

The VFS read replica feature (v0.5.0+) allows read-only access to replicated data.
Writing to a read replica will corrupt it. Applications must enforce read-only mode:

```sql
-- SQLite CLI
sqlite3 -readonly /path/to/replica.db

-- Connection URL parameter
sqlite:///path/to/replica.db?mode=ro
```

### Single Replica Per Database (v0.5.0+)

Each database supports only one replica destination. The multiple-replica feature
from v0.3.x is deprecated.

### Asynchronous Replication Lag

Default sync interval is 1 second. Data written but not yet replicated can be lost
during catastrophic failure. Graceful shutdown triggers a final sync.

### No Network Filesystems

SQLite (and therefore Litestream) requires proper file locking. NFS, SMB, and similar
network filesystems do not provide this reliably. Use local/block storage only.

### Checkpointing Under High Load

Under very high write throughput, SQLite may autocheckpoint between Litestream
checkpoints, causing WAL breaks. Mitigate with `PRAGMA wal_autocheckpoint = 0`.

### Database Deletion

Deleting a database without cleaning up associated files causes issues. To properly
delete and recreate:

```bash
rm /var/lib/app.db /var/lib/app.db-wal /var/lib/app.db-shm
rm -rf /var/lib/.app.db-litestream
# Restart Litestream
```

---

## Integration with Rust

Litestream is a standalone Go binary with no Rust library. Integration is done at the
process level.

### Approach 1: Litestream Wraps Rust App (Recommended)

Use the `-exec` flag to have Litestream manage your Rust binary as a child process.
Litestream handles replication lifecycle; your app just uses SQLite normally.

```bash
litestream replicate -exec "/usr/local/bin/myapp" \
  /var/lib/app.db s3://mybucket/app
```

Or with a config file:

```yaml
exec: "/usr/local/bin/myapp --port 8080"

dbs:
  - path: /var/lib/app.db
    replica:
      url: s3://mybucket/app
```

```bash
litestream replicate -config /etc/litestream.yml
```

Lifecycle:
1. Litestream starts and begins replicating
2. Litestream launches your Rust binary as a subprocess
3. Your Rust app reads/writes SQLite normally
4. When your app exits, Litestream does a final sync and exits

### Approach 2: Rust App Spawns Litestream

Launch Litestream as a background child process from your Rust application:

```rust
use std::process::Command;

fn start_litestream() -> std::io::Result<std::process::Child> {
    Command::new("litestream")
        .args(["replicate", "-config", "/etc/litestream.yml"])
        .spawn()
}

fn main() -> anyhow::Result<()> {
    let mut litestream = start_litestream()?;

    // Your application logic here
    // ...

    // On shutdown, signal Litestream to stop
    litestream.kill()?;
    litestream.wait()?;
    Ok(())
}
```

For graceful shutdown, send SIGTERM instead of kill:

```rust
use nix::sys::signal::{self, Signal};
use nix::unistd::Pid;

// Graceful shutdown -- Litestream will do a final sync
signal::kill(Pid::from_raw(litestream.id() as i32), Signal::SIGTERM)?;
litestream.wait()?;
```

### Approach 3: Sidecar Process

Run Litestream as a completely separate process (systemd service, Docker sidecar,
Kubernetes sidecar). Your Rust app does not need to know about Litestream at all.
This is the simplest approach and works well for server deployments.

### Docker with Rust

```dockerfile
FROM rust:1.82 AS builder
WORKDIR /app
COPY . .
RUN cargo build --release

FROM debian:bookworm-slim

# Install Litestream
ADD https://github.com/benbjohnson/litestream/releases/download/v0.5.6/litestream-0.5.6-linux-amd64.tar.gz /tmp/litestream.tar.gz
RUN tar -xzf /tmp/litestream.tar.gz -C /usr/local/bin/ && rm /tmp/litestream.tar.gz

COPY --from=builder /app/target/release/myapp /usr/local/bin/myapp
COPY litestream.yml /etc/litestream.yml
COPY scripts/run.sh /scripts/run.sh
RUN chmod +x /scripts/run.sh

CMD ["/scripts/run.sh"]
```

**scripts/run.sh:**

```bash
#!/bin/bash
set -e
litestream restore -if-db-not-exists -if-replica-exists -o /var/lib/app.db "${REPLICA_URL}"
exec litestream replicate -exec "myapp"
```

### Rust SQLite Setup

Regardless of integration approach, configure SQLite in your Rust app:

```rust
use rusqlite::Connection;

fn open_db(path: &str) -> rusqlite::Result<Connection> {
    let conn = Connection::open(path)?;
    conn.execute_batch("
        PRAGMA journal_mode = wal;
        PRAGMA busy_timeout = 5000;
        PRAGMA synchronous = NORMAL;
        PRAGMA foreign_keys = ON;
    ")?;
    Ok(conn)
}
```

### Alternative: Verneuil (Rust-Native)

[Verneuil](https://lib.rs/crates/verneuil) is a Rust-native SQLite VFS for async
replication to S3. However, it only supports rollback journal mode (not WAL), and is
designed for read replication rather than disaster recovery. It is not a drop-in
replacement for Litestream.
