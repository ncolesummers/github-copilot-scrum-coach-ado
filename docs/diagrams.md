# SprintPilot — Architecture Diagrams

> Generated from `SprintPilot-PRD.md` v0.1.0

---

## 1. System Architecture

End-to-end view of all runtime layers, from the browser through FastAPI, the pooled CopilotClient, Copilot CLI, and the ADO MCP server, plus the supporting PostgreSQL store and the `/skills` directory.

```mermaid
graph TD
    subgraph Browser["Browser"]
        UI["React + shadcn/ui\nChat Interface"]
    end

    subgraph API["FastAPI — Python 3.12+"]
        WS["WebSocket Endpoint\n/api/v1/chat/ws"]
        REST["REST Endpoints\n/api/v1/reports/{id}/download"]
        Pool["CopilotClient Pool\n3–5 instances"]
    end

    subgraph AgentRuntime["Agent Runtime (per pool slot)"]
        CLI["Copilot CLI\nJSON-RPC over stdio"]
        MCP["ADO MCP Server\n@azure-devops/mcp\n(stdio child process)"]
    end

    subgraph External["External Services"]
        ADO["Azure DevOps\n(work items · test plans · wiki · search)"]
        GH["GitHub Copilot\n(LLM via Copilot subscription)"]
    end

    subgraph Persistence["Persistence"]
        PG[("PostgreSQL\nUsers · ChatSessions\nChatMessages · GeneratedReports")]
    end

    subgraph Skills["/skills directory"]
        S1["pptx/"]
        S2["docx/"]
        S3["sprint-review/"]
        S4["sprint-readiness/"]
        S5["work-item-decomposition/"]
        S6["test-plan/"]
        S7["development/"]
    end

    UI -- "WebSocket (streaming tokens,\ntool call events, file links)" --> WS
    UI -- "HTTP GET (file download)" --> REST
    WS --> Pool
    REST --> PG
    Pool -- "JSON-RPC / stdio" --> CLI
    CLI -- "stdio (child process)" --> MCP
    MCP -- "REST API (PAT auth)" --> ADO
    CLI -- "Copilot API calls" --> GH
    Pool -- "read/write sessions,\npersist reports" --> PG
    Pool -- "load skill instructions\nat session creation" --> Skills
```

---

## 2. Entity Relationship Diagram

Extends the FastAPI full-stack-template's existing `User` model. New entities are `ChatSession`, `ChatMessage`, and `GeneratedReport`.

```mermaid
erDiagram
    USER {
        uuid id PK
        string email
        string hashed_password
        boolean is_active
        boolean is_superuser
        string full_name
    }

    CHAT_SESSION {
        uuid id PK
        uuid owner_id FK
        string title
        string project_name
        string workflow_type
        datetime created_at
        datetime updated_at
    }

    CHAT_MESSAGE {
        uuid id PK
        uuid session_id FK
        string role
        text content
        json tool_call_metadata
        datetime created_at
    }

    GENERATED_REPORT {
        uuid id PK
        uuid session_id FK
        uuid owner_id FK
        string filename
        string content_type
        bytea file_data
        int file_size_bytes
        string report_type
        json metadata_
        datetime created_at
    }

    USER ||--o{ CHAT_SESSION : "owns"
    CHAT_SESSION ||--o{ CHAT_MESSAGE : "contains"
    CHAT_SESSION ||--o{ GENERATED_REPORT : "produces"
    USER ||--o{ GENERATED_REPORT : "owns"
```

> **`role`** values: `user` · `assistant` · `tool`
> **`report_type`** values: `sprint_review` · `work_item_export` · `test_plan_export`
> **`workflow_type`** values: `sprint_review` · `work_item_decomposition` · `sprint_readiness` · `test_plan`

---

## 3. Request Flow — WebSocket Chat Turn

Sequence from user message through FastAPI, the CopilotClient pool, Copilot CLI, and ADO MCP server, back to the browser as streaming tokens.

```mermaid
sequenceDiagram
    actor User
    participant UI as React Chat UI
    participant WS as FastAPI WebSocket
    participant Pool as CopilotClient Pool
    participant CLI as Copilot CLI
    participant MCP as ADO MCP Server
    participant ADO as Azure DevOps
    participant PG as PostgreSQL

    User->>UI: Types message & sends
    UI->>WS: WebSocket frame (ChatMessageCreate)
    WS->>PG: Persist ChatMessage (role=user)
    WS->>Pool: checkout CopilotClient
    Pool-->>WS: client handle

    WS->>CLI: JSON-RPC: session.send_message\n(+ loaded skill instructions)
    CLI->>CLI: Agent plans response,\nselects tools

    loop Tool invocations (0..n)
        CLI->>MCP: stdio: tool_call (e.g. list_work_items)
        MCP->>ADO: REST API request (PAT auth)
        ADO-->>MCP: JSON response
        MCP-->>CLI: tool_result
        CLI->>WS: event: tool.call (streamed)
        WS->>UI: WebSocket frame: tool indicator
    end

    CLI->>WS: event: assistant.message (streaming tokens)
    WS->>UI: WebSocket frames: streaming tokens
    UI->>User: Renders response in real time

    CLI->>WS: event: session.idle
    WS->>Pool: return CopilotClient
    WS->>PG: Persist ChatMessage (role=assistant)

    opt Report generated
        WS->>PG: Persist GeneratedReport (bytea)
        WS->>UI: WebSocket frame: report_ready (filename, report_id)
        UI->>User: Shows file download link
    end
```

