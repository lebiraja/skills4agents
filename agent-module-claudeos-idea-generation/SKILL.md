---
name: agent-module-idea-generation
description: "Module: Generate novel research ideas with iterative refinement and novelty checking against literature. Score ideas on Interestingness, Feasibility, and Novelty. Use when brainstorming research directions or validating idea novelty."
---
# Module: Idea Generation

## 1. Module Metadata
- Module ID: `agent.module.idea-generation`
- Version: `1.0.0`
- Maturity: `migrated`
- Scope: Generate novel research ideas with iterative refinement and novelty checking against literature. Score ideas on Interestingness, Feasibility, and Novelty. Use when brainstorming research directions or validating idea novelty.
- Primary outcomes:
  - Facilitates operations for Idea Generation.

## 2. Mission and Applicability
Use this module when you need to execute tasks related to Idea Generation.

Apply when:
- Executing the legacy idea-generation skill workflow.

Do not apply when:
- Functionality falls outside the scope of Generate novel research ideas with iterative refinement and novelty checking against literature. Score ideas on Interestingness, Feasibility, and Novelty. Use when brainstorming research directions or validating idea novelty..

## 3. Architecture Pattern
- Pattern: `Legacy Claude Code Skill Execution Pattern`
- Core design rules:
  - This module was automatically migrated from Claude-Research-Paper-OS.
  - Rely on provided shell scripts and reference files in the local directory.

## 4. Implementation Workflow


# Idea Generation

Generate and refine novel research ideas with literature-backed novelty assessment.

## Input

- `$0` — Research area, task description, or existing codebase context
- `$1` — Optional: additional context (e.g., "for NeurIPS", constraints)

## Scripts

### Novelty check against Semantic Scholar
```bash
python /home/lebi/SKILLS/agent-module-claudeos-idea-generation/scripts/novelty_check.py \
  --idea "Adaptive attention head pruning via gradient-guided importance" \
  --max-rounds 5
```

Performs iterative literature search to assess if an idea is novel.

## References

- Ideation prompts (generation, reflection, novelty): `/home/lebi/SKILLS/agent-module-claudeos-idea-generation/references/ideation-prompts.md`

## Workflow

### Step 1: Generate Ideas
Given a research area and optional code/paper context:
1. Generate 3-5 diverse research ideas
2. For each idea, provide: Name, Title, Experiment plan, and ratings
3. Use the ideation prompt templates from references

### Step 2: Iterative Refinement (up to 5 rounds per idea)
For each idea:
1. Critically evaluate quality, novelty, and feasibility
2. Refine the idea while preserving its core spirit
3. Stop when converged ("I am done") or max rounds reached

### Step 3: Novelty Assessment
For each promising idea:
1. Run `novelty_check.py` or manually search Semantic Scholar / arXiv
2. Use the novelty checking prompts from references
3. Multi-round search: generate queries, review results, decide
4. Binary decision: Novel / Not Novel with justification

### Step 4: Rank and Select
- Score each idea on three dimensions (1-10): Interestingness, Feasibility, Novelty
- Be cautious and realistic on ratings
- Select the top idea(s) for development

## Output Format

```json
{
  "Name": "adaptive_attention_pruning",
  "Title": "Adaptive Attention Head Pruning via Gradient-Guided Importance Scoring",
  "Experiment": "Detailed implementation plan...",
  "Interestingness": 8,
  "Feasibility": 7,
  "Novelty": 9,
  "novel": true,
  "most_similar_papers": ["paper1", "paper2"]
}
```

## Rules

- Ideas must be feasible with available resources (no requiring new datasets or massive compute)
- Do not overfit ideas to a specific dataset or model — aim for wider significance
- Be a harsh critic for novelty — ensure sufficient contribution for a conference paper
- Each idea should stem from a simple, elegant question or hypothesis
- Always check novelty before committing to an idea

## Related Skills
- Upstream: [literature-search](../literature-search/), [deep-research](../deep-research/)
- Downstream: [research-planning](../research-planning/), [experiment-design](../experiment-design/)
- See also: [novelty-assessment](../novelty-assessment/)


## 5. Decision Framework
| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| Skill Usage | Utilize bundled scripts | Manual execution | Prefer automated scripts if available. |

## 6. Validation Strategy
### Functional Validation
- Ensure bundled scripts execute without runtime errors.
- Validate that the output matches the expected JSON or Markdown schema.

## 7. Benchmarks and SLO Targets
- Execution Latency: `< 30s` for standard local script execution.
- Quality: Output adheres to constraints specified in the prompt references.

## 8. Risks and Controls
- Risk: Bundled scripts rely on missing absolute paths (`/home/lebi/SKILLS/agent-module-claudeos-...`).
  - Control: The paths should be mapped or modified to reflect current working paths.

## 9. Agent Execution Checklist
- [ ] Determine required inputs for idea-generation.
- [ ] Execute primary workflows or bundled scripts.
- [ ] Parse Output Format.

## 10. Reuse Notes
Adapt only:
- Command-line arguments for the bundled shell/python scripts.
