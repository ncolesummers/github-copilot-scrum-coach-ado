# ADR-0002: Use GitHub Copilot SDK Over Direct LLM API

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

SprintPilot requires an agentic LLM runtime capable of multi-turn conversations, tool/function call loops, and MCP server integration. The choices are:

1. **GitHub Copilot SDK** (`github-copilot-sdk`, Python) — programmatic access to the Copilot CLI's agentic engine via JSON-RPC over stdio. GitHub Copilot is already approved and deployed across OIT.
2. **Direct LLM API** (OpenAI, Azure OpenAI, Anthropic) — call an LLM API directly, implement the tool loop ourselves, wire MCP servers manually.
3. **LangChain / LangGraph** — open-source orchestration framework with agent abstractions and tool routing.
4. **Semantic Kernel** — Microsoft's agent framework with Azure-first integrations.

Key constraints:

- GitHub Copilot is the only AI service currently approved for use in OIT's environment.
- The Azure DevOps MCP Server must be connected to the agent; the Copilot SDK supports MCP servers natively as first-class configuration.
- The team wants to avoid building and maintaining a custom tool loop and MCP protocol implementation.
- Using the Copilot SDK means SprintPilot counts against OIT's existing Copilot premium request quota rather than requiring a separate API key or billing account.

## Decision

Use the **GitHub Copilot SDK** (`github-copilot-sdk`, Python) as the agent runtime. The SDK communicates with the Copilot CLI via JSON-RPC over stdio. The Copilot CLI manages the LLM request loop, tool execution, and MCP server lifecycle.

Direct LLM APIs (BYOK via OpenAI, Azure AI Foundry, Anthropic) remain available as a scaling path if the Copilot premium request quota is exhausted — the Copilot SDK supports BYOK without code changes to the agent layer.

## Consequences

**Positive:**
- No custom tool loop or MCP protocol implementation required.
- ADO MCP server integration is a single configuration block at session creation.
- Uses existing Copilot subscription — no new procurement or billing approvals needed.
- BYOK scaling path available without architectural changes.

**Negative / Risks:**
- The Copilot SDK is in Technical Preview; breaking changes are possible. Mitigated by pinning SDK version and abstracting session management behind an internal interface (`core/copilot.py`).
- The Copilot CLI must be available in the container image at runtime. Dockerfile must install it; health checks must verify it at startup.
- JSON-RPC over stdio is opaque compared to direct HTTP calls; debugging requires structured logging of all CLI events (see ADR-0008).
- Premium request quota is shared with other OIT Copilot users; quota exhaustion could degrade SprintPilot before other users notice.