---

## 4. Skills Architecture — Dynamic Loading

How skills are structured on disk and composed into the agent's system prompt at session creation, based on the detected workflow.

```mermaid
flowchart LR
    subgraph Disk["/skills  (versioned in repo)"]
        direction TB
        subgraph DocSkills["Document Creation Skills"]
            PPTX["pptx/\n├── SKILL.md\n├── references/\n│   ├── uidaho-brand-guide.md\n│   ├── slide-layouts.md\n│   └── chart-styles.md\n└── scripts/\n    └── create_pptx.py"]
            DOCX["docx/\n├── SKILL.md\n├── references/\n│   └── document-styles.md\n└── scripts/\n    └── create_docx.py"]
        end
        subgraph WorkflowSkills["Workflow Skills"]
            SR["sprint-review/\n├── SKILL.md\n├── references/\n│   ├── report-structure.md\n│   ├── stakeholder-comm.md\n│   ├── data-storytelling.md\n│   └── example-reports/\n└── templates/\n    └── sprint-review.pptx"]
            SRR["sprint-readiness/\n├── SKILL.md\n└── references/\n    ├── review-checklist.md\n    ├── risk-patterns.md\n    └── followup-actions.md"]
            WID["work-item-decomposition/\n├── SKILL.md\n└── references/\n    ├── hierarchy-rules.md\n    ├── acceptance-criteria.md\n    └── estimation-guide.md"]
            TP["test-plan/\n├── SKILL.md\n└── references/\n    ├── test-case-patterns.md\n    └── shared-steps.md"]
        end
        subgraph DevSkills["Development Skills"]
            DEV["development/\n├── SKILL.md\n└── references/\n    ├── fastapi-patterns.md\n    ├── copilot-sdk-patterns.md\n    └── pr-template.md"]
        end
    end

    subgraph Loader["core/skills.py — load_skills()"]
        WD["Detect workflow\nfrom user message"]
        LS["Load matching\nSKILL.md + references"]
        SP["build_system_prompt()\nbase + skill instructions"]
    end

    subgraph Session["CopilotClient Session"]
        SYS["system_message\n(base prompt + skill context)"]
        TOOLS["MCP tools\n(ADO domains)"]
    end

    WD -->|sprint_review| SR
    WD -->|sprint_review| PPTX
    WD -->|work_item_decomposition| WID
    WD -->|sprint_readiness| SRR
    WD -->|test_plan| TP
    WD -->|test_plan| DOCX
    SR --> LS
    PPTX --> LS
    WID --> LS
    SRR --> LS
    TP --> LS
    DOCX --> LS
    LS --> SP
    SP --> SYS
    TOOLS --> Session
    SYS --> Session
```

---

## 5. Phased Delivery Timeline

Five delivery phases across 26 weeks, showing the progression from foundation through sprint reports, work item creation, readiness review, and test plans.

```mermaid
gantt
    title SprintPilot — Phased Delivery (26 weeks)
    dateFormat  YYYY-MM-DD
    axisFormat  Wk %W

    section Phase 1 · Foundation
    Template setup & /docs structure         :p1a, 2026-02-23, 1w
    FastAPI WebSocket + streaming            :p1b, after p1a, 1w
    Copilot SDK pool integration             :p1c, after p1a, 1w
    ADO MCP Server (read-only)               :p1d, after p1b, 1w
    shadcn chat UI components                :p1e, after p1b, 1w
    Pydantic & Zod schema definitions        :p1f, after p1c, 1w
    Basic sprint summaries (text only)       :p1g, after p1d, 1w
    SPIKE — Skill-driven PPTX generation ⚑  :crit, p1h, after p1d, 1w
    Initial pptx + development skills        :p1i, after p1h, 1w
    Docker Compose + Azure deploy (dev)      :p1j, after p1g, 1w

    section Phase 2 · Sprint Reports
    sprint-review + docx skills              :p2a, 2026-03-23, 2w
    Refine pptx skill (spike findings)       :p2b, after p2a, 1w
    Multi-turn report authoring workflow     :p2c, after p2a, 3w
    Velocity trend analysis                  :p2d, after p2c, 1w
    Report persistence + download endpoint   :p2e, after p2c, 1w
    Report preview in chat UI                :p2f, after p2e, 1w
    Chat history persistence                 :p2g, after p2c, 1w
    Skill iteration (scrum master feedback)  :p2h, after p2f, 1w

    section Phase 3 · Work Item Creation
    work-item-decomposition skill            :p3a, 2026-05-04, 2w
    ADO write ops (create/update work items) :p3b, after p3a, 1w
    Hierarchy preview UI component           :p3c, after p3b, 1w
    Confirmation flow (human-in-the-loop)    :p3d, after p3c, 1w
    Duplicate detection (search_work_items)  :p3e, after p3d, 1w
    Batch creation + parent/child links      :p3f, after p3e, 2w

    section Phase 4 · Sprint Readiness
    sprint-readiness skill                   :p4a, 2026-06-15, 2w
    Backlog analysis + risk identification   :p4b, after p4a, 1w
    Structured findings presentation         :p4c, after p4b, 1w
    Agent-proposed actions + approval flow   :p4d, after p4c, 1w
    Cross-sprint dependency detection        :p4e, after p4d, 1w

    section Phase 5 · Test Plans
    test-plan skill                          :p5a, 2026-07-13, 2w
    AC → test case mapping                   :p5b, after p5a, 1w
    Edge case & negative scenario inference  :p5c, after p5b, 1w
    Shared step extraction                   :p5d, after p5c, 1w
    Traceability links + wiki publishing     :p5e, after p5d, 2w
```

