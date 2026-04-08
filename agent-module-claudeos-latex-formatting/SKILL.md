---
name: agent-module-latex-formatting
description: "Module: Handle LaTeX formatting, templates, and styling for academic papers. Set up conference templates (ICML, ICLR, NeurIPS, AAAI, ACL), fix formatting issues, manage packages, and ensure venue-specific compliance. Use when the user needs to set up a paper template, fix LaTeX formatting, or prepare for submission."
---
# Module: Latex Formatting

## 1. Module Metadata
- Module ID: `agent.module.latex-formatting`
- Version: `1.0.0`
- Maturity: `migrated`
- Scope: Handle LaTeX formatting, templates, and styling for academic papers. Set up conference templates (ICML, ICLR, NeurIPS, AAAI, ACL), fix formatting issues, manage packages, and ensure venue-specific compliance. Use when the user needs to set up a paper template, fix LaTeX formatting, or prepare for submission.
- Primary outcomes:
  - Facilitates operations for Latex Formatting.

## 2. Mission and Applicability
Use this module when you need to execute tasks related to Latex Formatting.

Apply when:
- Executing the legacy latex-formatting skill workflow.

Do not apply when:
- Functionality falls outside the scope of Handle LaTeX formatting, templates, and styling for academic papers. Set up conference templates (ICML, ICLR, NeurIPS, AAAI, ACL), fix formatting issues, manage packages, and ensure venue-specific compliance. Use when the user needs to set up a paper template, fix LaTeX formatting, or prepare for submission..

## 3. Architecture Pattern
- Pattern: `Legacy Claude Code Skill Execution Pattern`
- Core design rules:
  - This module was automatically migrated from Claude-Research-Paper-OS.
  - Rely on provided shell scripts and reference files in the local directory.

## 4. Implementation Workflow


# LaTeX Formatting

Set up and manage LaTeX formatting for academic papers.

## Input

- `$0` — Action: `setup`, `fix`, `check`
- `$1` — Venue name (for `setup`) or `.tex` file path (for `fix`/`check`)

## Scripts

### Pre-submission format checker
```bash
python /home/lebi/SKILLS/agent-module-claudeos-latex-formatting/scripts/latex_checker.py paper/main.tex --venue neurips --check-anon
```

Checks: word count, required sections, TODO markers, anonymization, mismatched environments, content stats.

### Validate citations and references
```bash
python /home/lebi/SKILLS/agent-module-claudeos-citation-management/scripts/validate_citations.py \
  --tex paper/main.tex --bib paper/references.bib --check-figures
```

### Clean LaTeX text (fix special characters)
```bash
python /home/lebi/SKILLS/agent-module-claudeos-latex-formatting/scripts/clean_latex.py \
  --input paper/main.tex --output paper/main_cleaned.tex
```

Replaces special/non-UTF8 characters with LaTeX equivalents, skips math environments.
Key flags: `--dry-run`, `--tables-only`

### Auto-fix after checking
```bash
python /home/lebi/SKILLS/agent-module-claudeos-latex-formatting/scripts/latex_checker.py paper/main.tex --venue neurips --fix
```

Runs checks then applies clean_latex.py fixes, writing to `main_fixed.tex`.

## References

- Venue specs, project structure, packages, commands: `/home/lebi/SKILLS/agent-module-claudeos-latex-formatting/references/venue-templates.md`

## Action: `setup`
Create the project directory structure and main.tex for the specified venue. Use the template from `references/venue-templates.md`.

## Action: `fix`
Fix common LaTeX issues: unescaped special chars, math mode errors, float placement, overfull/underfull boxes, cross-reference issues.

## Action: `check`
Run `latex_checker.py` for pre-submission validation. Check page count, anonymization, required sections, TODO markers.

## Related Skills
- Upstream: [paper-writing-section](../paper-writing-section/), [table-generation](../table-generation/)
- Downstream: [paper-compilation](../paper-compilation/)
- See also: [citation-management](../citation-management/)


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
- [ ] Determine required inputs for latex-formatting.
- [ ] Execute primary workflows or bundled scripts.
- [ ] Parse Output Format.

## 10. Reuse Notes
Adapt only:
- Command-line arguments for the bundled shell/python scripts.
