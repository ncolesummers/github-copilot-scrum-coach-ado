# ADR-0006: Skill-Driven Document Generation (Pending Phase 1 Spike Validation)

| | |
|---|---|
| **Status** | Proposed — decision gate at end of Phase 1 |
| **Date** | 2026-02-14 |
| **Author** | OIT Application Development |

## Context

SprintPilot needs to produce branded, professional PPTX and DOCX reports. There are two broad approaches:

1. **Skill-driven generation** — the Copilot agent authors documents iteratively using skill instructions (SKILL.md), reference materials (brand guide, slide layouts, chart styles), and python-pptx/python-docx helper scripts. The agent decides content structure and produces output through multi-turn reasoning, guided by the skill.
2. **Deterministic templates** — fixed PPTX/DOCX masters with defined slide/section layouts. A code layer populates template placeholders with ADO data. The agent narrates; the document structure is fixed.

Key tradeoffs:

| Dimension | Skill-Driven | Deterministic Templates |
|---|---|---|
| Content flexibility | High — agent adapts structure to sprint context | Low — fixed layout regardless of sprint characteristics |
| Stakeholder customization | Conversational ("emphasize blockers", "soften tone") | Requires code changes for layout variation |
| Brand consistency | Relies on skill references; agent could deviate | Guaranteed by template master |
| Token cost | Higher — multi-turn authoring | Lower — single-pass data fill |
| Generation time | Higher | Lower |
| Quality floor | Depends on skill quality and agent capability | Predictable |
| Maintenance | Skill iteration (markdown + scripts) | Template file versioning |

The Copilot agent's ability to produce professional, on-brand documents using skill instructions has not been validated for the University of Idaho's specific requirements. The Phase 1 spike is required to establish this empirically before committing Phase 2 resources to the skill-driven approach.

## Decision

**Proceed with skill-driven generation as the target architecture**, subject to validation in the Phase 1 PPTX spike.

The spike will:
- Adapt existing pptx skill patterns with U of I branding (colors, fonts, logo, approved slide layouts).
- Have the agent generate a representative sprint review slide deck using the skill and python-pptx helpers.
- Evaluate: output quality vs. a manually authored deck, token cost per report, generation time for a 50-item sprint.
- Produce a finding that either **accepts** this ADR (skill-driven meets the quality bar) or **supersedes** it with a deterministic template approach.

If the spike fails the quality bar, a new ADR will be filed superseding this one and documenting the deterministic template design.

## Consequences

**If accepted (spike passes):**
- Phase 2 proceeds with full skill-driven multi-turn report authoring.
- The `/skills` directory is a first-class deliverable iterated alongside code.
- Report quality improves continuously through skill refinement based on user feedback.
- Token cost and generation time must be monitored; BYOK scaling is available if Copilot quota is insufficient.

**If superseded (spike fails):**
- Phase 2 uses deterministic PPTX/DOCX templates with agent narration.
- Skill-driven generation may be revisited in a future phase with improved model capabilities.
- The `/skills` directory still provides workflow expertise (sprint review structure, stakeholder communication) even in the deterministic approach — the document layout is fixed, but the agent's narrative content still benefits from skill references.

**Risks:**
- Skill quality is subjective; "meets the quality bar" must be defined with scrum masters before the spike begins (see Open Question #2 and #8 in PRD §12).
- LLM output for document generation can be inconsistent across runs. The skill's reference materials and branding constraints are the primary quality stabilizers.
