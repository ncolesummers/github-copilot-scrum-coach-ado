# ADR-0007: Work Item Hierarchy — Epic → User Story → Task (No Features)

| | |
|---|---|
| **Status** | Accepted |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

Azure DevOps supports several process templates with different work item type hierarchies. The default Agile and Scrum process templates include a **Feature** level between Epics and User Stories:

```
Epic → Feature → User Story → Task
```

OIT's Azure DevOps organization uses a customized process that **does not include the Feature work item type**. The hierarchy in use is:

```
Epic → User Story → Task
Bugs → linked to Epic or User Story (not a separate hierarchy level)
```

SprintPilot's work item decomposition workflow (Phase 3) and sprint review report generation (Phase 2) must model work items in terms of this actual hierarchy. If SprintPilot's skills or agent prompts assume Features exist, they will produce incorrect hierarchies, fail to create parent/child links correctly, or confuse scrum masters with work item types that don't exist in their environment.

## Decision

All SprintPilot skills, agent prompts, example outputs, and ADO MCP tool invocations will use the **Epic → User Story → Task** hierarchy exclusively. Feature work item types will not be referenced anywhere in SprintPilot.

Bug handling:
- Bugs are created at the Epic or User Story level as appropriate to the context.
- Bugs carry Story Points and are included in velocity calculations alongside User Stories.
- Bugs are never given Task children in this hierarchy.

This decision is codified in:
- `skills/work-item-decomposition/references/hierarchy-rules.md` — the authoritative reference for the agent's decomposition logic.
- `skills/sprint-review/references/report-structure.md` — velocity grouping by Epic, not Feature.
- The `work-item-decomposition` SKILL.md — explicit instruction to never create Feature work items.

## Consequences

**Positive:**
- SprintPilot's output matches OIT's actual ADO structure; no manual cleanup of incorrectly typed work items.
- Velocity calculations are straightforward: sum Story Points on User Story and Bug items in the sprint iteration.
- Simpler decomposition logic — three levels instead of four.

**Negative / Risks:**
- If OIT's ADO process is ever extended to include Features, this decision must be revisited and the affected skills updated.
- New developers working on SprintPilot who are familiar with standard Agile/Scrum process templates may instinctively reach for Features — the `development` skill and this ADR serve as the reference to correct that assumption.
- If SprintPilot is ever extended for use by teams outside OIT that do use Features, a configurable hierarchy would be required.
