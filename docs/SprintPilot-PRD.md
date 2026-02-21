# SprintPilot — Product Requirements Document

**AI-Powered Scrum Master Assistant for Azure DevOps**

| | |
|---|---|
| **Author** | OIT Application Development |
| **Organization** | University of Idaho – OIT |
| **Version** | 0.1.0 (Draft) |
| **Date** | February 14, 2026 |
| **Status** | Draft – Internal Review |

---

## 1. Executive Summary

SprintPilot is an internal, chat-driven web application that augments scrum master workflows at the University of Idaho's Office of Information Technology. The application gives scrum masters a conversational interface for generating sprint reports, reviewing sprint backlog readiness, converting unstructured requirements into Azure DevOps work items, and scaffolding test plans — all within an already-approved technology stack.

The system is built on three pillars:

- **GitHub Copilot SDK (Python)** as the agent runtime, providing LLM orchestration, tool execution, and session management through the Copilot CLI.
- **Azure DevOps MCP Server** (`@azure-devops/mcp`) connected to the Copilot agent via the SDK's native MCP integration, giving the agent direct access to ADO's work items, test plans, wiki, search, and project configuration — without writing custom API wrappers.
- **FastAPI + React** using the [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) as the application scaffold, with shadcn/ui chat components on the frontend.

Rather than replacing Azure DevOps processes, SprintPilot acts as an intelligent intermediary: the Copilot agent reads from and writes to ADO through the MCP server's toolset, plans multi-step operations autonomously, and streams results to users through a chat interface that requires minimal training to adopt.

> **Why Now?**
>
> - GitHub Copilot is already approved and deployed across OIT.
> - Scrum ceremony artifacts (sprint reviews, retrospectives) consume significant manual effort.
> - AI literacy initiatives are actively underway — SprintPilot serves as both a productivity tool and a reference implementation for AI-assisted development patterns.
> - The Azure DevOps MCP Server is now GA, and the Copilot SDK supports MCP servers natively, making the integration path well-defined.

---

## 2. Problem Statement

OIT scrum teams spend a disproportionate amount of time on administrative ceremony work that follows predictable patterns but still requires manual effort:

- **Sprint review reports** require aggregating completed work items, summarizing accomplishments, capturing metrics, and formatting the output for stakeholders — often as both a written summary and a PowerPoint deck.
- **Requirements and meeting transcripts** arrive as unstructured text (emails, Teams chat logs, meeting notes) and must be manually decomposed into epics, user stories, tasks, and acceptance criteria in Azure DevOps.
- **Test plans** are created ad hoc, often missing coverage for edge cases that a systematic approach would catch. Test suites, test cases, and shared steps must be manually entered into Azure DevOps Test Plans.
- **Context switching** between the chat/document where requirements live and the Azure DevOps UI where work items are created introduces friction, errors, and lost nuance.

These pain points are amplified across multiple teams and projects. A conversational AI tool that understands Azure DevOps' data model and can interact with it directly would eliminate the translation overhead while preserving human judgment in the loop.

---

## 3. Goals and Non-Goals

### 3.1 Goals

1. Reduce time-to-artifact for sprint review reports and presentations by at least 60%.
2. Enable conversational decomposition of unstructured requirements into properly structured Azure DevOps work items with parent/child relationships.
3. Provide AI-assisted test plan generation that produces test suites, test cases, and shared steps directly in Azure DevOps Test Plans.
4. Leverage the Azure DevOps MCP Server for all ADO interactions, avoiding custom API integration code.
5. Develop and maintain a library of agent skills that encode OIT's branding, communication standards, and workflow expertise — usable both within SprintPilot and as reference patterns for other AI-assisted projects.
6. Establish a reference architecture for Copilot SDK + MCP + skills composition that other OIT teams can learn from.
7. Operate entirely within already-approved tooling (GitHub Copilot subscription, Azure AD, University SSO).

### 3.2 Non-Goals

- Replacing Azure DevOps Boards UI for day-to-day work item management.
- Automated sprint planning or capacity allocation (future phase).
- Integration with non-Azure DevOps project management tools.
- Public-facing or student-facing deployment.
- Real-time collaboration (multi-user simultaneous editing of the same session).
- Building custom Azure DevOps REST API wrappers — the MCP server handles this.

---

## 4. Users and Personas

| Persona | Description | Primary Workflows |
|---|---|---|
| **Scrum Master** | Facilitates ceremonies, generates reports, tracks velocity and health metrics across sprints. | Sprint review reports, retrospective summaries, velocity charts, PowerPoint generation |
| **Product Owner** | Manages backlog, decomposes stakeholder requirements into actionable work items. | Requirements → work items, acceptance criteria generation, backlog grooming prep |
| **QA Lead / Developer** | Creates and maintains test plans, ensures coverage aligns with acceptance criteria. | Test plan generation, test case scaffolding, shared step creation |
| **Team Lead** | Oversees technical delivery, needs quick status summaries. | Sprint summaries, blocker identification, cross-team dependency reports |

---

## 5. Core Workflows

### 5.1 Sprint Review Report Generation

The user initiates a sprint review report through natural language. The Copilot agent uses ADO MCP tools to gather sprint data, then *authors* the report using document creation skills — iteratively building the narrative, charts, and slides while the user steers the process through conversation. This is a multi-turn collaborative workflow, not a one-shot template fill.

**Example Interaction:**

> **User:** "Generate a sprint review report for Sprint 24 in the MyUI project."
>
> **SprintPilot:** Queries ADO via MCP tools for Sprint 24's iteration path, retrieves all work items (completed, carried over, added mid-sprint), calculates velocity. Presents a summary: *"I found 23 completed items and 4 carried over. Here's my plan for the report: executive summary, completed work by epic, carried-over items with context, blockers, and velocity trends. I'll generate both a Word doc and a PowerPoint deck. Want me to proceed with this structure, or adjust?"*
>
> **User:** "Proceed, but emphasize the blockers — two of them are cross-team dependencies that leadership needs to see."
>
> **SprintPilot:** Drafts the report, promoting the blocker section with a dedicated slide including a dependency diagram. Presents the draft for review: *"Here's the first draft. The blocker section is on slides 4–5 with a visual dependency map. The velocity chart shows a 3-sprint trend. Take a look — what would you change?"*
>
> **User:** "The velocity chart needs to go back 5 sprints, not 3. And soften the language on the missed deliverable — this is going to the VP."
>
> **SprintPilot:** Revises the deck, adjusting the chart range and refining the tone per the communication best practices in its skill references. Presents the final version for download.

