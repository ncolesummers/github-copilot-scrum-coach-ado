# Claude Code Agent Teams: Patterns, Team Size, and What to Expect

## Executive Summary

Claude Code “agent teams” are an **experimental** multi-session orchestration feature that lets you run **multiple independent Claude Code instances** (teammates) under a **single coordinating lead**, with a **shared task list** and **inter-agent messaging**. It’s designed for work where **parallel exploration is genuinely additive** (multiple “angles” investigated simultaneously) rather than simply “more hands on the same file.” citeturn3view0turn8view0

The most reliable use cases cluster into a few families:

- **Parallel review and analysis** (e.g., code review split by security/perf/tests) where each teammate applies a distinct lens and the lead synthesizes results. citeturn3view0  
- **Debugging with competing hypotheses** (explicitly adversarial investigation) to counter anchoring on the first plausible root cause. citeturn3view0  
- **Cross-layer work** (frontend/backend/tests), where each teammate owns a bounded surface so context stays clean and merges are predictable. citeturn1view0  
- **New modules/features partitioned by file ownership**, so teammates produce separable artifacts without overwriting each other. citeturn1view0turn13view0  
- **Large-scale autonomous builds only when you have strong harnesses** (tests + CI + guardrails) and accept that coordination and integration become the bottleneck; Anthropic’s own large-scale demo emphasized harness design and high-quality tests as the main driver of sustained progress. citeturn7view0  

The biggest pitfalls repeat across official docs, maintainer issue reports, and practitioner writeups:

- **Same-file edits cause overwrites**; agent teams do not magically “merge safely.” citeturn3view0  
- **Session resumption is imperfect** (notably in-process teammates don’t restore on `/resume` / `/rewind`), and task state can lag, blocking dependent tasks. citeturn3view0  
- **Token costs scale with active teammates**, plus broadcast messaging cost scales with team size, making “chatty swarms” expensive. citeturn3view0turn4view0  
- **Tool/terminal fragility at higher concurrency**, especially in tmux split-pane mode; a maintainer-tracked bug report shows pane spawning races and `send-keys` corruption when dispatching 4+ agents simultaneously in tmux. citeturn26view0  
- **Illusion of progress**: lots of tasks flipping states and lots of messages, but poor integration, weak verification, and rework. This is echoed in both community cautionary notes and “production pain points” reports that highlight crash recovery, file-transaction semantics, and persistent backlog/memory as the blockers at scale. citeturn25view0turn26view1turn27view0  

Practical recommendation (as of February 2026 release state):

- **Adopt now** if you can consistently (a) decompose work by file ownership or by “review lens,” (b) enforce quality gates (tests/lint/typecheck) via hooks, and (c) keep teams small and monitored. The official docs explicitly recommend starting with research/review and emphasize task sizing and steering. citeturn3view0turn21view0turn29view2  
- **Wait (or pilot only)** if your work is dominated by tight architectural coupling, large refactors with many shared files, or environments where tmux/iTerm2 split panes are unreliable—because coordination overhead, merge conflicts, and resumption gaps can erase the win. citeturn1view0turn3view0turn26view0  

Reflective questions to ask before using agent teams:
- *Is this task actually parallelizable, or does it require tight architectural coherence?* citeturn1view0  
- *What is your integration choke point: tests, merge conflicts, permission prompts, or spec ambiguity?* citeturn3view0turn4view0turn26view0  

## How Agent Teams Work

This section is grounded primarily in the official agent teams docs and supporting configuration docs, with cross-validation from practitioners who inspected on-disk coordination artifacts during live runs. citeturn1view0turn5view0turn30view0  

**Core roles**

- **Lead**: the initial Claude Code session that creates the team, spawns teammates, coordinates tasks, and synthesizes results. citeturn3view0  
- **Teammates**: separate, full Claude Code sessions, each with its own context window; they can message each other directly (not just report back to the lead). citeturn1view0turn3view0  
- **Shared task list**: work items with dependency tracking and states (pending / in progress / completed). citeturn3view0  
- **Mailbox / messaging**: a communication channel where teammate messages are delivered automatically to recipients; broadcast exists but should be used sparingly due to cost scaling. citeturn3view0  

