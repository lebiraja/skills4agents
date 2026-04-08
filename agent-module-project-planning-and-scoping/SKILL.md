---
name: agent-module-project-planning-and-scoping
description: "Module: Agent-Led Project Planning, Scoping, and Execution Structuring"
---

# Module: Agent-Led Project Planning, Scoping, and Execution Structuring

## 1. Module Metadata

- Module ID: `agent.module.project-planning-scoping`
- Version: `1.0.0`
- Maturity: `production`
- Scope: End-to-end agent-guided strategic planning, risk-aware work decomposition, dependency mapping, and execution phase structuring for any project type (full-stack systems, features, refactors, data pipelines, etc.)
- Primary outcomes:
  - Shared understanding between agent and human on problem, constraints, and scope
  - Risk inventory and assumption mapping before execution
  - Deterministic execution plan organized by skill modules and dependency ordering
  - Clear phase boundaries with exit criteria and validation gates
  - Early escalation of scope creep or risk changes

---

## 2. Mission and Applicability

Use this module when an agent is tasked with planning *any* project, whether it's a new feature, full-stack system, refactor, migration, or incident response.

Apply when:
- Agent needs to partner with a human to lock down strategy before execution
- Work is complex enough to have multiple phases, dependencies, or risk vectors
- Clarity on scope, constraints, and success is needed before diving into implementation
- Work will consume multiple skill modules (database, testing, observability, etc.)

Do not apply directly when:
- Task is single-function, single-file, or trivial (< 30 mins of work)
- Scope and approach are already locked in writing (pre-approved spec exists)
- Human is asking only for code review or small edits (use focused skills instead)

---

## 3. Agent-Planning Architecture Pattern

**Pattern: Collaborative Discovery → Risk-Aware Execution Planning → Monitored Checkpoint Delivery**

**Core design rules:**
- Agent and human co-author the strategy; neither owns planning alone
- All assumptions are made explicit and documented before work starts
- Work is decomposed into phases; each phase has clear exit criteria and validation gates
- Phases are organized by skill-module dependencies (e.g., input validation before database design)
- Highest-risk/highest-uncertainty work is de-risked first, within dependency constraints
- Parallelizable work is identified and visualized; sequential bottlenecks are flagged
- Each phase conclusion includes a rollback/abort plan in case risks materialize

**Three-Stage Planning Flow:**

```
Stage 1: Collaborative Discovery (Human + Agent)
    ├─ Problem Definition
    ├─ Constraints & Trade-offs
    └─ Scope Boundary
         ↓
Stage 2: Execution Planning (Agent-Driven)
    ├─ Risk Inventory & Ranking
    ├─ Dependency Graph Analysis
    └─ Skill-Module Phase Organization
         ↓
Stage 3: Execution with Checkpoints (Agent-Monitored)
    ├─ Phase Execution (with monitoring)
    ├─ Exit Criteria Validation
    ├─ Risk Escalation (if new risks emerge)
    └─ Checkpoint Review & Rollback Plan
```

---

## 4. Implementation Workflow

### Phase A: Collaborative Discovery (Problem → Constraints → Scope)

**Sub-phase A1: Problem Definition**

1. Agent asks: *What problem are we solving?* (open-ended, let human narrate)
2. Agent clarifies: *Who are the primary users/stakeholders affected by this?*
3. Agent probes: *What does success look like? How will we measure it?* (gather 3–5 measurable criteria)
4. Agent documents: Narrative problem statement + stakeholders + success metrics
5. Agent confirms: *Have I understood the core problem correctly?*

Exit criteria:
- Problem statement is one paragraph, jargon-free, and actionable
- Primary users/stakeholders are named with their goals
- Success metrics are quantifiable (e.g., "latency < 200ms", "adoption > 50%", "error rate < 0.1%")
- Agent and human agree on what "done" means

**Sub-phase A2: Constraints & Trade-offs**

1. Agent asks: *What are your hard constraints?* (timeline, budget, team size, tech stack, compliance, regulatory)
2. Agent asks: *What trade-offs matter most to you?* (quality vs. speed, scope vs. polish, cost vs. capability, risk vs. simplicity)
3. Agent maps trade-offs: *If timeline is tight, we prioritize core features over edge cases; if budget is constrained, we use open-source over managed services*
4. Agent documents: Constraint table + trade-off priority order
5. Agent confirms: *If we're forced to choose between these, should we prioritize X over Y?*