**Report Contents (flexible, guided by skill):**

- Sprint goal and whether it was met
- Completed work items grouped by epic with narrative descriptions
- Carried-over items with reason codes and remaining effort
- Blocked items with blocker details, resolution status, and escalation recommendations
- Velocity metrics: planned vs. delivered story points, configurable trend window
- Burn-down/burn-up summary with visual charts
- Key decisions, risks, or dependencies surfaced during the sprint
- PowerPoint deck with University of Idaho branding, professional layout, data-driven narrative

> **Deterministic Metrics Requirement:** Story point calculations (completed, carried over, planned) must be computed deterministically from ADO work item data — not generated or estimated by the LLM. The agent queries User Story and Bug work items in the sprint iteration, sums their Story Points fields, and passes these numbers to the report as fixed values. The agent may *narrate* the metrics (trends, comparisons, context), but the underlying numbers must come directly from ADO data and be verifiable. This is critical for stakeholder trust and reduces the impact of hallucinations on quantitative claims.

**ADO MCP Tools Used:**

- `list_work_items` — query by iteration path
- `get_work_item_details` — expand fields and relations
- `get_sprint_details` — sprint dates, capacity
- `search_work_items` — find related/blocked items

**Agent Skills Used:**

- `pptx` skill — slide creation, layout conventions, chart generation, branding enforcement
- `sprint-review` skill — report structure, section ordering, narrative best practices, stakeholder communication guidelines (see §6.4)

### 5.2 Requirements → Work Item Decomposition

Users paste or upload unstructured requirements (meeting transcripts, emails, PRD excerpts, stakeholder requests) and SprintPilot decomposes them into a structured hierarchy of Azure DevOps work items.

**Decomposition Model:**

| Level | ADO Work Item Type | Generated Fields |
|---|---|---|
| Epic | Epic | Title, Description, Business Value, Acceptance Criteria |
| User Story | User Story | Title, As a/I want/So that format, Acceptance Criteria, Story Points (estimate), Parent link to Epic |
| Task | Task | Title, Description, Remaining Work (estimate), Activity type, Parent link to User Story |
| Bug (if identified) | Bug | Title, Repro Steps, Severity, Priority, Story Points, Parent link to Epic or User Story |

> **Note:** OIT's Azure DevOps process does not use the Feature work item type. The hierarchy is Epic → User Story → Task, with Bugs linked to Epics or User Stories as appropriate.

**Example Interaction:**

> **User:** *[pastes a meeting transcript about new reporting requirements]*
>
> **SprintPilot:** Analyzes the transcript and identifies 2 epics and 12 user stories. Presents the proposed hierarchy in a structured preview with edit capabilities.
>
> **User:** "Merge stories 3 and 4, and add an acceptance criterion to story 7 about handling null values."
>
> **SprintPilot:** Updates the hierarchy and asks for confirmation before committing to Azure DevOps.
>
> **User:** "Looks good. Create these in the MyUI project under the Q3 Reporting epic."
>
> **SprintPilot:** Creates all work items via the MCP server's `create_work_item` tool with parent/child links and reports back with direct links to each item.

**Key Behaviors:**

- **Human-in-the-loop:** SprintPilot always presents the proposed work item hierarchy for review before creating anything in ADO.
- **Iterative refinement:** users can modify, merge, split, or add items conversationally before committing.
- **Context preservation:** the original source text is linked in the work item description for traceability.
- **Duplicate detection:** before creation, SprintPilot uses the MCP server's `search_work_items` tool to check for existing items with similar titles/descriptions.

### 5.3 Sprint Readiness Review

Before a sprint begins, scrum masters need to verify that the work items in the iteration are well-defined, feasible, and free of hidden risks. SprintPilot reviews the upcoming sprint's user stories and tasks, identifies implementation risks, gaps, and dependencies, and suggests concrete follow-up actions it can take to improve the work items.

**Example Interaction:**

> **User:** "Review Sprint 25 readiness for the MyUI project."
>
> **SprintPilot:** Queries all work items in the Sprint 25 iteration. Analyzes each user story and task for completeness, then presents findings:
>
> *"I reviewed 14 user stories and 31 tasks in Sprint 25. Here's what I found:*
>
> *🔴 **2 high-risk items:** Story US-412 ('Integrate with Banner API') has no acceptance criteria and references an external dependency that isn't represented in ADO. Task T-891 estimates 2 hours but involves a database migration with no rollback plan mentioned.*
>
> *🟡 **4 items with gaps:** Stories US-408 and US-410 have acceptance criteria but no child tasks. US-415 references 'the new auth flow' but there's no linked story covering auth implementation in this sprint or the backlog.*
>
> *🟢 **8 items look solid.***
>
> *Want me to add acceptance criteria to US-412, create tasks for US-408 and US-410, and flag the auth dependency on US-415?"*
>
> **User:** "Yes to all. For US-412, base the AC on the Banner API docs we discussed last sprint. And create a spike task for the migration rollback plan on T-891."
>
> **SprintPilot:** Updates the work items via MCP write tools, adding acceptance criteria, creating child tasks, adding a dependency tag to US-415, and creating a spike task linked to T-891. Reports back with links to all modified and created items.

**Analysis Dimensions:**