**Enablement and configuration**

Agent teams are **disabled by default** and must be enabled via `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in environment or settings. citeturn1view0turn4view0  

Claude Code supports hierarchical settings scopes (user, project, local), so you can enable agent teams globally in `~/.claude/settings.json` or per-repo in `.claude/settings.json`. citeturn5view0turn15view0  

**Display and interaction modes**

Two teammate display modes are supported:

- **In-process**: teammates run “inside” the main terminal; you can cycle/select teammates with keyboard controls and message them directly. citeturn1view0turn3view0  
- **Split panes**: each teammate gets its own pane (tmux or iTerm2). This is powerful for monitoring, but it introduces terminal/tooling dependency and known limitations (certain terminals unsupported; tmux has OS-specific rough edges). citeturn1view0turn3view0turn26view0  

You can set `teammateMode` in settings or override per session using `claude --teammate-mode in-process`. citeturn1view0turn5view0turn24view0  

**Spawning teammates and controlling autonomy**

- A team can start because you explicitly request it, or Claude proposes it—**but Claude won’t create a team without your approval**. citeturn3view0  
- You may specify the number of teammates (“Create a team with 4 teammates…”), or let Claude pick based on the task. citeturn2view3turn3view0  
- For risky work, you can require **plan approval**: the teammate stays in read-only planning until the lead approves its plan; the lead can be instructed to apply acceptance criteria (“only approve plans that include test coverage”). citeturn2view2turn3view0  
- **Delegate mode** restricts the *lead* to coordination-only behaviors, reducing a common failure where the lead starts implementing instead of orchestrating. citeturn2view2turn25view0  

A related (often paired) mechanism is **Plan Mode**, which is a permission mode intended for safe, read-only analysis and planning; you can cycle into it with Shift+Tab, or start a session with `--permission-mode plan`. citeturn23view0  

**Task lifecycle and contention model**

- Tasks have states and dependencies; blocked tasks cannot be claimed until dependencies complete. citeturn3view0  
- The lead can assign tasks explicitly, or teammates can self-claim the next available unblocked task. citeturn3view0  
- Task claiming uses **file locking** to prevent race conditions when multiple teammates try to claim the same task. citeturn3view0  

Practitioner inspections strongly suggest that the “shared task list + mailbox” is implemented as simple JSON files under `~/.claude/teams/<team>` and `~/.claude/tasks/<team>`, with a `.lock` file in the tasks directory—consistent with the official statement that teams and tasks are stored locally at those paths. citeturn3view0turn30view0  

**Shutdown and cleanup**

- The lead can request a teammate shutdown; the teammate may approve (exit) or reject with explanation. citeturn3view0  
- Cleanup removes shared team resources, but will fail if teammates are still running; official guidance is to clean up via the lead (not teammates) to avoid inconsistent state. citeturn3view0  

**Limitations and failure modes called out officially**

Key documented limitations include:

- `/resume` and `/rewind` do **not** restore in-process teammates; after resuming, the lead may attempt to message teammates that no longer exist. citeturn3view0  
- Task status can lag (tasks appear stuck / dependencies blocked), and shutdown can be slow because teammates finish their current request before stopping. citeturn3view0  
- One team per session; no nested teams; lead is fixed and cannot be transferred. citeturn3view0  
- Split-pane mode is not supported in certain terminals (including VS Code integrated terminal and others), and tmux/iTerm2 are required for panes. citeturn3view0  

Maintainership signals: early releases after launch included agent-team-specific fixes (tmux teammate messaging, hook events for multi-agent workflows, and crashes when team settings changed), indicating the feature is evolving quickly. citeturn11view0  

image_group{"layout":"carousel","aspect_ratio":"16:9","query":["Claude Code agent teams tmux split panes","Claude Code agent teams shared task list screenshot","Claude Code agent teams subagents comparison diagram"],"num_per_query":1}

Reflective questions:
- *What parts of “coordination” do you want automated: task queueing, messaging, or merge integration?* citeturn3view0turn30view0  
- *Are you depending on `/resume` for continuity? If yes, can you tolerate manual teammate respawn and task-state repair?* citeturn3view0  

## Pattern Catalog

The patterns below are organized for repeatability: each specifies team size band, role split, guardrails, expected outcome, and failure modes. Patterns marked as “high-confidence” are directly documented in official materials or backed by consistent practitioner reports; others are evidence-supported syntheses (explicitly grounded in the cited failure modes and mechanics). citeturn3view0turn7view0turn30view0turn26view0turn29view2  

| Pattern | Best team size band | When to use it | Setup (roles + task split) | Guardrails | Expected outcome | Failure modes |
|---|---|---|---|---|---|---|
| Lens-based parallel code review (high-confidence) | 2–3 | You need fast, thorough review across different risk dimensions | Lead + 2–3 reviewer teammates (“security”, “performance”, “test coverage”) reviewing same PR with distinct lenses citeturn3view0 | No code edits by reviewers; require evidence (file/line refs); synthesize in lead | Broader review coverage than a single reviewer | Overlap if lenses not distinct; “review spam” without prioritization |
| Competing-hypotheses debugging (high-confidence) | 4–6 | Root cause unclear; you want to avoid anchoring | Lead + 4–5 investigators; each assigned a different hypothesis; require cross-challenge debate citeturn3view0 | Single shared findings doc; limit cross-file edits; require reproduction steps | Faster convergence on plausible root cause | Hypotheses converge prematurely; confusion if no shared artifact |
| Cross-layer feature split (high-confidence) | 2–3 (or 4–6 if larger) | Work naturally partitions by layer or service boundary | Teammates own frontend/backend/tests (or service A/B/C) citeturn1view0turn25view0 | Explicit file ownership; integration checklist; CI gates | Parallel progress with minimal context switching | Boundary mismatch causes integration rework |
| Partitioned refactor by directory ownership (high-confidence) | 4–6 | Similar transformation across multiple modules/packages | One teammate per directory/package; lead owns “integration + final pass”; require plan approval for risky files citeturn2view2turn13view0 | Hard rule: no two teammates edit same file citeturn3view0 | Refactor throughput without constant merge conflicts | Overwrites if boundaries leak; inconsistent style across modules |
| Delegate-only lead orchestration (high-confidence) | 4–10 | You want the lead to stay “PM-like” and avoid coding | Enable delegate mode after team starts; lead only spawns/messages/manages tasks citeturn2view2turn3view0 | Ensure teammates have correct permission mode/tools; review that delegate doesn’t unintentionally constrain productive work (practitioner-reported gotcha) citeturn3view0turn24view0 | Cleaner orchestration; less lead interference | Teammates stall if permissions are restrictive or prompts are vague |
| Plan-approval gated implementation (high-confidence) | 2–6 | Risky module changes (auth, data, security) | Spawn “architect” teammate in plan-approval mode; once approved, spawn implementer(s) citeturn2view2turn23view0 | Define explicit approval criteria (tests, constraints) citeturn2view2 | Reduced rework from wrong initial direction | Approval criteria too vague; plan bikeshedding |
| Hook-gated task completion (high-confidence) | 4–10 | Multiple agents produce code; you need “stop-the-line” gates | Use `TaskCompleted` hook to block completion if tests/lint fail; use `TeammateIdle` to keep teammate working when failing citeturn3view0turn29view2 | Automated checks; keep hook outputs concise; avoid flaky tests | Improved convergence; fewer “greenwashed” completions | Flaky gates cause churn; hooks add local complexity |
| Async test feedback loop (high-confidence) | 2–6 | You want continuous verification without blocking agents | PostToolUse async hook runs tests after writes and reports results next turn citeturn29view0turn29view1 | Timeouts; scope to file types; summarize failures | Faster defect detection; less manual “run tests” | Too-noisy output pollutes context; redundant runs |
| Replacement teammate for error-stalled work (high-confidence) | 4–6 | A teammate stops on errors or gets stuck | Inspect output; either redirect directly or spawn replacement teammate citeturn3view0 | Preserve partial artifact; reassign tasks explicitly | Keeps throughput when one agent derails | Duplicate work; lost context due to new teammate cold start |
| “Consolidator” pass to reduce duplication (evidence-supported) | 7–10 | LLMs tend to re-implement functionality; you need cleanup | Assign one teammate to identify duplication and coalesce patterns; run after feature burst citeturn7view0 | Run after merges stabilize; require tests passing before refactor | Lower long-term entropy and duplicated helpers | Consolidator touches too many files → conflict explosion |
| Dedicated documentation/progress tracker teammate (evidence-supported) | 4–10 | Project needs living docs, changelogging, architectural notes | Assign one teammate to maintain README/progress notes; others implement citeturn7view0 | Keep docs in a defined location; require “update docs” as a task dependency | Better shared context; less repeated orientation | Docs drift from reality if not tied to CI/tasks |
| Multi-artifact parallel build (practitioner example) | 2–3 | Two independent deliverables can be built in parallel | One teammate per deliverable; only share deployment checklist with lead citeturn18view0 | Strict boundary: no shared files; separate acceptance tests | Wall-clock compression for independent workstreams | If deliverables share hidden dependencies, integration drags |
| File-system observability audit (practitioner example) | 2–5 | You need to understand “what the team is doing” and debug coordination | Inspect `~/.claude/teams/<team>` + `~/.claude/tasks/<team>` during a run; verify tasks, owners, inbox messages citeturn30view0turn3view0 | Treat as read-only; do not mutate team files manually unless restoring from stuck state | Faster diagnosis of stuck tasks and misrouting | Manual edits risk corrupting team state; internals may change citeturn30view0 |

**Anti-patterns and “illusion of progress” traps**

- **Same-file parallel implementation**: official guidance is blunt—two teammates editing the same file leads to overwrites. If you can’t draw clean file boundaries, agent teams are often slower than a single focused session. citeturn3view0turn1view0  
- **Oversharding tasks** (“too small”): coordination and messaging overhead dominates when tasks are tiny; tasks should be “self-contained units with a clear deliverable,” and the docs suggest keeping multiple tasks per teammate to maintain productivity. citeturn3view0turn25view0  
- **Oversized tasks** (“too large”): teammates run too long without check-ins, increasing risk of wasted effort and misalignment; official guidance explicitly calls this out. citeturn3view0  
- **Unbounded broadcast + status chatter**: broadcast costs scale with team size, and excessive messaging can create the feeling of momentum while increasing token burn and decreasing focus. citeturn3view0turn4view0  
- **tmux pane “swarm shock”**: dispatching 4+ teammates simultaneously can break pane spawning / command injection in tmux in current builds (a reproducible issue). Mitigation: in-process mode, fewer simultaneous dispatches, or staggered spawns. citeturn26view0turn3view0  
- **Assuming crash recovery and persistent backlog exist out of the box**: both maintainers and practitioners report missing crash recovery, persistent task backlogs across sessions, and cross-session knowledge retention as “blockers at scale.” citeturn26view1turn27view0turn3view0  

Reflective questions:
- *If everyone is “busy,” who is integrating and verifying?* citeturn25view0turn7view0  
- *Can you define “done” as an executable gate (tests/typecheck), not a narrative update?* citeturn29view2turn7view0  

## Team Size Playbook

Claude Code’s own guidance repeatedly emphasizes that agent teams add overhead and token cost, and that they work best when teammates can operate independently. The playbooks below translate that into size-specific operating norms, adding scale-related failure evidence (tmux concurrency bugs, cost scaling, and large-scale autonomous harness lessons). citeturn1view0turn3view0turn4view0turn26view0turn7view0  

**Two to three agents**

Best for:
- Parallel review (security/perf/tests) and small, clearly separable feature slices. citeturn3view0turn1view0  
- “Architect + implementer + critic” exploration where output is a plan or design decision more than code. citeturn1view0turn2view2  

Recommended role mix:
- Lead (orchestrator), 1 implementer or researcher, 1 reviewer/critic. citeturn1view0turn2view2  

Task granularity:
- Aim for tasks that complete in one “agent sitting” and yield a discrete artifact: a review report, a patch to a bounded directory, or a plan with acceptance criteria. citeturn3view0turn23view0  

Coordination topology:
- Star topology is usually enough: lead assigns tasks; teammates report back; limited cross-teammate messaging. Broadcast rarely needed. citeturn3view0  

Integration strategy:
- Lead synthesizes results and performs final integration/merge pass; keep file ownership crisp to avoid overwrites. citeturn3view0turn1view0  

Red flags and mitigations:
- If teammates touch the same file, stop and repartition. citeturn3view0  
- If the lead starts coding instead of orchestrating, use delegate mode and explicitly instruct “wait for teammates.” citeturn3view0turn2view2  

Reflective question: *Do you actually need a team, or would subagents (or a single session) do this with less overhead and fewer conflicts?* citeturn1view0  

**Four to six agents**

Best for:
- Competing-hypothesis debugging (5 agents is the official exemplar). citeturn3view0  
- Medium refactors by directory/service boundary; cross-layer features with testing/QA as a dedicated lane. citeturn1view0turn25view0  

Recommended role mix:
- Lead (coordination), 2–3 implementers partitioned by ownership, 1 reviewer (security/perf lens), 1 QA/test agent. citeturn3view0turn29view0  

Task granularity:
- Official guidance suggests keeping multiple tasks per teammate (the docs cite “5–6 tasks per teammate” as a productivity heuristic), which implies tasks should be small enough to reshuffle when someone stalls, but not so small that orchestration dominates. citeturn3view0turn25view0  

Coordination strategy:
- Use the **shared task list** as the single source of truth; let teammates self-claim only if tasks are well-scoped and non-overlapping. citeturn3view0turn30view0  

Integration strategy:
- Introduce explicit “integration tasks” owned by one agent (often the lead) to keep merges from becoming everyone’s job. This is an inference from the combination of overwrite risk and the need to monitor/steer. citeturn3view0turn7view0  

Operational hazards unique to this band:
- **tmux concurrency**: 4+ simultaneous teammate spawns can corrupt tmux `send-keys` and crash agents (as reported in an issue with repro steps). Mitigate via in-process mode, fewer simultaneous spawns, or staggered dispatch. citeturn26view0turn3view0  
- **Permission prompt storms**: permission requests bubble up to the lead; pre-approving common operations reduces friction. citeturn3view0turn24view0  

Reflective question: *What is your “merge conflict budget”? If merges become frequent, are you willing to re-scope to file ownership?* citeturn3view0turn7view0  

**Seven to ten agents**

Best for:
- Large, partitionable initiatives where each teammate owns a submodule, plus dedicated review/QA/docs roles. This is where “team lead as orchestrator” starts to resemble real engineering management (strong briefs, strong gates). citeturn25view0turn3view0turn7view0  

Recommended role mix:
- Lead (delegate mode often helpful), 4–6 implementers with strict ownership, 1–2 reviewers, 1 QA/test runner, 1 documentation/progress agent. citeturn2view2turn7view0turn29view0  

Task granularity guidance:
- Prefer tasks that are large enough to amortize overhead but bounded enough to be verified quickly. The official “too small / too large” guidance becomes more important as the team grows. citeturn3view0  

Coordination topology:
- Strongly favor a **hub-and-spoke** model: teammates primarily communicate with the lead or through shared artifacts (docs, task list). Mesh chatter scales token cost and increases misalignment. This is an inference anchored in broadcast cost scaling and “monitor and steer” guidance. citeturn3view0turn4view0  

Integration strategy:
- Use hook-based quality gates so “task complete” implies “passes checks,” not “agent said it’s done.” In this band, hook-gated completion often becomes mandatory for sanity. citeturn29view2turn3view0  

Red flags and mitigations:
- Rising “task status lag” and blocked dependencies; be prepared to manually correct status or nudge teammates. citeturn3view0  
- Rising duplication (“two agents implemented similar helpers”): add a consolidator pass once per iteration. citeturn7view0  

Reflective question: *Have you formalized your acceptance tests enough that a teammate can verify its own work without you reading everything?* citeturn7view0turn29view0  

**Ten-plus agents**

This band is where you should assume you are doing **research-preview operations**, not a normal dev workflow—even if the UI lets you spawn that many teammates. Evidence from large-scale autonomous work suggests the bottlenecks shift from “writing code” to “orientation, test harness quality, merge conflicts, and task partitioning.” citeturn7view0turn4view0turn27view0  

Best for:
- Massive, test-driven bug backlogs where each agent can attack an independent failing test or independent module, and you have strong CI gates. citeturn7view0  
- “Swarm-scale” experiments where the goal is throughput exploration and you accept heavy spend. citeturn7view0  

Recommended role mix:
- One orchestrator-like lead, a small number of “high-leverage” reviewers/critics, and many implementers tightly constrained by ownership and tests. The need for specialized roles is supported by Anthropic’s own multi-role approach in its 16-agent experiment (dedicated duplication remover, perf agent, design critic, docs agent). citeturn7view0  

Coordination strategy:
- Do not rely on high-frequency interactive messaging. Prefer task-driven coordination with minimal cross-talk. This is consistent with file-based mailbox delivery semantics (messages queue until a turn ends) and cost scaling. citeturn30view0turn4view0  

Integration strategy:
- Treat CI as the “real orchestrator.” Without near-perfect verifiers, autonomous agents solve the wrong problem; Anthropic explicitly describes building stricter CI enforcement when regression became frequent. citeturn7view0  

Diminishing returns and failure modes:
- When the work collapses into a single coupled bottleneck, adding agents stops helping. In Anthropic’s 16-agent experiment, compiling the Linux kernel initially became “one giant task,” causing agents to overwrite each other until the harness was redesigned to restore parallelism. citeturn7view0  
- Practitioners reporting production-scale coordination describe missing crash recovery, persistent backlog, and cross-session knowledge retention as blockers, requiring custom coordination layers beyond default agent-team behavior. citeturn26view1turn27view0  

Reflective question: *Are you scaling the team because the task is parallelizable—or because it “feels” faster to watch many agents work?* citeturn25view0turn4view0  

## Operator Templates

These templates are designed around three realities in Claude Code agent teams:

- Teammates load project context (e.g., CLAUDE.md, skills, MCP servers) but **do not inherit the lead’s conversation history**, so briefs must be explicit. citeturn3view0  
- Tasks work best when scoped and verifiable; same-file contention is a top failure mode. citeturn3view0  
- Quality gates can be enforced with hooks (including blocking completion and preventing idle). citeturn29view2turn3view0  

### Team brief prompt

```text
Create an agent team for this goal:

GOAL
- <one-sentence outcome>

CONTEXT
- Repo/module(s): <paths>
- Constraints: <performance/security/compat constraints>
- Non-goals: <explicitly exclude tempting scope>

TEAM
- Teammate A: <role> focusing on <area>, owns files: <paths/globs>
- Teammate B: <role> focusing on <area>, owns files: <paths/globs>
- Teammate C: <role> focusing on <area>, owns files: <paths/globs>

COORDINATION RULES
- No two teammates edit the same file.
- Every task must include: definition of done + commands to run + expected outputs.
- Prefer Plan Mode for design decisions; require plan approval for risky modules.
- If a teammate gets stuck >2 iterations, escalate to the lead with a concrete blocker summary.

DELIVERABLES
- <artifact list: PR, docs, tests, benchmark results, etc>

QUALITY GATES
- Must pass: <tests>, <lint>, <typecheck>
- Add/update tests for changed behavior.
```

### Role cards

```text
ROLE: Architect
MISSION: Produce a change plan that is safe, minimal, and verifiable.
METHOD:
- Explore relevant code paths read-only first.
- Propose a plan with steps, impacted files, risks, and acceptance tests.
OUTPUT:
- A numbered plan + a checklist of tests to run + rollback strategy.
```

```text
ROLE: Implementer
MISSION: Implement assigned tasks exactly within owned files.
RULES:
- Do not edit files outside your ownership list without asking.
- Keep changes small and run the provided checks.
OUTPUT:
- Code changes + updated tests + short summary + any follow-up tasks created.
```

```text
ROLE: Reviewer
MISSION: Review for one lens only (pick one): security OR performance OR correctness OR maintainability.
RULES:
- No code edits unless explicitly requested; produce actionable findings with evidence.
OUTPUT:
- Prioritized findings (P0/P1/P2), file:line references, and suggested fixes.
```

```text
ROLE: QA / Test
MISSION: Translate “done” into executable verification.
METHOD:
- Identify regression tests and add missing ones.
- Run test suite; minimize output noise; summarize failures crisply.
OUTPUT:
- New/updated tests + failing/passing evidence + reproduction steps when failing.
```

```text
ROLE: Critic
MISSION: Actively disprove assumptions and surface hidden coupling.
METHOD:
- Ask “what breaks?” and “what’s the rollback cost?”
- Propose adversarial scenarios and edge cases.
OUTPUT:
- A risk register + “don’t ship until” checks.
```

### Task writing template

```text
TASK TITLE: <verb + object>