Exit criteria:
- All hard constraints (timeline, budget, team, tech) are documented and quantified
- Trade-off hierarchy is explicit (what matters most when conflicts arise)
- Team composition and availability is mapped (who is working on this, when)
- Regulatory/compliance constraints are explicitly listed

**Sub-phase A3: Scope Boundary**

1. Agent proposes: *Based on problem, constraints, and success criteria, here's what I think is in-scope vs. out-of-scope* (provide concrete lists)
2. Human adjusts: *I need to add X, remove Y, reprioritize Z*
3. Agent identifies: *These items could expand scope or create new dependencies:* (risk flags)
4. Agent documents: Feature list (in-scope + out-of-scope) + scope assumptions + scope-creep risks
5. Agent confirms: *Are we aligned on what's in the box and what's out?*

Exit criteria:
- In-scope features are named and briefly described
- Out-of-scope items are explicitly listed (clarifies what won't be done)
- Scope assumptions are documented (what could change this boundary?)
- Scope-creep risk flags are identified (e.g., "stakeholder X might ask for Y mid-project")

**Deliverable:** Discovery Document
```
# Project Discovery: [Project Name]

## Problem Definition
[One paragraph problem statement]

**Primary Stakeholders:**
- [Name/Role]: [Goal/Need]

**Success Metrics:**
- [Metric 1]: [Target]
- [Metric 2]: [Target]

## Constraints & Trade-offs
| Constraint | Value | Impact |
|---|---|---|
| Timeline | [Date] | [What gets cut if slipped] |
| Budget | [Amount] | [Scope implications] |
| Team | [Size/Availability] | [Parallelization limits] |

**Trade-off Priority:** [If forced to choose, we prioritize...]

## Scope Boundary
**In-Scope:** [Feature 1, Feature 2, Feature 3]
**Out-of-Scope:** [Feature A, Feature B, Feature C]
**Assumptions:** [What could change this boundary?]
```

---

### Phase B: Execution Planning (Risk-Aware Dependency Structuring)

**Sub-phase B1: Risk Inventory & Ranking**

1. Agent identifies: *Based on the problem and constraints, what are the highest-risk areas?*
   - Technical risks (unproven approaches, new tech, integration complexity)
   - Resource risks (team availability, skill gaps, external dependencies)
   - Schedule risks (tight timeline, parallel work, handoff complexity)
   - Scope risks (unclear requirements, scope creep, stakeholder misalignment)
2. Agent ranks: Risk matrix (likelihood × impact)
3. Agent flags: *These are our biggest unknowns; we should de-risk these first*

Exit criteria:
- Risk inventory has 5–10 ranked items with concrete mitigation
- De-risk strategy is explicit (e.g., "spike on auth architecture before building BFF")
- Escalation triggers are defined (e.g., "if timeline slips by 1 week, we cut feature X")

**Sub-phase B2: Dependency Graph Analysis**

1. Agent maps: All major work items as nodes; dependencies as edges (A must complete before B)
2. Agent identifies: Parallelizable clusters (independent work that can run concurrently)
3. Agent identifies: Critical path (longest dependent chain; this determines minimum project duration)
4. Agent visualizes: ASCII or text dependency tree (shows what's parallel vs. sequential)

Exit criteria:
- Dependency graph is documented (text or diagram)
- Critical path is identified and highlighted
- Parallelizable work is clustered

**Sub-phase B3: Skill-Module Phase Organization**

1. Agent assigns: Each major work chunk to a SKILLS module (e.g., Input Validation, RAG System, Testing Strategy, Error Handling)
2. Agent orders: Phases follow dependency ordering + skill prerequisites
3. Agent creates: Execution phase structure with explicit handoffs

Example structure:
```
Phase 1 (De-Risk & Foundation)
├─ Spike: Architecture + tech decisions
├─ Skill: Input Validation Standard (define contracts)
└─ Exit: API contracts are locked, team aligned on tech stack

Phase 2 (Core Backend)
├─ Skill: Django + PostgreSQL Standard
├─ Skill: Error Taxonomy & Handling
└─ Exit: Core API endpoints tested, error handling consistent

Phase 3 (Frontend & BFF)
├─ Skill: Frontend UI (Shadcn + Magic UI)
├─ Skill: Full-Stack AI Platform (if AI involved)
└─ Exit: UI implemented, BFF routes authenticated

Phase 4 (Observability & Hardening)
├─ Skill: Observability & Instrumentation
├─ Skill: Testing Strategy & Coverage
└─ Exit: Logging/metrics in place, test coverage > 80%

Phase 5 (Launch Readiness)
├─ Deploy to staging
├─ Run validation suite
├─ Execute runbooks
└─ Exit: Production-ready, rollback plan in place
```

Exit criteria:
- Each phase is assigned 1–2 relevant SKILLS modules
- Phases respect dependency order
- Parallelizable phases are grouped
- Each phase has clear exit criteria

**Deliverable:** Execution Plan
```
# Execution Plan: [Project Name]

## Risk Inventory (Ranked by Impact × Likelihood)
| Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|
| [Risk 1] | High | High | [Spike, proto, expert review] |

## Dependency Graph
[ASCII or narrative: what depends on what]

## Critical Path
[Longest chain; this determines project duration]

## Execution Phases

### Phase 1: [Name]
**Skills Applied:** [Module 1, Module 2]
**Dependencies:** [What must be done first]
**Deliverables:** [Concrete outputs]
**Exit Criteria:** [Definition of done for this phase]
**Rollback Plan:** [If this phase fails, what do we do?]

[Repeat for each phase]

## Checkpoint Gates
- **After Phase 1:** Risk review + stakeholder alignment
- **After Phase 2:** Architecture validation + security audit
- **[etc.]**
```

---

### Phase C: Execution with Checkpoints (Monitoring & Risk Escalation)

During execution, agent monitors:

1. **Phase Progress:** Are we meeting exit criteria?
2. **New Risk Emergence:** Are unexpected blockers appearing?
3. **Scope Creep:** Are new requirements sneaking in?
4. **Timeline Health:** Are we on track against critical path?

At each phase boundary:

1. Agent validates: *All exit criteria met?*
2. Agent assesses: *Have risks changed? New risks emerged?*
3. Agent escalates: *If risk status changed, notify human immediately*
4. Agent documents: *What we learned, what changed, rollback triggers*

Exit criteria:
- All phase exit criteria are met before advancing
- New risks are flagged and triaged before moving forward
- Rollback triggers are documented and monitored

---

## 5. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Planning depth | Collaborative (both voices) | Agent-only or human-only | Both perspectives prevent blind spots and build buy-in. |
| Risk ranking method | Impact × Likelihood matrix | Gut feel / intuition | Matrix is repeatable and helps prioritize mitigation. |
| Work decomposition | Skill-module aligned + dependency ordering | Linear sequential phases | Skill alignment reuses existing guidance; dependency ordering enables parallelization. |
| Execution monitoring | Checkpoint gates after each phase | Continuous free-form | Gates create decision moments; prevent silent scope creep. |
| Scope change process | Explicit re-plan (risk/timeline impact) | Ad-hoc "just add it" | Explicit process clarifies trade-offs and prevents chaos. |

---

## 6. Validation Strategy

### Discovery Validation

- [ ] Problem statement is one paragraph, clear, and actionable
- [ ] Success metrics are quantifiable, not aspirational
- [ ] Constraints are listed and prioritized; trade-offs are explicit
- [ ] Scope boundary is agreed by agent and human
- [ ] Human has signed off: *I agree with this understanding*

### Planning Validation

- [ ] Risk inventory is ranked (top 3 risks are clear)
- [ ] Dependency graph shows parallelizable work
- [ ] Each phase is assigned 1–2 relevant SKILLS modules
- [ ] Exit criteria for each phase are concrete and measurable
- [ ] Agent and human agree: *This plan is executable within constraints*

### Execution Validation

- [ ] Phase exit criteria are met before advancing
- [ ] New risks are surfaced and escalated within 24 hours of discovery
- [ ] Scope change requests are triaged (yes/no/defer with trade-off explanation)
- [ ] Checkpoints occur at phase boundaries; no silent handoffs

### Failure-Mode Validation

- [ ] If a phase fails: Rollback plan is executed; root cause is documented
- [ ] If timeline slips: Trade-off hierarchy is consulted; scope is adjusted
- [ ] If new risks emerge: Plan is revisited; escalation happens (not silent mitigation)

---

## 7. Benchmarks and SLO Targets

| Metric | Target | Rationale |
|---|---|---|
| **Discovery completion** | < 1 week for most projects | Prevents analysis paralysis; lock strategy early |
| **Planning completion** | < 3 days after discovery | Execution plan should be ready before first line of code |
| **Phase duration** | 1–2 weeks per phase (typical) | Longer phases are risky; shorter checkpoints catch issues faster |
| **Risk escalation time** | < 24 hours of discovery | New risks should surface quickly; silent problems are worst |
| **Scope change decision time** | < 48 hours of request | Delays create chaos; explicit yes/no/defer is better than indecision |
| **Phase success rate** | >= 90% phases meet exit criteria on first attempt | If lower, planning accuracy is poor; re-examine up-front work |
| **Rollback execution time** | <= 4 hours per phase | Rollback should be faster than continuing with broken work |

---

## 8. Risks and Controls

| Risk | Likelihood | Impact | Control |
|---|---|---|---|
| Analysis paralysis (discovery too long) | High | Medium | Set hard deadline for discovery (1 week max); use discovery doc template to keep focused |
| Scope creep (new features sneak in mid-project) | High | High | Explicit scope boundary in Phase A; checkpoint gates that require scope-change triage; escalate immediately |
| Hidden dependencies (phases blocked on undiscovered work) | Medium | High | Dependency graph must be explicit; agent asks "what could block this" for each phase |
| Risk blindness (risks not identified upfront) | Medium | High | Risk inventory should have 5–10 items; if fewer, agent probes deeper; escalate if major risk appears mid-project |
| Timeline slip (constant pushing of deadlines) | High | Medium | Critical path is explicit; escalation triggers are pre-agreed (if slip > 1 week, scope is cut); human is consulted before adjustment |
| Misalignment (agent and human disagree on plan) | Medium | Medium | Collaborative discovery (both voices); explicit sign-off on discovery doc; checkpoint reviews before advancing |

---

## 9. Agent Execution Checklist

### Discovery Phase
- [ ] Problem statement is documented and one paragraph
- [ ] Primary stakeholders and their goals are named
- [ ] 3–5 success metrics are defined and quantifiable
- [ ] All hard constraints (timeline, budget, team, tech, compliance) are listed
- [ ] Trade-off hierarchy is explicit and prioritized
- [ ] Scope boundary is documented (in-scope, out-of-scope, assumptions)
- [ ] Human has reviewed and approved the discovery document
- [ ] Scope-creep risks are identified and flagged

### Planning Phase
- [ ] Risk inventory has 5–10 ranked items with mitigation strategies
- [ ] Dependency graph shows what depends on what; critical path is identified
- [ ] Parallelizable work is clustered; sequential bottlenecks are flagged
- [ ] Each phase is assigned 1–2 relevant SKILLS modules
- [ ] Phases respect dependency order and skill prerequisites
- [ ] Each phase has explicit exit criteria (measurable, not vague)
- [ ] Rollback plan exists for each phase
- [ ] Execution plan is documented and human has reviewed
- [ ] Human has confirmed: *This plan is executable within constraints*

### Execution Phase
- [ ] Phase exit criteria are validated before advancing
- [ ] New risks are discovered and escalated within 24 hours
- [ ] Scope change requests are triaged (yes/no/defer with trade-off explanation)
- [ ] Checkpoints occur at phase boundaries
- [ ] Phase success rate is tracked (>= 90% should meet exit criteria on first attempt)
- [ ] Rollback is executed if a phase fails; root cause is documented
- [ ] Escalation triggers are monitored (timeline slip, budget burn, resource unavailability)

---

## 10. Reuse Notes

This module is foundational for all projects. Adapt:
- **Discovery depth** — Simple features may skip sub-phase A3 (scope is obvious); complex systems should expand each sub-phase
- **Risk inventory size** — Small projects: 3–5 risks; large projects: 10–15 risks
- **Phase duration** — Adjust based on project size (1-week phases for small projects, 2–4 week phases for large systems)
- **Checkpoint frequency** — Weekly for high-risk projects; bi-weekly for stable projects
- **SKILLS module assignments** — Substitute domain-specific modules based on the actual work (this template shows examples; your project will have different modules)

**Do NOT adapt:**
- The three-stage structure (Discovery → Planning → Execution)
- The requirement for explicit scope boundary and trade-off hierarchy
- Checkpoint gates at phase boundaries
- Risk escalation within 24 hours
- Explicit rollback plans for each phase

These are non-negotiable patterns that prevent silent failures and scope chaos.

---