- **Completeness:** User stories without acceptance criteria; stories without child tasks; tasks without descriptions or estimates
- **Implementation risks:** Large estimates with vague descriptions; references to unfamiliar systems or APIs; database changes without migration/rollback considerations
- **Hidden dependencies:** References to work not in the current sprint or backlog; cross-team dependencies not represented as linked items; external system integrations without documented contracts
- **Missed steps:** Features that typically require supporting work (e.g., auth changes need session invalidation, API changes need documentation updates, UI changes need accessibility review)
- **Estimation concerns:** Tasks with suspiciously low or high estimates given their description; stories with no estimates

**Agent Actions (with user approval):**

- Add or refine acceptance criteria on user stories
- Create missing child tasks for user stories
- Add dependency tags or links between related work items
- Create spike/investigation tasks for identified risks
- Update descriptions with flagged concerns or open questions
- Adjust story point estimates with rationale

**ADO MCP Tools Used:**

- `list_work_items` — query by iteration path for upcoming sprint
- `get_work_item_details` — read full fields, relations, and history
- `search_work_items` — find related items, check for missing dependencies in backlog
- `create_work_item` — create missing tasks, spikes
- `update_work_item` — add acceptance criteria, tags, descriptions

**Agent Skills Used:**

- `sprint-readiness` skill — review checklist, risk patterns, common gaps by work item type, follow-up action templates (see §6.4)

### 5.4 Test Plan Generation

Given a set of user stories (referenced by ID or provided as text), SprintPilot generates a comprehensive test plan structure in Azure DevOps Test Plans.

**Generated Artifacts:**

| ADO Artifact | Source | Generated Content |
|---|---|---|
| Test Plan | Sprint or epic scope | Name, area path, iteration, start/end dates |
| Test Suite | User story or epic | Suite per user story, linked to corresponding work item |
| Test Case | Acceptance criteria + inferred scenarios | Title, steps (action/expected result pairs), priority, automation status |
| Shared Steps | Common patterns across test cases | Reusable step groups (e.g., login flow, navigation to module) |

**Key Behaviors:**

- **Acceptance-criteria-first:** each acceptance criterion on a user story maps to at least one test case.
- **Edge case inference:** SprintPilot suggests boundary conditions, error states, and negative test cases beyond what's explicitly stated.
- **Shared step detection:** when multiple test cases share setup or verification sequences, SprintPilot extracts them into shared steps.
- **Traceability:** test cases are linked back to their source user stories.

**ADO MCP Tools Used:**

- `create_test_plan` — create plans with area/iteration path
- `create_test_suite` — static and requirement-based suites
- `add_test_cases` — test cases with step definitions
- `get_work_item_details` — read acceptance criteria from source stories

---

## 6. Technical Architecture

### 6.1 System Overview

SprintPilot is built on the [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template), a production-ready scaffold that provides FastAPI (Python) on the backend, React with TypeScript on the frontend, PostgreSQL for persistence, Docker Compose for orchestration, and Traefik for reverse proxy / HTTPS. The template ships with shadcn/ui components, Tailwind CSS, JWT auth, and Playwright testing — all of which SprintPilot extends rather than replaces.

The application composes three runtime layers:

| Layer | Technology | Responsibility |
|---|---|---|
| **Presentation** | React + shadcn/ui chat components | Chat UI, message history, streaming display, file download handling |
| **API** | FastAPI (Python) | WebSocket/SSE endpoints for chat, session management, file generation endpoints, auth |
| **Agent Runtime** | GitHub Copilot SDK (`github-copilot-sdk`) | LLM orchestration via Copilot CLI, tool loop execution, MCP server management, model access |
| **ADO Integration** | Azure DevOps MCP Server (`@azure-devops/mcp`) | Work item CRUD, test plan management, sprint queries, search, wiki — all via MCP tools |
| **Auth (App)** | Template's built-in JWT auth + Azure AD | User authentication, session management |
| **Auth (ADO)** | Service Account PAT | ADO MCP server authenticates with a scoped PAT for all ADO API operations |
| **Persistence** | PostgreSQL (from template) | Chat history, session metadata, user preferences, generated reports (binary, stored as `bytea`) |

### 6.2 Request Flow

1. User sends a message via the shadcn chat component, which connects to a FastAPI WebSocket endpoint (`/api/v1/chat/ws`).
2. The FastAPI handler creates or resumes a `CopilotClient` session (Python SDK) with the ADO MCP server configured.
3. The message is forwarded to the Copilot CLI runtime via JSON-RPC.
4. The Copilot agent plans a response, potentially invoking ADO MCP tools (work item queries, creation, test plan scaffolding) and/or custom tools (document generation).
5. MCP tool invocations are handled transparently by the Copilot CLI, which communicates with the ADO MCP server process via stdio.
6. The agent's streaming response events (`assistant.message`, `tool.call`, `session.idle`) are forwarded over the WebSocket to the React frontend.
7. The chat UI renders streaming tokens, tool call indicators, and file download links in real time.

### 6.3 Copilot SDK + ADO MCP Server Configuration

The core integration happens in the Copilot SDK session creation. The ADO MCP server is registered as a local MCP server, and the Copilot CLI manages its lifecycle:

```python
from copilot import CopilotClient

client = CopilotClient()
await client.start()

session = await client.create_session({
    "model": "gpt-5",
    "mcp_servers": {
        "azure-devops": {
            "type": "local",
            "command": "npx",
            "args": [
                "-y", "@azure-devops/mcp", 
                ADO_ORG_NAME,
                "-d", "core", "work", "work-items", 
                "test-plans", "search", "wiki"
            ],
            "env": {
                "AZURE_DEVOPS_PAT": settings.ADO_PAT,
            },
            "tools": ["*"],
        },
    },
    "system_message": {
        "content": SPRINTPILOT_SYSTEM_PROMPT,
    },
})
```

**MCP Domains Enabled:**

| Domain | Tools Provided | Used By |
|---|---|---|
| `core` | `list_projects`, `get_project_details`, `get_me` | All workflows — project/team context |
| `work` | `get_sprint_details`, team/iteration settings | Sprint report generation, capacity data |
| `work-items` | `list_work_items`, `get_work_item_details`, `create_work_item`, `update_work_item` | All workflows — CRUD on work items |
| `test-plans` | `create_test_plan`, `create_test_suite`, `add_test_cases` | Test plan generation |
| `search` | `search_work_items`, `search_code` | Duplicate detection, cross-referencing |
| `wiki` | `get_wiki_page`, `create_wiki_page` | Future — publish reports to wiki |

