# Claude Code Telemetry Reference

Covers: telemetry services, data collection, privacy controls, environment variables, network endpoints.

---

## Quick Reference: Disable Everything

```bash
# Master kill switch — disables Statsig, Sentry, bug reports, surveys, auto-updater
CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1
```

Add to `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1"
  }
}
```

---

## Telemetry Services

### Statsig (Usage Analytics)
- **Purpose**: Operational metrics, A/B testing, feature flags
- **Data**: Latency, reliability, usage patterns — NOT code or file paths
- **Local cache**: `~/.claude/statsig/` (evaluations, session IDs, stable IDs)
- **Endpoint**: `statsig.anthropic.com` (`/v1/initialize`, `/v1/rgstr`)
- **Default**: ON (Claude API), OFF (Bedrock/Vertex/Foundry)
- **Opt-out**: `DISABLE_TELEMETRY=1`
- **Note**: Statsig was acquired by OpenAI in 2025

### Sentry (Error Logging)
- **Purpose**: Crash reports, error stack traces
- **Endpoint**: `*.sentry.io`
- **Default**: ON (Claude API), OFF (Bedrock/Vertex/Foundry)
- **Opt-out**: `DISABLE_ERROR_REPORTING=1`

### OpenTelemetry (Advanced Monitoring)
- **Purpose**: Detailed metrics/events for enterprise observability
- **Default**: DISABLED (opt-in only)
- **Opt-in**: `CLAUDE_CODE_ENABLE_TELEMETRY=1`
- **Backends**: Datadog, Honeycomb, Grafana, any OTLP-compatible endpoint
- **WARNING**: If `OTEL_LOG_USER_PROMPTS=1` is set, actual prompt content is logged

### Perfetto (Performance Tracing)
- **Purpose**: API call timings (TTFT, TTLT, token throughput), tool execution spans
- **Controlled by**: `CLAUDE_CODE_PERFETTO_TRACE` env var
- **Default**: DISABLED

---

## Environment Variables

