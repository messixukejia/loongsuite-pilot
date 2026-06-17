# Trace Output

English | [简体中文](zh-CN/trace-output.md)

Trace output exports GenAI activity as OpenTelemetry traces. Use it when you want to analyze sessions, turns, model calls, and tool calls in a trace or APM backend.

Trace output is separate from log output. SLS, JSONL, and HTTP receive event records; OTLP trace output converts those records into trace spans.

## Generic OTLP Trace Output

```json
{
  "collectTrace": true,
  "otlpTrace": {
    "endpoint": "https://otel-collector.example.com/v1/traces",
    "headers": {
      "Authorization": "Bearer token"
    },
    "serviceName": "loongsuite-pilot",
    "resourceAttributes": {
      "deployment.environment": "prod"
    },
    "captureMessageContent": false,
    "debug": false,
    "turnIdleTimeoutMs": 0
  }
}
```

| Setting | Description |
|---------|-------------|
| `collectTrace` | Master switch for trace export. |
| `otlpTrace.endpoint` | OTLP HTTP trace endpoint. |
| `otlpTrace.headers` | Headers sent to the OTLP endpoint. |
| `otlpTrace.serviceName` | Service name attached to exported spans. |
| `otlpTrace.resourceAttributes` | Extra OpenTelemetry resource attributes. |
| `otlpTrace.captureMessageContent` | Whether trace export may include message content. |
| `otlpTrace.debug` | Enables local debug output for trace conversion. |
| `otlpTrace.turnIdleTimeoutMs` | Optional idle timeout for grouping turn-level trace data. |

Environment variables:

| Variable | Description |
|----------|-------------|
| `LOONGSUITE_PILOT_COLLECT_TRACE` | Set `false` or `0` to disable trace export. |
| `LOONGSUITE_PILOT_OTLP_ENDPOINT` | OTLP trace endpoint. |
| `LOONGSUITE_PILOT_OTLP_HEADERS` | JSON string for OTLP headers. |

## ARMS/CMS-Compatible Trace Output

Pilot also supports a CMS-style trace configuration:

```json
{
  "collectTrace": true,
  "cms": {
    "licenseKey": "your-license-key",
    "endpoint": "https://your-arms-endpoint/v1/traces",
    "workspace": "your-workspace",
    "debug": false
  }
}
```

Environment variables:

| Variable | Description |
|----------|-------------|
| `LOONGSUITE_PILOT_CMS_LICENSE_KEY` | CMS or ARMS license key. |
| `LOONGSUITE_PILOT_CMS_ENDPOINT` | CMS or ARMS trace endpoint. |
| `LOONGSUITE_PILOT_CMS_WORKSPACE` | Workspace header value. |

## Backend Examples

### Jaeger

[Jaeger](https://www.jaegertracing.io/) natively supports OTLP ingestion. Use the all-in-one image for a quick local setup:

```bash
docker run -d --name jaeger \
  -e COLLECTOR_OTLP_ENABLED=true \
  -p 16686:16686 \
  -p 4317:4317 \
  -p 4318:4318 \
  jaegertracing/all-in-one:latest
```

Configure Pilot:

```json
{
  "collectTrace": true,
  "otlpTrace": {
    "endpoint": "http://localhost:4318",
    "serviceName": "loongsuite-pilot"
  }
}
```

Or via environment variables:

```bash
export LOONGSUITE_PILOT_OTLP_ENDPOINT=http://localhost:4318
export LOONGSUITE_PILOT_COLLECT_TRACE=true
```

Open [http://localhost:16686](http://localhost:16686) and select the service name to view traces.

### Langfuse

[Langfuse](https://langfuse.com/) is an LLM observability platform with native OTLP ingestion. It provides LLM-specific views including cost tracking, token usage, and prompt/completion content.

**1. Start Langfuse (self-hosted):**

```bash
mkdir -p /tmp/langfuse && cd /tmp/langfuse
curl -sLO https://raw.githubusercontent.com/langfuse/langfuse/main/docker-compose.yml

cat > .env << 'EOF'
NEXTAUTH_SECRET=your-nextauth-secret
SALT=your-salt
ENCRYPTION_KEY=0000000000000000000000000000000000000000000000000000000000000000
LANGFUSE_INIT_ORG_NAME=MyOrg
LANGFUSE_INIT_PROJECT_NAME=loongsuite-pilot
LANGFUSE_INIT_PROJECT_PUBLIC_KEY=pk-lf-my-public-key
LANGFUSE_INIT_PROJECT_SECRET_KEY=sk-lf-my-secret-key
LANGFUSE_INIT_USER_EMAIL=admin@example.com
LANGFUSE_INIT_USER_NAME=admin
LANGFUSE_INIT_USER_PASSWORD=<your-password>
TELEMETRY_ENABLED=false
EOF

docker compose up -d
```

**2. Configure Pilot:**

Langfuse OTLP endpoint requires Basic authentication with `Base64(public_key:secret_key)`:

```bash
echo -n "<public_key>:<secret_key>" | base64
```

Add to `~/.loongsuite-pilot/config.json`:

```json
{
  "collectTrace": true,
  "otlpTrace": {
    "endpoint": "http://localhost:3000/api/public/otel",
    "headers": {
      "Authorization": "Basic <base64-encoded-credentials>"
    },
    "serviceName": "loongsuite-pilot",
    "captureMessageContent": true
  }
}
```

Open [http://localhost:3000](http://localhost:3000) and navigate to **Traces** to view agent sessions with model name, token usage, and cost details.

> **Note:** Langfuse uses HTTP for OTLP — gRPC (port 4317) is not supported. The endpoint path `/api/public/otel` is the OTLP base; Pilot auto-appends `/v1/traces`.

## Content Capture In Traces

Trace spans can carry sensitive content if message capture is enabled. For sensitive or team-managed setups, prefer:

```json
{
  "otlpTrace": {
    "captureMessageContent": false
  },
  "agents": {
    "claude-code": { "captureMessageContent": false },
    "codex": { "captureMessageContent": false },
    "cursor": { "captureMessageContent": false }
  }
}
```

Also enable [Data Masking](masking.md) when trace data may include secrets.

## Verify Trace Output

```bash
loongsuite-pilot restart
loongsuite-pilot status
```

If `otlpTrace.debug` or `cms.debug` is enabled, debug output is written under:

```text
~/.loongsuite-pilot/logs/otlp-debug/
```

Failed trace export data may be persisted under:

```text
~/.loongsuite-pilot/logs/otlp-failed/
```