### 6.4 Agent Skills and Document Creation

SprintPilot uses **agent skills** as the primary mechanism for document creation and workflow expertise. Skills are structured directories containing instructions, reference materials, and helper scripts that teach the Copilot agent how to perform specific tasks at a professional level. Skills are a first-class deliverable of this project — they are developed, reviewed, versioned, and iterated alongside the application code.

**How skills work with the Copilot CLI:**

The Copilot CLI supports custom instructions and tool definitions. Skills are loaded into the agent's session via the `system_message` configuration and, where applicable, as custom tools that wrap helper scripts. The agent reads the skill's instructions, follows its conventions, and uses its reference materials to produce output that meets OIT's quality standards.

**Skill directory structure:**

```
/skills
├── pptx/
│   ├── SKILL.md                  # Instructions for creating PPTX files
│   ├── references/
│   │   ├── uidaho-brand-guide.md # University brand colors, fonts, logo usage
│   │   ├── slide-layouts.md      # Approved slide layouts and conventions
│   │   └── chart-styles.md       # Chart formatting (colors, labels, axes)
│   └── scripts/
│       └── create_pptx.py        # python-pptx helper for slide generation
├── docx/
│   ├── SKILL.md                  # Instructions for creating DOCX files
│   ├── references/
│   │   └── document-styles.md    # Heading hierarchy, margins, fonts
│   └── scripts/
│       └── create_docx.py        # python-docx helper for document generation
├── sprint-review/
│   ├── SKILL.md                  # How to author a sprint review report
│   ├── references/
│   │   ├── report-structure.md   # Section ordering, required vs optional sections
│   │   ├── stakeholder-comm.md   # Best practices for communicating with university leadership
│   │   ├── data-storytelling.md  # Using metrics to tell a compelling story
│   │   └── example-reports/      # Exemplar reports for tone and structure reference
│   └── templates/
│       └── sprint-review.pptx    # Branded PPTX template (master slides, layouts)
├── sprint-readiness/
│   ├── SKILL.md                  # How to review sprint backlog for risks and gaps
│   └── references/
│       ├── review-checklist.md   # Completeness checks by work item type
│       ├── risk-patterns.md      # Common implementation risks and red flags
│       └── followup-actions.md   # Action templates (add AC, create spike, flag dependency)
├── work-item-decomposition/
│   ├── SKILL.md                  # How to decompose requirements into ADO work items
│   └── references/
│       ├── hierarchy-rules.md    # Epic → User Story → Task/Bug conventions (no Features)
│       ├── acceptance-criteria.md # AC writing guidelines
│       └── estimation-guide.md   # Story point estimation heuristics
├── test-plan/
│   ├── SKILL.md                  # How to generate test plans from user stories
│   └── references/
│       ├── test-case-patterns.md # Edge case inference, negative testing
│       └── shared-steps.md       # When and how to extract shared steps
└── development/
    ├── SKILL.md                  # Project coding conventions and patterns
    └── references/
        ├── fastapi-patterns.md   # How to extend the template correctly
        ├── copilot-sdk-patterns.md # Session management, error handling
        └── pr-template.md        # Pull request format and review checklist
```

**Skill categories:**

| Category | Skills | Purpose |
|---|---|---|
| **Document creation** | `pptx`, `docx` | Low-level file generation with branding, layout rules, and helper scripts. Adapted from existing skill patterns with University of Idaho branding and conventions. |
| **Workflow** | `sprint-review`, `sprint-readiness`, `work-item-decomposition`, `test-plan` | Domain-specific expertise — how to structure a report, review a sprint backlog, decompose requirements, or generate test cases. Includes references on professional communication and data storytelling. |
| **Development** | `development` | Coding conventions, architecture patterns, and PR standards for developing SprintPilot itself. Used by developers working with Copilot on this project. |

**Skill development is ongoing work.** Skills are iterated based on user feedback — when a scrum master says "the report tone was too technical for the VP audience," that feedback becomes a refinement to the `stakeholder-comm.md` reference. When a new report type is needed (retrospective, release notes), a new workflow skill is created. The `/skills` directory is part of the repository and goes through the same review process as code.

**Relationship to Copilot SDK session configuration:**

```python
session = await client.create_session({
    "model": "gpt-5",
    "mcp_servers": { ... },  # ADO MCP server
    "system_message": {
        "content": build_system_prompt(
            base_prompt=SPRINTPILOT_BASE_PROMPT,
            skills=load_skills(["sprint-review", "pptx"]),  # Dynamic based on workflow
        ),
    },
})
```

Skills are loaded dynamically based on the detected workflow. A sprint review request loads the `sprint-review` and `pptx` skills. A work item decomposition request loads `work-item-decomposition`. The agent's effective context includes the skill instructions and references, giving it the knowledge to produce professional, OIT-standard output.

### 6.5 Technology Stack