> **⚑ Decision gate:** The Phase 1 PPTX spike (ADR-0006) must pass a quality bar before Phase 2 skill-driven report authoring proceeds. If it doesn't meet the bar, Phase 2 falls back to deterministic templates.

---

---

## 6. Observability Architecture

How OpenTelemetry signals flow from each instrumented component through the Collector to Jaeger (dev) and Azure Monitor (production). See ADR-0008.

```mermaid
flowchart LR
    subgraph App["Application (Docker Compose)"]
        direction TB
        FW["FastAPI\n(OTLP auto-instrumentation\nHTTP · WebSocket · SQLAlchemy)"]
        Pool["CopilotClient Pool\n(manual spans:\npool.checkout · pool.active)"]
        CLI["Copilot CLI Bridge\n(manual spans:\ncopilot.session.turn\ncopilot.tokens.input/output)"]
        MCP["MCP Tool Events\n(manual child spans:\nmcp.tool.call · duration)"]
        PG["PostgreSQL\n(auto-instrumented\nvia SQLAlchemy)"]
    end

    subgraph OTel["Observability Pipeline"]
        Collector["OTel Collector\n─────────────\nReceiver: OTLP gRPC :4317\nReceiver: OTLP HTTP :4318\nProcessor: batch\nExporter: jaeger\nExporter: prometheus"]
    end

    subgraph Backends["Backends"]
        Jaeger["Jaeger\n(all-in-one)\nUI :16686\nDev only — in-memory"]
        Prom["Prometheus\n(scrapes Collector\n:8888/metrics)"]
        AM["Azure Monitor\n(Application Insights)\nProduction only\nvia azuremonitor exporter"]
    end

    FW -- "OTLP gRPC\ntraces · metrics · logs" --> Collector
    Pool -- "OTLP gRPC" --> Collector
    CLI -- "OTLP gRPC" --> Collector
    MCP -- "OTLP gRPC" --> Collector
    PG -- "OTLP gRPC" --> Collector

    Collector -- "OTLP (traces)" --> Jaeger
    Collector -- "Prometheus scrape\n(metrics)" --> Prom
    Collector -- "azuremonitor\nexporter (prod)" --> AM

    FW -. "trace_id injected\ninto log records" .-> CLI
    CLI -. "trace context\npropagated via\nlog correlation" .-> MCP
```

**Key instrumented spans:**

| Span | Parent | Key Attributes |
|---|---|---|
| `http.server` / `websocket` | root | `http.method`, `http.route`, `user.id` |
| `chat.turn` | websocket | `session.id`, `workflow.type` |
| `pool.checkout` | chat.turn | `pool.wait_ms`, `pool.size` |
| `copilot.session.turn` | pool.checkout | `copilot.model`, `tokens.input`, `tokens.output` |
| `mcp.tool.call` | copilot.session.turn | `tool.name`, `ado.project`, `duration_ms` |
| `report.generate` | chat.turn | `report.type`, `report.size_bytes` |
| `db.query` | (any) | Auto — table, operation, duration |

---

## Diagram Index

| # | Diagram | Type | Key Insight |
|---|---|---|---|
| 1 | System Architecture | Flowchart | Stdio all the way down: FastAPI → CopilotClient pool → Copilot CLI → ADO MCP Server |
| 2 | Entity Relationship | ER | Extends template `User`; three new models: `ChatSession`, `ChatMessage`, `GeneratedReport` |
| 3 | Request Flow | Sequence | WebSocket chat turn with tool invocation loop and optional report persistence |
| 4 | Skills Architecture | Flowchart | Dynamic skill loading based on detected workflow; skills are loaded into `system_message` |
| 5 | Phased Delivery | Gantt | 26-week roadmap; Phase 1 PPTX spike gates Phase 2 approach |
| 6 | Observability Architecture | Flowchart | OTel signals → Collector → Jaeger (dev) / Azure Monitor (prod); key span inventory |