WHY
- <1–2 sentences>

SCOPE
- Allowed files: <paths/globs>
- Disallowed files: <paths/globs>
- Dependencies: <task IDs>

DEFINITION OF DONE
- Behavior: <observable behavior change>
- Tests: <what tests must pass / what new tests added>
- Performance: <budget or “no regression”>

COMMANDS ALLOWED
- <exact commands> (e.g., npm test, cargo test, go test ./...)

ACCEPTANCE TESTS
- Step 1: ...
- Expected: ...
- Step 2: ...
- Expected: ...

REPORT BACK
- Summary of changes
- Any risks / follow-ups
- Link to diff or files touched
```

### Daily or iteration loop for the lead

```text
Iteration kickoff
- Restate goal + constraints.
- Confirm file ownership boundaries.
- Ensure tasks are small enough and have clear DoD.

During execution (every N minutes or after each task completes)
- Check task board: what’s blocked, what’s stuck, what’s done.
- Ping only stalled teammates; avoid broadcast unless necessary.
- If lead starts implementing, switch to delegate mode and re-focus on orchestration.

Convergence
- Run full quality gates.
- Decide: ship / iterate / rollback.
- Clean up team when done.
```

Reflective questions:
- *Can each task be verified by an automated gate, not a narrative summary?* citeturn29view2turn7view0  
- *Does every teammate know exactly which files they own and what they must not touch?* citeturn3view0turn13view0  

## Cost, Throughput, and Reliability Expectations

**Token and cost scaling model**

Official guidance is consistent on the fundamental scaling law: **each teammate runs as its own Claude instance with its own context window**, so token usage scales with the number of active teammates and how long each runs. citeturn4view0turn3view0  

Practical implications:

- **Linear-ish scaling (baseline)**: adding a teammate roughly adds another “session’s worth” of context processing and tool calls. citeturn4view0  
- **Superlinear pressures (integration)**: merge conflicts, redundant exploration, and cross-agent misalignment tend to increase with team size—especially when boundaries aren’t strict. This is supported by both (a) official warnings about same-file overwrite and (b) large-scale autonomous merges being “frequent.” citeturn3view0turn7view0  
- **Messaging amplification**: broadcast scales cost with team size; mailbox-based messaging is powerful but should be used sparingly. citeturn3view0  

Claude Code’s cost docs recommend concrete cost controls for teams: use a cheaper model for teammates (often Sonnet), keep teams small, keep spawn prompts focused, and clean up teams when done. citeturn4view0  

The cost docs also provide two useful calibration points:
- Average Claude Code cost is reported as **about $6 per developer per day**, with daily costs below $12 for 90% of users (not specific to agent teams, but a baseline). citeturn4view0  
- Agent teams can be **~7× more tokens than standard sessions** when teammates run in plan mode (important as a worst-case multiplier when you overuse parallel planning). citeturn4view0  

**Where diminishing returns tend to begin**

Evidence suggests diminishing returns usually start when one of these becomes dominant:

- **Shared-file contention** (overwrites, merge conflicts): explicitly warned in the docs, and observable in large-team autonomous work when tasks collapse into one coupled bottleneck. citeturn3view0turn7view0  
- **Tooling/terminal orchestration limits** (tmux pane spawning races at 4+ simultaneous dispatches). citeturn26view0  
- **Verification bottlenecks**: when tests are slow, flaky, or unclear, you can generate changes faster than you can validate them, creating rework debt. Anthropic’s experiment explicitly escalated to stricter CI to stop regressions. citeturn7view0turn29view0  
- **Cross-session amnesia / lack of persistent backlog**: maintainers and practitioners highlight missing crash recovery and knowledge retention as blockers for production-scale multi-agent loops. citeturn26view1turn27view0  

**Metrics practitioners use**

Claude Code offers built-in observability mechanisms that teams tie to success criteria:

- `/cost` provides token cost and timing details (useful for “cost per PR” or “cost per merged change” approximations). citeturn4view0  
- Practitioners additionally measure operational coordination through the on-disk task board and inboxes for “who is doing what” and stuck-state diagnosis. citeturn30view0  

A practical metric set (recommended, evidence-aligned) is:

- **Throughput**: tasks completed per hour, PRs merged per day (but treat as a vanity metric unless paired with quality gates). citeturn25view0turn7view0  
- **Rework rate**: number of iterations to converge; how often a completed task is reopened due to test failure or integration break. citeturn7view0turn3view0  
- **Merge conflict frequency**: especially when splitting by ownership fails. citeturn7view0turn3view0  
- **Cost per merged PR / per passing CI**: use `/cost` + CI outcomes. citeturn4view0turn7view0  
- **Stall metrics**: permission prompts per hour, teammate idle-without-deliverable events, and task status lag incidents. citeturn3view0turn24view0  

**Quality gates that improve reliability**

Hooks are the most concrete mechanism available for enforcing multi-agent correctness:

- `TaskCompleted` can be blocked (exit code 2) to prevent a task from being marked complete. citeturn29view2  
- `TeammateIdle` can be blocked to keep a teammate working and feed it feedback. citeturn29view2  
- Async hooks can run tests after file writes and report results back in the next turn, reducing the “verify later” trap. citeturn29view0turn29view1  

**Budgeting tips and stop conditions**

Based on official cost controls plus observed tooling fragility at higher concurrency:

- Cap initial teams to **2–3** until you have stable gates and clear ownership, then scale to 4–6. citeturn4view0turn3view0turn26view0  
- Define an **early stop condition**: if merge conflicts spike, if CI becomes the bottleneck, or if `/cost` rises without proportional progress, revert to a smaller team or single-session mode. citeturn4view0turn7view0turn25view0  
- Treat “10+” as an experiment: without persistent memory/backlog/crash recovery, you will pay for repeated re-orientation and duplicated work. citeturn27view0turn26view1turn7view0  

Reflective questions:
- *If you doubled the team size, would you halve wall-clock time—or just double token burn?* citeturn4view0turn3view0  
- *Is CI fast and deterministic enough to be your primary orchestrator?* citeturn7view0turn29view0  

## When Agent Teams Beat a Single Agent

Official guidance frames agent teams as best when **parallel exploration adds real value** and teammates can work independently; for sequential tasks, same-file edits, or work with many dependencies, single session or subagents are more effective. citeturn1view0turn3view0  

A pragmatic decision rule (evidence-aligned):

Agent teams win when:
- You can partition by **lens** (review) or **ownership boundary** (feature/refactor) and define verifiable outputs. citeturn3view0turn1view0  
- You expect “anchoring risk” in debugging and benefit from explicit competing hypotheses. citeturn3view0  
- You can enforce quality gates so parallelism doesn’t just accelerate wrong code. citeturn29view2turn7view0  

Single-agent (or single-agent + subagents) often wins when:
- The task is inherently sequential or tightly coupled across shared files. citeturn1view0turn3view0  
- You rely heavily on session continuity and `/resume` restoring full state (agent teams currently have resumption limitations). citeturn3view0  
- You need low token cost for routine edits and can’t justify parallel-session overhead. citeturn4view0turn3view0  

One additional official alternative is manual parallelization via git worktrees (multiple sessions without automated team coordination), which can be preferable when you want parallel work but need tighter human control over integration. citeturn3view0