| Component | Technology | Rationale |
|---|---|---|
| **Project Scaffold** | [full-stack-fastapi-template](https://github.com/fastapi/full-stack-fastapi-template) | Production-ready FastAPI + React + Docker; ships with auth, DB, CI/CD |
| **Backend** | FastAPI (Python 3.12+) | Async-native, WebSocket support, Pydantic validation, team familiarity |
| **Frontend** | React + TypeScript + Vite | From template; Tailwind CSS, shadcn/ui, TanStack Router/Query |
| **Chat UI** | shadcn/ui chat components | Consistent with template's design system, customizable |
| **Agent Runtime** | `github-copilot-sdk` (Python) | Approved Copilot subscription, agentic tool loop, MCP server support, model access |
| **ADO Integration** | `@azure-devops/mcp` (GA) | Official Microsoft MCP server; work items, test plans, wiki, search, pipelines |
| **Database** | PostgreSQL (from template) | Chat history, session state, user preferences, generated report storage (bytea) |
| **ORM** | SQLModel (from template) | Pydantic + SQLAlchemy hybrid, already configured |
| **Doc Generation** | python-pptx, python-docx | Server-side Office document generation, wrapped in skill helper scripts |
| **Agent Skills** | Markdown + Python scripts in `/skills` | Workflow instructions, branding references, communication guides, helper scripts; loaded into agent context dynamically |
| **Auth (App)** | JWT via template + Azure AD (future) | Built-in auth works for Phase 1; Azure AD SSO for production |
| **Auth (ADO)** | Service Account PAT | Scoped PAT passed to MCP server via environment variable |
| **Testing** | pytest + Playwright (from template) | Unit tests for tools/API, E2E for chat workflows |
| **Deployment** | Docker Compose → Azure (container) | Template ships with Docker Compose + Traefik; maps to Azure Container Apps |
| **CI/CD** | GitHub Actions (from template) | Pre-configured workflows for test + deploy |

### 6.6 Extending the Template

The full-stack-fastapi-template provides a working application with users, items, auth, and CRUD. SprintPilot extends it with:

| Template Component | SprintPilot Extension |
|---|---|
| `backend/app/api/routes/` | Add `chat.py` with WebSocket endpoint, `sessions.py` for session management |
| `backend/app/models.py` | Add `ChatSession`, `ChatMessage`, `GeneratedReport` SQLModel classes |
| `backend/app/core/` | Add `copilot.py` for pooled CopilotClient lifecycle management, `prompts.py` for system prompt builder, `skills.py` for skill loader |
| `frontend/src/routes/` | Add `/_layout/chat.tsx` route with shadcn chat components |
| `frontend/src/components/` | Add `Chat/`, `WorkItemPreview/`, `FileDownload/`, `ReportPreview/` component directories |
| `frontend/src/client/` | Extend auto-generated API client with WebSocket hooks |
| `.env` | Add `ADO_PAT`, `ADO_ORG_NAME`, `COPILOT_MODEL`, `COPILOT_POOL_SIZE` variables |
| `compose.yml` | Ensure Node.js 18+ available for Copilot CLI + MCP server processes |
| `/skills` | New top-level directory: `pptx/`, `docx/`, `sprint-review/`, `sprint-readiness/`, `work-item-decomposition/`, `test-plan/`, `development/` (see §6.4) |
| `/docs` | New top-level directory: `decisions/` for ADRs, `contracts/` for API contract documentation and WebSocket protocol spec (see §6.9) |

### 6.7 Concurrency Model — Pooled CopilotClient

Each `CopilotClient` instance spawns a Copilot CLI process (JSON-RPC over stdio), which in turn spawns the ADO MCP server as a child process:

```
FastAPI worker
  └─ CopilotClient (from pool)
       └─ copilot CLI process (JSON-RPC / stdio)
            └─ npx @azure-devops/mcp (stdio child)
```

A single CLI process can multiplex multiple sessions (each session is an independent state machine), but practical throughput degrades under heavy concurrent tool loops. For SprintPilot's usage profile (average ~3 concurrent users, peak up to 30), we use a **pooled client pattern**:

- **Pool size:** 3–5 `CopilotClient` instances, each with its own CLI + MCP server process tree. Configurable via `COPILOT_POOL_SIZE` environment variable.
- **Checkout/return:** FastAPI's async WebSocket handler checks out a client from the pool for each user interaction, runs the session turn, and returns the client. Sessions persist across messages (Copilot SDK supports this), but client binding is per-request, not per-user.
- **Backpressure:** If all clients are busy, new requests queue with a configurable timeout. At peak load (30 users), some users may experience brief delays rather than degraded responses.
- **Health monitoring:** Each client in the pool has a health check; unhealthy clients (CLI process crashed, MCP server unresponsive) are recycled automatically.
- **Resource footprint:** Each CLI + MCP server pair is roughly one Node.js process (~50–100MB). A pool of 5 adds ~500MB to the container's memory baseline.

**Scaling beyond the prototype:** If usage grows beyond 30 users or the OIT deploys SprintPilot to additional departments, the Copilot SDK supports **BYOK (Bring Your Own Key)** with pay-per-request pricing from supported LLM providers (OpenAI, Azure AI Foundry, Anthropic). This decouples scaling from the Copilot premium request quota and allows horizontal scaling of FastAPI workers, each with their own client pool, behind a load balancer.

### 6.8 Report Storage

Generated reports (PPTX, DOCX, CSV) are persisted in PostgreSQL as binary data. This avoids introducing a separate blob store while keeping reports accessible across sessions and tied to the user/session that created them.

**SQLModel schema:**

```python
class GeneratedReport(SQLModel, table=True):
    id: uuid.UUID = Field(default_factory=uuid4, primary_key=True)
    session_id: uuid.UUID = Field(foreign_key="chatsession.id", index=True)
    owner_id: uuid.UUID = Field(foreign_key="user.id", index=True)
    filename: str                         # e.g., "Sprint-24-Review.pptx"
    content_type: str                     # e.g., "application/vnd.openxmlformats-officedocument.presentationml.presentation"
    file_data: bytes                      # bytea column — binary content
    file_size_bytes: int
    report_type: str                      # "sprint_review" | "work_item_export" | "test_plan_export"
    metadata_: dict = Field(default={}, sa_column_kwargs={"name": "metadata"})  # Sprint name, project, date range, etc.
    created_at: datetime = Field(default_factory=datetime.utcnow)
```

**Design decisions:**

- **bytea over file system:** Reports are small (typically < 5MB). PostgreSQL `bytea` keeps storage transactional with the session data, simplifies backups, and avoids volume mount complexity in containers.
- **Retention:** Reports are retained indefinitely by default. A future cleanup job can prune reports older than a configurable threshold.
- **Download endpoint:** `GET /api/v1/reports/{report_id}/download` streams the binary content with appropriate `Content-Type` and `Content-Disposition` headers. Auth-gated to the report owner.
- **Size guard:** Reports exceeding 20MB are rejected at generation time (this threshold is well above expected sizes and prevents accidental DB bloat).

### 6.9 Project Documentation

SprintPilot maintains two categories of living documentation alongside the codebase: architecture decision records that capture the *why* behind technical choices, and API contracts that enforce type safety across the frontend/backend boundary.

**Architecture Decision Records (ADRs)**

All significant technical decisions are recorded as lightweight ADRs in the repository. ADRs capture context, options considered, the decision made, and consequences — making it possible for future contributors (or future-us) to understand why the system is shaped the way it is without archaeology through commit history or chat logs.

```
/docs
├── decisions/
│   ├── 0001-use-fastapi-template.md
│   ├── 0002-copilot-sdk-over-direct-llm.md
│   ├── 0003-ado-mcp-server-over-custom-api.md
│   ├── 0004-pooled-copilot-client.md
│   ├── 0005-postgresql-bytea-for-reports.md
│   ├── 0006-skill-driven-document-generation.md
│   ├── 0007-work-item-hierarchy-no-features.md
│   └── ...
```

ADR format follows [Michael Nygard's template](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions): Title, Status (proposed/accepted/deprecated/superseded), Context, Decision, Consequences. New ADRs are submitted as part of the PR that implements the decision. The Phase 1 PPTX spike (§9, Phase 1) will produce ADR-0006's final status — accepted if skill-driven generation meets the quality bar, superseded if deterministic templates are chosen instead.

**Frontend/Backend API Contracts**

Type safety across the WebSocket and REST boundaries is maintained through shared contract definitions. The backend defines canonical models in **Pydantic**, and the frontend mirrors them in **Zod** schemas (for runtime validation) and **TypeScript interfaces** (for compile-time safety). These contracts are the source of truth for what data crosses the wire.

```
# Backend (Pydantic) — backend/app/schemas/
class ChatMessageCreate(BaseModel):
    content: str
    session_id: uuid.UUID

class ChatMessageResponse(BaseModel):
    id: uuid.UUID
    role: Literal["user", "assistant", "tool"]
    content: str
    tool_call_metadata: dict | None = None
    created_at: datetime

class ReportResponse(BaseModel):
    id: uuid.UUID
    filename: str
    content_type: str
    file_size_bytes: int
    report_type: str
    metadata_: dict
    created_at: datetime
```

```
// Frontend (Zod) — frontend/src/contracts/
export const ChatMessageResponseSchema = z.object({
  id: z.string().uuid(),
  role: z.enum(["user", "assistant", "tool"]),
  content: z.string(),
  tool_call_metadata: z.record(z.unknown()).nullable(),
  created_at: z.string().datetime(),
});

export type ChatMessageResponse = z.infer<typeof ChatMessageResponseSchema>;
```

**Contract maintenance rules:**

- Pydantic models are the source of truth. Frontend Zod schemas must match.
- Any PR that changes a Pydantic schema must include the corresponding Zod schema update.
- WebSocket message types (streaming chunks, tool call notifications, report-ready events) are defined as discriminated unions in both languages.
- The `development` skill (§6.4) includes contract update procedures so AI-assisted development follows this pattern consistently.

```
/docs
├── decisions/               # ADRs
├── contracts/
│   ├── README.md            # Contract maintenance rules and sync procedures
│   ├── websocket-protocol.md # WebSocket message types and sequencing
│   └── rest-endpoints.md    # REST endpoint inventory with request/response schemas
```

---

## 7. Authentication and Authorization

### 7.1 Application Auth

Phase 1 uses the template's built-in JWT authentication (email/password). Users are provisioned manually for the small OIT team. Future phases will integrate Azure AD SSO via the existing University identity provider, replacing local auth while keeping the JWT session mechanism.

### 7.2 Azure DevOps Auth

A single **service account PAT** is used for all ADO operations. The PAT is:

- Scoped to the minimum permissions required (see below)
- Stored as an environment variable (`ADO_PAT`), never in code or database
- Passed to the ADO MCP server process via the Copilot SDK's `env` configuration
- Rotated per University IT security policy

**Required PAT Scopes:**

| Scope | Access Level | Used By |
|---|---|---|
| Work Items | Read & Write | All workflows: query, create, update work items |
| Test Management | Read & Write | Test plan generation |
| Project and Team | Read | Iteration paths, team settings, area paths |
| Wiki | Read & Write | Future: publish reports to wiki |
| Search | Read | Duplicate detection |

**Tradeoff:** A service account PAT means all ADO operations are attributed to the service account, not individual users. This is acceptable for Phase 1 given the small team size. Phase 2+ can migrate to delegated OAuth for per-user attribution if needed.

### 7.3 GitHub Copilot Auth

The Copilot CLI authenticates using the host machine's GitHub credentials (or a `GITHUB_TOKEN` in containerized deployments). The OIT Copilot subscription covers SDK usage, with each prompt counting against the premium request quota.

---

## 8. Non-Functional Requirements

| Category | Requirement | Target |
|---|---|---|
| **Performance** | Time to first streamed token after user message | < 2 seconds |
| **Performance** | Sprint report generation (full report + PPTX) | < 30 seconds for sprints with ≤ 50 work items |
| **Performance** | Work item batch creation | < 15 seconds for ≤ 20 items with links |
| **Availability** | Uptime during business hours (M–F, 7am–6pm PT) | 99.5% |
| **Security** | ADO PAT storage | Environment variable only, never in DB or logs |
| **Security** | Data residency | All data remains within University Azure tenant |
| **Compliance** | FERPA considerations | No student PII processed; tool is staff-only |
| **Scalability** | Concurrent users | Up to 30 concurrent sessions via pooled CopilotClient (pool of 3–5); average ~3 |
| **Scalability** | Report storage | PostgreSQL bytea; 20MB per-file limit; indefinite retention with future pruning |
| **Observability** | Logging | Structured logs for all MCP tool invocations and agent events |
| **Observability** | Metrics | Token usage, tool call frequency, session duration, error rates |
| **Accessibility** | WCAG 2.1 AA | Chat interface meets accessibility standards |

---

## 9. Phased Delivery Plan

### Phase 1: Foundation (Weeks 1–4)

**Milestone:** Chat interface connected to Copilot SDK with ADO read access + validated skill-driven document creation

- Fork and configure full-stack-fastapi-template
- Establish `/docs` directory: initial ADRs (template choice, Copilot SDK, ADO MCP server, pooled client, skill-driven generation), contract documentation structure, WebSocket protocol spec
- Implement FastAPI WebSocket endpoint for chat with streaming
- Define initial Pydantic schemas and corresponding Zod schemas for chat message types (§6.9)
- Integrate `github-copilot-sdk` Python client with pooled client lifecycle management
- Configure ADO MCP server with `core`, `work`, `work-items` domains (read-only usage)
- Build shadcn chat components (message list, input, streaming indicators, file download)
- Basic sprint summary generation (text only, no PPTX)
- **Spike: Skill-driven PPTX generation.** Adapt existing pptx skill patterns to include University of Idaho branding (colors, fonts, logo, slide layouts). Validate that the Copilot agent can iteratively build a slide deck using skill instructions and python-pptx helper scripts. Evaluate output quality, token cost, and generation time. This spike gates the approach for Phase 2 — if skill-driven generation doesn't meet quality bar, fall back to deterministic templates.
- Establish `/skills` directory structure; draft initial `pptx` and `development` skills
- Docker Compose configuration with Copilot CLI + Node.js for MCP server
- Deploy to Azure Container Apps (dev environment)

### Phase 2: Sprint Reports (Weeks 5–10)

**Milestone:** Conversational sprint review report authoring with PPTX/DOCX output

- Develop `sprint-review` skill with references: report structure, stakeholder communication guidelines, data storytelling best practices, example reports
- Develop `docx` skill with University of Idaho document styling conventions
- Refine `pptx` skill based on Phase 1 spike findings; add chart style references, branded templates
- Multi-turn report authoring workflow: agent gathers data → presents plan → drafts report → user steers → agent revises → finalize
- Velocity trend analysis across sprints via MCP tools
- Report persistence in PostgreSQL (GeneratedReport model, download endpoint)
- Report preview in chat UI (summary of sections/slides before download)
- Chat history persistence in PostgreSQL
- Enable `search` MCP domain for cross-referencing
- Iterate skills based on scrum master feedback on first real sprint reports

### Phase 3: Work Item Creation (Weeks 11–16)

**Milestone:** Conversational requirements decomposition with ADO write access

- Develop `work-item-decomposition` skill with references: hierarchy rules, acceptance criteria guidelines, estimation heuristics
- Enable write operations via ADO MCP server (`create_work_item`, `update_work_item`)
- Hierarchy preview UI component (tree view of proposed items before creation)
- Confirmation flow — agent presents plan, user approves before MCP write calls
- Duplicate detection via `search_work_items` MCP tool
- Batch creation with parent/child link wiring
- Source traceability (original text linked in work item descriptions)

### Phase 4: Sprint Readiness Review (Weeks 17–20)

**Milestone:** AI-assisted sprint backlog review with risk identification and work item updates

- Develop `sprint-readiness` skill with references: review checklist, risk patterns, follow-up action templates
- Read-and-analyze workflow: query all work items in upcoming sprint, evaluate completeness, risks, and dependencies
- Structured findings presentation: categorized by severity (high risk, gaps, solid)
- Agent-proposed follow-up actions with confirmation flow (add acceptance criteria, create tasks, flag dependencies)
- Uses write operations established in Phase 3 (`create_work_item`, `update_work_item`)
- Cross-sprint dependency detection via `search_work_items`

### Phase 5: Test Plans (Weeks 21–26)

**Milestone:** Test plan scaffolding from user stories

- Develop `test-plan` skill with references: test case patterns, edge case inference, shared step extraction
- Enable `test-plans` MCP domain
- Acceptance criteria → test case mapping
- Edge case and negative scenario inference
- Shared step extraction and reuse
- Traceability links between test cases and user stories
- Enable `wiki` MCP domain for publishing reports

### Future Considerations

- Retrospective facilitation: structured retro capture with action item creation (new `retrospective` skill)
- Sprint planning assistance: capacity-based work assignment and load balancing suggestions (extends Phase 4 sprint readiness with quantitative capacity analysis)
- Azure AD SSO integration replacing template's local auth
- Delegated OAuth for per-user ADO attribution
- Cross-project dependency visualization
- Integration with Microsoft Teams for in-channel summaries
- Expose SprintPilot's capabilities as an MCP server itself for use in VS Code / Copilot Chat
- Skills marketplace: share skills across OIT teams for other AI-assisted workflows
- Release notes skill: auto-generate release notes from completed work items and PR descriptions

---

## 10. Risks and Mitigations

| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| **Copilot SDK breaking changes** (Technical Preview) | High | Medium | Pin SDK version, abstract session management behind internal interface, maintain fallback patterns |
| **Copilot CLI must be available in container** | Medium | High | Dockerfile installs CLI; health check verifies startup; document manual install path |
| **ADO MCP server tool limitations** | Medium | Medium | Validate each workflow's tool requirements against MCP server docs during Phase 1; fall back to custom tools for gaps |
| **LLM hallucination in work item content** | Medium | Medium | Human-in-the-loop confirmation for all write operations; source text attribution |
| **Premium request quota exhaustion** | Medium | High | Monitor usage, implement session token budgets; scale to BYOK pay-per-request pricing (OpenAI, Azure AI Foundry, Anthropic) if usage exceeds quota |
| **Service account PAT over-permissioning** | Low | High | Scope PAT to minimum required permissions; rotate regularly; audit via ADO access logs |
| **MCP server process lifecycle in containers** | Medium | Medium | Copilot SDK manages MCP server process; add health monitoring; restart policy in Docker Compose |
| **CopilotClient pool exhaustion at peak load** | Low | Medium | Queue with timeout at pool checkout; monitor pool utilization; increase pool size if wait times exceed threshold |
| **Scope creep into sprint planning automation** | Medium | Low | Explicit non-goals in PRD, phase boundaries, stakeholder alignment |
| **Agentic report quality inconsistency** | Medium | Medium | Skills encode branding and structure constraints; example reports in skill references set quality bar; user steering in chat provides real-time correction; iterate skills based on feedback |
| **Token cost of multi-turn report authoring** | Medium | Medium | Monitor per-session token usage; compare against deterministic baseline from Phase 1 spike; set session budgets; BYOK pay-per-request as scaling path |
| **Skill maintenance overhead** | Low | Low | Skills are markdown + scripts in the repo, reviewed like code; changes are incremental; development skill helps maintain consistency |

---

## 11. Success Metrics

| Metric | Baseline (Manual) | Target (SprintPilot) | Measurement |
|---|---|---|---|
| Sprint report creation time | 2–4 hours | < 30 minutes (including conversational revisions) | Self-reported by scrum masters |
| Post-generation manual edits | Significant (formatting, tone, structure) | Minimal (< 10 min touch-up, if any) | Self-reported; measures skill quality |
| Requirements → ADO work items | 1–2 hours per feature set | < 20 minutes (including review) | Time from paste to confirmed creation |
| Test plan coverage | Ad hoc, inconsistent | Every acceptance criterion has ≥ 1 test case | ADO traceability query |
| Sprint readiness issues caught pre-sprint | Discovered during sprint (disruptive) | ≥ 80% of gaps/risks identified before sprint start | Tracked via items created/updated by readiness review |
| Custom ADO API code maintained | Hundreds of lines per integration | Near-zero (MCP server handles it) | Lines of ADO-specific code in repo |
| User adoption (OIT scrum teams) | N/A | ≥ 3 teams actively using within 3 months of Phase 2 | Session telemetry |
| User satisfaction | N/A | ≥ 4.0/5.0 on internal survey | Quarterly survey |

---

## 12. Open Questions

| # | Question | Owner | Status |
|---|---|---|---|
| 1 | Which Copilot-supported model performs best for work item decomposition (GPT-5 vs. Claude Sonnet 4.5 via BYOK)? | Dev Team | Needs Evaluation |
| 2 | Are there existing sprint report templates that SprintPilot should match, or can we define a new standard? | Scrum Masters | Open |
| 3 | What is the premium request quota allocation for the OIT Copilot subscription? | IT Admin | Open |
| 4 | ~~Should generated files be stored persistently or treated as ephemeral?~~ | Product Owner | **Resolved** — stored in PostgreSQL as `bytea`; see §6.8 |
| 5 | Does the ADO MCP server's `test-plans` domain cover shared step creation, or will we need a custom tool? | Dev Team | Needs Validation |
| 6 | Should the service account PAT be per-project or organization-wide? | IT Admin | Open |
| 7 | Is there appetite to expose SprintPilot tools via MCP for use in VS Code / Copilot Chat directly? | Dev Team | Future |
| 8 | What existing sprint report examples should be included as reference material in the `sprint-review` skill? | Scrum Masters | Open |
| 9 | What branding assets (logo files, hex colors, approved fonts) are available for the `pptx` skill references? | OIT / University Communications | Open |

---

## Appendix A: Glossary

| Term | Definition |
|---|---|
| **ADO** | Azure DevOps |
| **ADO MCP Server** | Microsoft's official MCP server for Azure DevOps (`@azure-devops/mcp`), providing tool-based access to work items, test plans, repos, wiki, and more |
| **ADR** | Architecture Decision Record — a lightweight document capturing the context, options, decision, and consequences of a significant technical choice. Stored in `/docs/decisions/` (see §6.9) |
| **Agent Skill** | A structured directory containing instructions (SKILL.md), reference materials, and helper scripts that teach the Copilot agent how to perform a specific task (e.g., creating PPTX files, authoring sprint reviews) |
| **BYOK** | Bring Your Own Key — Copilot SDK feature allowing use of third-party LLM API keys (OpenAI, Azure AI Foundry, Anthropic) with pay-per-request pricing instead of GitHub's Copilot subscription quota |
| **Copilot CLI** | GitHub Copilot's command-line agent runtime that the SDK communicates with via JSON-RPC |
| **Copilot SDK** | GitHub Copilot SDK (`github-copilot-sdk` for Python) — programmatic access to the Copilot CLI's agentic engine |
| **MCP** | Model Context Protocol — open standard for connecting AI models to external tools and data sources |
| **MCP Domain** | A named group of related tools in the ADO MCP server (e.g., `core`, `work-items`, `test-plans`); used to limit which tools are loaded |
| **OIT** | Office of Information Technology, University of Idaho |
| **PAT** | Personal Access Token — Azure DevOps authentication credential |
| **shadcn/ui** | React component library built on Radix UI and Tailwind CSS; included in the FastAPI template |
| **WIQL** | Work Item Query Language — SQL-like query syntax for Azure DevOps work items |

---

## Appendix B: ADO MCP Server Tool Reference

The following tools from the Azure DevOps MCP Server are expected to be used across SprintPilot's workflows. Exact tool names may vary; validate against the [ADO MCP Server documentation](https://github.com/microsoft/azure-devops-mcp).

**Core Domain:**
`get_me`, `list_organizations`, `list_projects`, `get_project_details`

**Work Domain:**
`get_sprint_details`, team iteration settings, capacity data

**Work-Items Domain:**
`list_work_items`, `get_work_item_details`, `create_work_item`, `update_work_item`, `add_work_item_comment`

**Test-Plans Domain:**
`create_test_plan`, `create_test_suite`, `add_test_cases`, `list_test_plans`

**Search Domain:**
`search_work_items`, `search_code`

**Wiki Domain:**
`get_wiki_page`, `create_wiki_page`, `update_wiki_page`
