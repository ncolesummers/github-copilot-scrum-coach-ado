# ADR-0008: OpenTelemetry + Collector + Jaeger for Observability

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-21 |
| **Author** | OIT Application Development |

## Context

SprintPilot's runtime spans multiple processes connected by stdio:

```
FastAPI worker
  └─ CopilotClient (pool)
       └─ Copilot CLI (JSON-RPC / stdio)
            └─ ADO MCP Server (stdio child)
```

This process nesting makes traditional single-service observability insufficient. When a user's sprint report request takes 25 seconds, we need to know whether the time was spent in FastAPI request handling, pool checkout queuing, the Copilot CLI's tool planning loop, MCP tool invocations against Azure DevOps, or python-pptx document generation. Without distributed tracing, the answer requires manual log correlation across process boundaries.

The PRD's non-functional requirements (§8) specify:
- Structured logs for all MCP tool invocations and agent events
- Metrics: token usage, tool call frequency, session duration, error rates
- Time to first token < 2 seconds; sprint report generation < 30 seconds for ≤ 50 work items

These targets are unverifiable without instrumentation that spans the full request path.

Options considered:

| Option | Assessment |
|---|---|
| **OpenTelemetry + OTel Collector + Jaeger** | Vendor-neutral; FastAPI has first-class OTel support; spans can be propagated into CLI subprocess events via context injection; Jaeger provides full trace visualization; Collector decouples instrumentation from backend |
| **Datadog / New Relic / Dynatrace** | Full-featured APM; requires licensing and external data egress; not currently approved in OIT's environment |
| **Azure Monitor + Application Insights** | Native Azure integration; good fit for long term; heavier setup; requires Azure SDK instrumentation alongside OTel; can be added as a Collector exporter later |
| **Structured logs only (no tracing)** | Easier to implement; no cross-process trace correlation; makes latency attribution across the stdio chain very difficult |
| **Prometheus + Grafana (metrics only)** | Good for aggregate metrics; no distributed tracing; doesn't address the cross-process visibility gap |

OpenTelemetry is the clear choice for a Python/FastAPI service with multi-process boundaries. The OTel Collector pattern decouples what the application emits from where it's sent, allowing the backend to be swapped (e.g., to Azure Monitor) without touching application code.

## Decision

Instrument SprintPilot with **OpenTelemetry** (traces, metrics, logs), collect signals via the **OpenTelemetry Collector**, and visualize traces in **Jaeger**. All three run as services in Docker Compose alongside the application.

### Instrumentation scope

| Component | Signal | Key spans / metrics |
|---|---|---|
| FastAPI | Traces, Metrics | All HTTP/WebSocket requests; `http.server.duration`; WebSocket connection lifecycle |
| WebSocket chat handler | Traces | `chat.turn` span per message; attributes: `session_id`, `workflow_type`, `user_id` |
| CopilotClient pool | Traces, Metrics | `pool.checkout` span with wait time; `pool.size`, `pool.active`, `pool.queue_depth` gauges |
| Copilot CLI interaction | Traces | `copilot.session.turn` span; child spans per streaming event; `copilot.tokens.input`, `copilot.tokens.output` counters |
| MCP tool invocations | Traces, Metrics | `mcp.tool.call` span per ADO tool call; attributes: `tool.name`, `ado.project`; `mcp.tool.duration` histogram |
| Document generation | Traces | `report.generate` span; attributes: `report.type`, `report.size_bytes` |
| PostgreSQL | Traces | Auto-instrumented via `opentelemetry-instrumentation-sqlalchemy`; all DB queries as child spans |

### Libraries

```
opentelemetry-sdk
opentelemetry-api
opentelemetry-exporter-otlp-proto-grpc   # OTLP gRPC export to Collector
opentelemetry-instrumentation-fastapi
opentelemetry-instrumentation-sqlalchemy
opentelemetry-instrumentation-logging    # Injects trace_id into structured log records
```

### Docker Compose additions

```yaml
otel-collector:
  image: otel/opentelemetry-collector-contrib:latest
  volumes:
    - ./otel-collector-config.yaml:/etc/otel-collector-config.yaml
  command: ["--config=/etc/otel-collector-config.yaml"]
  ports:
    - "4317:4317"   # OTLP gRPC receiver
    - "4318:4318"   # OTLP HTTP receiver
    - "8888:8888"   # Collector self-metrics

jaeger:
  image: jaegertracing/all-in-one:latest
  ports:
    - "16686:16686"  # Jaeger UI
    - "4317"         # OTLP gRPC (internal, received from Collector)
  environment:
    - COLLECTOR_OTLP_ENABLED=true
```

Collector pipeline: OTLP receiver → batch processor → Jaeger exporter (OTLP). A Prometheus exporter is also configured on the Collector for metrics scraping.

### Trace propagation across the stdio boundary

The Copilot CLI and ADO MCP server are external processes and do not natively carry OTel context. Propagation is handled by:

1. Injecting the current `trace_id` and `span_id` as attributes on the `copilot.session.turn` span at the FastAPI handler boundary.
2. Structuring all CLI event logs to include `trace_id` via `opentelemetry-instrumentation-logging`.
3. Emitting manual child spans for each MCP tool call result received from the CLI's event stream.

This gives partial propagation: the trace is continuous from FastAPI through the pool and CLI interaction spans, with MCP tool call timing captured as child spans. True end-to-end W3C context propagation into the CLI process is not possible without SDK changes and is deferred.

### Development vs. production

- **Development (Docker Compose):** OTel Collector + Jaeger `all-in-one` included by default. Jaeger UI at `http://localhost:16686`.
- **Production (Azure Container Apps):** Collector configured to export to Azure Monitor (Application Insights) via the `azuremonitor` exporter in addition to or instead of Jaeger. No code changes required — only Collector config.

## Consequences

**Positive:**
- Full latency attribution across the FastAPI → pool → CLI → MCP chain.
- Performance targets from §8 of the PRD are now measurable and alertable.
- `trace_id` injected into all structured logs enables fast log-to-trace correlation.
- Collector decoupling means the observability backend (Jaeger today, Azure Monitor tomorrow) can change without touching application code.
- Token usage metrics (`copilot.tokens.input/output`) directly address the PRD's quota exhaustion risk.

**Negative / Risks:**
- OTel SDK adds ~20–30ms of instrumentation overhead per request at low cardinality; acceptable given the 2-second time-to-first-token target.
- Two additional containers (Collector + Jaeger) in Docker Compose increase local development resource usage and `docker compose up` startup time.
- Manual span creation for MCP tool events requires discipline — if developers don't instrument new tool interactions, they become blind spots. The `development` skill should document the instrumentation pattern.
- Jaeger `all-in-one` uses in-memory storage; traces are lost on container restart in development. This is intentional for local dev; production uses the Collector-to-Azure-Monitor pipeline with durable storage.