### Disable Controls

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1` | Master switch — disables all below |
| `DISABLE_TELEMETRY=1` | Disables Statsig analytics |
| `DISABLE_ERROR_REPORTING=1` | Disables Sentry error logging |
| `DISABLE_BUG_COMMAND=1` | Disables `/bug` report command |
| `CLAUDE_CODE_DISABLE_FEEDBACK_SURVEY=1` | Disables session quality surveys |

### Enable Controls (opt-in)

| Variable | Effect |
|----------|--------|
| `CLAUDE_CODE_ENABLE_TELEMETRY=1` | Enables OpenTelemetry export |
| `CLAUDE_CODE_ENHANCED_TELEMETRY_BETA=1` | Enhanced telemetry (beta) |
| `OTEL_LOG_USER_PROMPTS=1` | Logs prompt content to OTLP backends |
| `OTEL_LOG_TOOL_DETAILS=1` | Logs MCP server/tool and skill names |
| `CLAUDE_CODE_PERFETTO_TRACE=1` | Enables Perfetto performance tracing |

### OTLP Configuration

| Variable | Effect |
|----------|--------|
| `OTEL_METRICS_EXPORTER=otlp` | Exporter type: `otlp`, `prometheus`, `console` |
| `OTEL_LOGS_EXPORTER=otlp` | Exporter type: `otlp`, `console` |
| `OTEL_EXPORTER_OTLP_PROTOCOL` | `grpc` (port 4317), `http/protobuf` or `http/json` (port 4318) |
| `OTEL_EXPORTER_OTLP_ENDPOINT` | Base OTLP endpoint (applies to all signals) |
| `OTEL_EXPORTER_OTLP_METRICS_ENDPOINT` | Signal-specific override for metrics |
| `OTEL_EXPORTER_OTLP_LOGS_ENDPOINT` | Signal-specific override for logs |
| `OTEL_EXPORTER_OTLP_HEADERS` | Auth headers: `Authorization=Bearer token` |
| `OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE` | `cumulative` (required for VictoriaMetrics) or `delta` |
| `OTEL_METRIC_EXPORT_INTERVAL` | Export interval in ms (default: 60000) |
| `OTEL_LOGS_EXPORT_INTERVAL` | Log export interval in ms (default: 5000) |
| `OTEL_RESOURCE_ATTRIBUTES` | Custom attributes: `department=eng,team=platform` |

### Cardinality Controls

| Variable | Default | Effect |
|----------|---------|--------|
| `OTEL_METRICS_INCLUDE_SESSION_ID` | `true` | Include session ID label |
| `OTEL_METRICS_INCLUDE_VERSION` | `false` | Include app version label |
| `OTEL_METRICS_INCLUDE_ACCOUNT_UUID` | `true` | Include account UUID label |

**Note**: `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC` does NOT affect OTel. The two systems are independent.

---

## Data Collection

### Conversation Data (All API Requests)
- All prompts + model outputs sent to `api.anthropic.com`
- **Encryption**: TLS in transit, NOT encrypted at rest
- **Retention**:
  - Consumer (Free/Pro/Max) with training enabled: **5 years**
  - Consumer with training disabled: **30 days**
  - Commercial (Team/Enterprise): **30 days**
  - Bedrock/Vertex/Foundry: per provider policy (zero-retention possible)
- **Training opt-out**: [claude.ai/settings/data-privacy-controls](https://claude.ai/settings/data-privacy-controls)

### Beta Product Feedback Clause
Claude Code is a Beta product. Per Anthropic's Commercial Terms:
> Code acceptance/rejection decisions and associated conversations constitute Feedback
> and may be used to improve products, including training models.

### Local Data Files
- `~/.claude/statsig/` — feature flag evaluations, session/stable IDs
- `~/.claude/stats-cache.json` — daily message counts, tool calls, token usage, session metadata

### Bug Reports (`/bug` command)
- Sends **full conversation history including code**
- Retained for 5 years
- Optionally creates public GitHub issue
- Disable: `DISABLE_BUG_COMMAND=1`

---

## Network Endpoints

### Required
| Endpoint | Purpose |
|----------|---------|
| `api.anthropic.com` | Claude API |
| `registry.npmjs.org` | Package updates |

### Telemetry (disabled by `CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC=1`)
| Endpoint | Purpose |
|----------|---------|
| `statsig.anthropic.com` | Analytics |
| `*.sentry.io` | Error reporting |

### Reported but Undocumented
| Endpoint | Purpose | Issue |
|----------|---------|-------|
| `datadoghq.com` | Unknown — continuous calls every ~3s | [#16433](https://github.com/anthropics/claude-code/issues/16433) |

---

## Defaults by API Provider

| Feature | Claude API | Bedrock/Vertex/Foundry |
|---------|-----------|------------------------|
| Statsig | ON | OFF |
| Sentry | ON | OFF |
| `/bug` reporting | ON | OFF |
| Session surveys | ON | OFF |

---

## Known Issues

| Issue | Summary |
|-------|---------|
| [#16433](https://github.com/anthropics/claude-code/issues/16433) | Undisclosed continuous Datadog connections every ~3s |
| [#10494](https://github.com/anthropics/claude-code/issues/10494) | `DISABLE_TELEMETRY` flag ignored — still connects to Google infra |
| [#5508](https://github.com/anthropics/claude-code/issues/5508) | OpenTelemetry can't be disabled on Windows, leaks email/user ID |
| [#11057](https://github.com/anthropics/claude-code/issues/11057) | No telemetry notification on startup |
| [#7151](https://github.com/anthropics/claude-code/issues/7151) | Statsig owned by OpenAI — privacy implications |

---

## OpenTelemetry Metrics (when enabled)

| Metric | Description |
|--------|-------------|
| `claude_code.session.count` | Session starts |
| `claude_code.lines_of_code.count` | Code additions/removals |
| `claude_code.pull_request.count` | PRs created |
| `claude_code.commit.count` | Commits created |
| `claude_code.cost.usage` | Estimated USD cost |
| `claude_code.token.usage` | Token counts by type (input/output/cacheRead/cacheCreation) |
| `claude_code.code_edit_tool.decision` | Edit tool accept/reject decisions |
| `claude_code.active_time.total` | Active usage time |

## OpenTelemetry Events (logs signal)

| Event | Description |
|-------|-------------|
| `claude_code.user_prompt` | User prompt submitted |
| `claude_code.api_request` | API request (includes model, tokens, cost, duration) |
| `claude_code.api_error` | API request failed |
| `claude_code.tool_result` | Tool execution completed |
| `claude_code.tool_decision` | Tool permission decision |

### Standard Attributes on All Signals

`session.id`, `app.version`, `organization.id`, `user.account_uuid`, `user.email`, `terminal.type`, `os.type`, `host.arch`

---

## Local Setup: VictoriaMetrics + VictoriaLogs

Config in `~/.claude/settings.json`:
```json
{
  "env": {
    "CLAUDE_CODE_DISABLE_NONESSENTIAL_TRAFFIC": "1",
    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
    "OTEL_METRICS_EXPORTER": "otlp",
    "OTEL_LOGS_EXPORTER": "otlp",
    "OTEL_EXPORTER_OTLP_PROTOCOL": "http/protobuf",
    "OTEL_EXPORTER_OTLP_METRICS_ENDPOINT": "http://localhost:8428/opentelemetry/v1/metrics",
    "OTEL_EXPORTER_OTLP_LOGS_ENDPOINT": "http://localhost:9428/insert/opentelemetry/v1/logs",
    "OTEL_EXPORTER_OTLP_METRICS_TEMPORALITY_PREFERENCE": "cumulative"
  }
}
```

Query examples:
```bash
# List all Claude Code metric series
curl 'http://localhost:8428/api/v1/series' --data-urlencode 'match[]={__name__=~"claude_code.*"}'

