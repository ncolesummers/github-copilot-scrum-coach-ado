# ADR-0004: Pooled CopilotClient for Concurrency

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

Each `CopilotClient` instance spawns a Copilot CLI process (JSON-RPC over stdio), which in turn spawns an ADO MCP server as a child process. The process tree per client slot is:

```
FastAPI worker
  └─ CopilotClient
       └─ copilot CLI process (JSON-RPC / stdio)
            └─ npx @azure-devops/mcp (stdio child)
```

SprintPilot's usage profile is approximately 3 concurrent users on average, with peaks up to 30. Options for handling concurrency:

1. **One client per request** — spawn a new CLI + MCP process pair for every WebSocket connection. Simple but expensive; startup latency is high and process churn would be significant.
2. **One client per user session** — a dedicated process tree per authenticated user. Scales linearly with active users; 30 concurrent users = 30 CLI + MCP process pairs permanently running.
3. **Pooled clients (3–5 instances)** — a fixed pool of warm CopilotClient instances, checked out per request turn and returned when idle. Sessions persist in the Copilot SDK across messages; client binding is per-turn, not per-user.
4. **Single shared client** — one CLI process multiplexes all sessions. Simplest, but a single process failure takes down all users and throughput degrades under concurrent tool loops.

Key constraints:

- Each CLI + MCP server pair uses ~50–100MB of memory.
- Session state (conversation history, MCP server config) is maintained by the Copilot SDK and survives client reuse.
- FastAPI's async WebSocket handlers need non-blocking access to a client instance.

## Decision

Use a **pool of 3–5 `CopilotClient` instances**, configurable via the `COPILOT_POOL_SIZE` environment variable. FastAPI's WebSocket handler checks out a client from the pool for each turn, executes the session interaction, and returns the client. Sessions persist across turns within the SDK.

The pool implementation lives in `backend/app/core/copilot.py`. Pool behavior:

- **Checkout/return:** async context manager wraps each WebSocket turn.
- **Backpressure:** requests that arrive when all clients are busy queue with a configurable timeout (default: 30s). Users see a "processing…" indicator, not an error.
- **Health monitoring:** each client has a periodic health check; unhealthy instances (CLI process crashed or MCP server unresponsive) are recycled and replaced.
- **Startup:** pool is pre-warmed at application startup, not lazily initialized.

## Consequences

**Positive:**
- Low memory footprint: 5 pool slots ≈ 500MB overhead, independent of user count.
- Warm clients eliminate per-request startup latency.
- Pool size is tunable without code changes.
- Graceful degradation under load: queuing is preferable to errors.

**Negative / Risks:**
- Pool exhaustion at peak load (30 users, all submitting simultaneously) causes request queuing. If queue timeout is exceeded, users receive an error. Monitor pool utilization; increase `COPILOT_POOL_SIZE` if wait times are consistently elevated.
- Session state is stored in the Copilot SDK, not tied to a specific client instance — this is by design, but it means per-session state must be managed carefully to avoid context bleed between users on the same client.
- Client recycling (unhealthy instance replacement) briefly reduces available pool capacity. Health check intervals and recycling behavior must be tuned for the production environment.
