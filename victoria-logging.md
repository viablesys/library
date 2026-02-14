# VictoriaLogs — Local Log Ingestion

Sending structured logs from scripts and hooks to the local VictoriaLogs instance.

## Endpoints

| Service | Port | Ingestion | Query |
|---------|------|-----------|-------|
| VictoriaLogs | 9428 | `POST /insert/jsonline` | `POST /select/logsql/query` |
| VictoriaMetrics | 8428 | `POST /opentelemetry/v1/metrics` | Grafana / PromQL |

**Note:** The OTLP log endpoint (`/insert/opentelemetry/v1/logs`) requires protobuf encoding. Use the native jsonline endpoint instead for simple scripts.

## Ingestion — Python (stdlib only)

```python
import json
import urllib.request

VLOGS = "http://localhost:9428/insert/jsonline"

def send_log(msg: str, level: str = "info", **fields):
    """Send a structured log entry. Extra kwargs become indexed fields."""
    record = {"_msg": msg, "level": level, **fields}
    try:
        req = urllib.request.Request(
            VLOGS,
            data=json.dumps(record).encode("utf-8") + b"\n",
            headers={"Content-Type": "application/stream+json"},
            method="POST",
        )
        urllib.request.urlopen(req, timeout=2)
    except Exception:
        pass  # never let logging failures cascade
```

Usage:
```python
send_log("observation stored", service="nmem-capture", obs_type="file_edit", session_id="abc-123")
send_log("redaction applied", level="warn", service="nmem-capture", pattern="aws_key")
send_log(traceback.format_exc(), level="error", service="nmem-capture", hook="PostToolUse")
```

### Batch ingestion

Multiple records in one request — one JSON object per line:

```python
def send_logs(records: list[dict]):
    body = "\n".join(json.dumps(r) for r in records) + "\n"
    try:
        req = urllib.request.Request(
            VLOGS,
            data=body.encode("utf-8"),
            headers={"Content-Type": "application/stream+json"},
            method="POST",
        )
        urllib.request.urlopen(req, timeout=2)
    except Exception:
        pass
```

### Rust (reqwest, non-blocking)

```rust
use reqwest::Client;
use serde_json::json;

async fn send_log(client: &Client, msg: &str, fields: serde_json::Value) {
    let mut record = fields;
    record["_msg"] = json!(msg);
    let body = format!("{}\n", record);
    let _ = client
        .post("http://localhost:9428/insert/jsonline")
        .header("Content-Type", "application/stream+json")
        .body(body)
        .timeout(std::time::Duration::from_secs(2))
        .send()
        .await;
}
```

## Querying — LogsQL

```bash
# All errors from a service
curl 'http://localhost:9428/select/logsql/query' \
  -d 'query=service:nmem-capture AND level:error'

# Recent entries (last hour)
curl 'http://localhost:9428/select/logsql/query' \
  -d 'query=service:nmem-capture' -d 'start=1h'

# Full-text search across messages
curl 'http://localhost:9428/select/logsql/query' \
  -d 'query=_msg:traceback AND service:nmem-capture'

# Count by level
curl 'http://localhost:9428/select/logsql/stats_query' \
  -d 'query=service:nmem-capture' -d 'stats=count() by (level)'
```

## Conventions

| Field | Purpose | Example |
|-------|---------|---------|
| `_msg` | Log message body (required by VictoriaLogs) | `"redaction applied"` |
| `level` | Severity: `debug`, `info`, `warn`, `error` | `"error"` |
| `service` | Source identifier | `"nmem-capture"`, `"nmem-record"` |
| `hook` | Claude Code hook event | `"PostToolUse"`, `"SessionStart"` |
| `tool_name` | Tool that triggered the event | `"Bash"`, `"Edit"` |
| `session_id` | Claude Code session UUID | `"f785470c-..."` |

All fields except `_msg` are optional. Extra fields are automatically indexed by VictoriaLogs.

## Hook error pattern

Hooks must never let logging failures block Claude Code. The pattern:

```python
if __name__ == "__main__":
    try:
        main()
    except Exception:
        send_log(traceback.format_exc(), level="error", service="my-hook")
```

This ensures: (1) the hook exits cleanly, (2) the error is captured for debugging, (3) Claude Code sees no hook error.