# Query recent cost
curl 'http://localhost:8428/api/v1/query' --data-urlencode 'query=claude_code.cost.usage'

# Query logs
curl 'http://localhost:9428/select/logsql/query' --data-urlencode 'query=service.name:claude-code'
```

---

## Max Plan Usage API (OAuth)

Undocumented endpoint returns real-time usage for Max subscribers:

```bash
ACCESS_TOKEN=$(python3 -c "import json; print(json.load(open('$HOME/.claude/.credentials.json'))['claudeAiOauth']['accessToken'])")

curl -s "https://api.anthropic.com/api/oauth/usage" \
  -H "Authorization: Bearer $ACCESS_TOKEN" \
  -H "anthropic-beta: oauth-2025-04-20" \
  -H "User-Agent: claude-code/2.1.37"
```

Response:
```json
{
  "five_hour":    { "utilization": 100.0, "resets_at": "2026-02-14T08:00:01+00:00" },
  "seven_day":    { "utilization": 13.0,  "resets_at": "2026-02-20T16:00:00+00:00" },
  "seven_day_sonnet": { "utilization": 2.0, "resets_at": "..." },
  "seven_day_opus": null,
  "extra_usage":  { "is_enabled": true, "monthly_limit": 5000, "used_credits": 556.0, "utilization": 11.12 }
}
```

Credentials: `~/.claude/.credentials.json` → `claudeAiOauth.accessToken` (starts with `sk-ant-oat01-`)

---

## tmux Status Bar Integration

`~/.tmux/claude-usage.sh` — fetches OAuth usage every 90s, caches, flashes on change.

Display: `████████ 100% ↻1h24m | 7d 13% | $556/5000`

- Gauge color: green (<50%) → yellow (50%) → orange (70%) → red (90%)
- `↻` countdown to 5-hour reset
- 7-day utilization percentage
- Extra usage credits consumed/limit

---

## Sources

- [Data usage — Claude Code Docs](https://code.claude.com/docs/en/data-usage)
- [Monitoring — Claude Code Docs](https://code.claude.com/docs/en/monitoring-usage)
- [Anthropic Privacy Center](https://privacy.claude.com/en/)
- [Anthropic Commercial Terms](https://www.anthropic.com/legal/commercial-terms)
