# ADR-0003: Use ADO MCP Server Over Custom Azure DevOps API Integration

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

SprintPilot needs to read from and write to Azure DevOps: querying work items and sprint data, creating and updating work items, managing test plans, searching, and publishing to the wiki. There are two integration approaches:

1. **Azure DevOps MCP Server** (`@azure-devops/mcp`, GA) — Microsoft's official MCP server that exposes ADO's REST API as MCP tools. The Copilot SDK connects to it natively via stdio. The agent calls `list_work_items`, `create_work_item`, etc. as first-class tools.
2. **Custom ADO API integration** — write a Python wrapper around the Azure DevOps REST API (or use the `azure-devops` Python client library). Expose ADO operations as custom tools registered with the Copilot agent.

Key factors:

- The ADO MCP Server reached GA shortly before SprintPilot development began, making it a viable production choice.
- The Copilot SDK supports MCP servers natively; wiring is a configuration block, not code.
- Custom ADO wrappers would require writing, testing, and maintaining hundreds of lines of API integration code covering authentication, pagination, error handling, and schema translation — all solved by the MCP server.
- The MCP server's tool coverage must be validated against each workflow's requirements during Phase 1 (see PRD §9, Phase 1 acceptance criteria).

## Decision

Use the **Azure DevOps MCP Server** (`@azure-devops/mcp`) for all ADO interactions. The MCP server is spawned as a stdio child process by the Copilot CLI, authenticated via a scoped service account PAT passed through the SDK's environment configuration.

Custom ADO API code is written only if the MCP server has gaps in tool coverage that cannot be worked around — and any such gaps are documented as follow-on ADRs.

## Consequences

**Positive:**
- Near-zero custom ADO integration code in the SprintPilot codebase.
- MCP tool coverage is maintained by Microsoft; SprintPilot benefits from upstream fixes and additions.
- Agent uses ADO tools via natural language planning — no explicit routing logic required.
- Authentication is handled by the MCP server process; PAT never touches SprintPilot application code.

**Negative / Risks:**
- Tool coverage must be validated in Phase 1. If a required operation (e.g., shared step creation for test plans) is not exposed, a custom tool must be written.
- The MCP server runs as a Node.js process (`npx @azure-devops/mcp`). Each pool slot adds one Node.js process (~50–100MB). Container sizing must account for this.
- `npx` pulls the package at startup; the container must have internet access to npm, or the package must be pre-installed in the image.
- Breaking changes to the MCP server's tool schemas would require prompt/skill updates.
