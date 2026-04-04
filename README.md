# Development Agent Knowledge Base

## Purpose
This repository is a Markdown-first knowledge base for development agents and engineering teams.

Each skill folder (and its nested `SKILL.md` document) is treated as an executable guidance module, aligned with the agent's internal skill loading format. Modules are designed to help agents and developers consistently produce production-ready systems by combining:
- Operational instructions
- Implementation workflows
- Architecture and decision patterns
- Validation strategies
- Benchmarks and quality bars

## What This System Is
This codebase is a standards library for agent-guided software delivery.

A module should answer the following:
- What problem this guide solves
- When to apply or not apply the guidance
- How to implement the approach step by step
- Which trade-offs and decisions to make
- How to validate correctness, quality, and production readiness

## Repository Organization

### Active Layout
Modules are organized as independent skill folders directly in the root directory (`SKILLS/`). Each module contains a mandatory `SKILL.md` file.

```text
SKILLS/
  README.md
  agent-module-aws-bedrock-python/
    SKILL.md
  agent-module-django-postgres-db-standard/
    SKILL.md
  agent-module-error-taxonomy-and-handling/
    SKILL.md
  agent-module-input-validation-standard/
    SKILL.md
  agent-module-testing-strategy-and-coverage/
    SKILL.md
  agent-module-fullstack-ai-platform-standard/
    SKILL.md
  agent-module-observability-instrumentation/
    SKILL.md
  agent-module-session-auto-naming/
    SKILL.md
```

### Optional Supporting Layout
Add these directories when repository scale or governance requirements justify them:

```text
SKILLS/
  templates/
    module-template.md
  standards/
    naming-conventions.md
    quality-checklist.md
  archive/
    <deprecated-modules>/
      SKILL.md
```

## Document Types
This repository supports these module types:
- Instruction module: prescriptive "what to do"
- Implementation module: procedural "how to build"
- Benchmark module: measurable "how good is good enough"
- Hybrid module: combines all three (preferred)

Default policy: create hybrid modules unless there is a strong reason to isolate scope.

## Naming Conventions
All module folders must follow this pattern:

```text
agent-module-<domain>-<use-case>
```

Naming rules:
- Use lowercase only for the folder name
- Use hyphen-separated words
- Keep names explicit and use-case-first
- Avoid vague suffixes like `guide`, `notes`, `v2`, `misc`
- The instructional file inside the folder **MUST** be named exactly `SKILL.md`.

Examples:
- `agent-module-aws-bedrock-python/SKILL.md`
- `agent-module-django-postgres-db-standard/SKILL.md`
- `agent-module-session-auto-naming/SKILL.md`

## Required Module Structure
Every new module must include all required sections in this order, starting with YAML frontmatter to allow seamless parsing by agent systems:

0. YAML Frontmatter (`name` and `description`)
1. Module Metadata
2. Mission and Applicability
3. Architecture Pattern
4. Implementation Workflow
5. Decision Framework
6. Validation Strategy
7. Benchmarks and SLO Targets
8. Risks and Controls
9. Agent Execution Checklist
10. Reuse Notes

## Authoring Standards

### Formatting Standards
- Use GitHub-flavored Markdown
- Include required YAML frontmatter at the very top of `SKILL.md`
- Use clear heading hierarchy (`#`, `##`, `###`)
- Use short paragraphs and explicit bullet points
- Use numbered steps for workflows
- Use tables for decision matrices and benchmark criteria
- Use code blocks for contracts, schemas, examples, and CLI sequences

### Writing Standards
- Write in precise technical language
- Prefer prescriptive statements over narrative descriptions
- Define scope boundaries explicitly (apply/do-not-apply)
- Include measurable success criteria
- Include operational and failure-mode guidance
- Avoid product-specific assumptions unless the module is intentionally product-scoped

### Reusability Standards
Each module must be:
- Portable across projects
- Deterministic enough for agent execution
- Explicit about preconditions and exit criteria
- Testable through named validation gates

## Required Level of Detail
A module is incomplete if it lacks any of these:
- Correctly formatted YAML frontmatter
- Implementation phases with exit criteria
- Decision matrix with selection rules
- Test/validation guidance for both happy path and failure path
- Quantified benchmarks or SLO targets
- Production readiness checklist

## Standard Template (Mandatory)
Use this template when adding a new module.

```md
---
name: agent-module-<domain>-<use-case>
description: "Module: <Title>"
---
# Module: <Title>

## 1. Module Metadata
- Module ID: `agent.module.<domain>-<use-case>`
- Version: `1.0.0`
- Maturity: `draft|beta|production`
- Scope: <what this module governs>
- Primary outcomes:
  - <outcome 1>
  - <outcome 2>

## 2. Mission and Applicability
Use this module when <conditions>.

Apply when:
- <condition>
- <condition>

Do not apply when:
- <condition>
- <condition>

## 3. Architecture Pattern
- Pattern: `<name>`
- Core design rules:
  - <rule>
  - <rule>

## 4. Implementation Workflow
### Phase A: <name>
1. <step>
2. <step>

Exit criteria:
- <criterion>

### Phase B: <name>
1. <step>
2. <step>

Exit criteria:
- <criterion>

## 5. Decision Framework
| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| <area> | <preferred> | <alternative> | <rule> |

## 6. Validation Strategy
### Functional Validation
- <test>

### Failure Validation
- <test>

### Contract/Security/Performance Validation
- <test>

## 7. Benchmarks and SLO Targets
- <metric>: `<target>`
- <metric>: `<target>`

## 8. Risks and Controls
- Risk: <risk>
  - Control: <control>

## 9. Agent Execution Checklist
- [ ] <item>
- [ ] <item>

## 10. Reuse Notes
Adapt only:
- <dimension>
- <dimension>
```

## Contribution Workflow

### Step 1: Choose Scope
- Confirm no existing module already covers the same use case.

### Step 2: Create Folder and File Using Naming Convention
- Create a directory following the format: `agent-module-<domain>-<use-case>`.
- Inside the directory, create a file named exactly `SKILL.md`.

### Step 3: Populate Mandatory Template
- Include the YAML frontmatter.
- Fill all required sections in order.
- Ensure decision matrix and benchmarks are concrete and measurable.

### Step 4: Quality Review Before Commit
- Confirm language is implementation-grade and unambiguous.
- Confirm workflows include exit criteria.
- Confirm validation covers both positive and negative scenarios.
- Confirm checklist is actionable.

### Step 5: Submit Changes
- Include a concise summary of:
  - Module purpose
  - Key benchmarks introduced
  - Any breaking changes to conventions

## Repository Cleanliness Rules
- Do not place ad-hoc notes in module directories.
- Do not duplicate module scope across multiple folders.
- Keep one primary use case per module.
- Move deprecated modules to a dedicated `archive/` folder when introduced.
- Update this `README.md` when structure or standards change.

## Quality Bar for Acceptance
A new module is accepted only if:
- Directory naming and `SKILL.md` placement are correct
- YAML Frontmatter is present and properly formatted
- Mandatory template sections are complete
- Guidance is reusable and implementation-ready
- Benchmarks are measurable and realistic
- Validation strategy is sufficient for production confidence
