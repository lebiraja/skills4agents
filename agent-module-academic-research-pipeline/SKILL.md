---
name: agent-module-academic-research-pipeline
description: "Module: Autonomous Academic Research and Paper Production Pipeline"
---
# Module: Autonomous Academic Research and Paper Production Pipeline

## 1. Module Metadata
- Module ID: `agent.module.academic-research-pipeline`
- Version: `1.0.0`
- Maturity: `production`
- Scope: End-to-end autonomous pipeline for literature search, structural planning, writing, peer-review simulation, citation management, and LaTeX compilation for academic papers.
- Primary outcomes:
  - Human-quality, production-ready LaTeX academic manuscripts.
  - Zero presence of AI writing patterns, hallmarks, or repetitive structures.
  - Verified and cross-checked citations with automated BibTeX management.
  - Architecturally sound paper structuring via a systematic 5-phase pipeline.

## 2. Mission and Applicability
Use this module to orchestrate an AI agent codebase for writing and researching high-quality academic papers (Q1 journal and conference level).

Apply when:
- Synthesizing broad academic literature (Semantic Scholar, arXiv, CrossRef).
- Drafting technical sections, related works, and methodologies.
- Reviewing, critiquing, and humanizing prose to meet academic standards.
- Managing LaTeX formatting, building, and compilation workflows.

Do not apply directly when:
- Writing non-academic content (e.g., blog posts, generic documentation).
- Performing raw code generation lacking an academic or theoretical context.
- Generating creative prose without evidence-based constraints.

## 3. Architecture Pattern
- Pattern: `5-Phase Autonomous Pipeline + Skill Auto-Trigger Orchestration`
- Core design rules:
  - **Decoupled Execution**: Research, Structuring, Writing, Quality Assurance, and Compilation are isolated phases.
  - **Continuous Humanization**: Anti-AI pattern scanning ("humanizer") runs immediately after every drafted section, not just at the end.
  - **Strict Constraints**: Enforcement of mandatory writing rules (no colons/semicolons in prose, academic humility, full formula explanations).
  - **Parallelization**: Independent sections (e.g., related work and methodology) can be written or reviewed simultaneously.

## 4. Implementation Workflow

### Phase A: RESEARCH
1. **Broad Search**: Query academic databases (Semantic Scholar, arXiv, CrossRef) for the topic.
2. **Filter & Categorize**: Sort results by relevance, recency, and citation count. Group into thematic buckets.
3. **Gap Analysis**: Correlate what has been studied versus what is missing. Identify the novel contribution.
4. **Output**: Generate `research_notes.md` containing categorized papers, gaps, and novelty assessments.

Exit criteria:
- Literature review yields >50 high-relevance papers with explicit novelty checks.

### Phase B: STRUCTURE
1. **Outline Generation**: Create a section-by-section outline based on `research_notes.md`.
2. **Dependency Mapping**: Determine logical writing order (e.g., Intro -> Methodology -> Results -> Related Work -> Discussion -> Conclusion -> Abstract).
3. **Placements**: Identify exact locations for figures and tables (must be near first reference).
4. **Output**: Complete structured outline with quality checkpoints inserted.

Exit criteria:
- Every section has a defined purpose, key arguments, and mapped citations/evidence.

### Phase C: WRITING
1. **Section Drafting**: Follow the outline to draft sections concurrently (up to 4 independent sections at once).
2. **Humanization Pass**: Immediately run anti-AI pattern analysis on the drafted section and rewrite flagged segments.
3. **Citation Insertion**: Automatically inject `\cite{}` references and validate against the `references.bib` file.
4. **Transition Linking**: Generate seamless textual transitions between fully drafted sections.

Exit criteria:
- Complete `main.tex` with all text, citations, and figure/table references integrated.

### Phase D: QUALITY
1. **Devil's Advocate Review**: Scan with a 13-dimension academic critique. Classify issues as Critical, Important, or Minor.
2. **Global AI Pattern Scan**: Run a secondary pass across the entire document for 25+ categories of AI writing signatures.
3. **Simulated Peer Review**: Execute multi-persona (methodological, theoretical, practical) reviews outputting scores and insights.
4. **Resolution Cycle**: Fix Critical and Important issues, triggering localized humanizer passes on structural changes.

Exit criteria:
- Document passes all reviewer personas and citation validation scripts without Critical/Important flags.

### Phase E: COMPILE AND SUBMIT
1. **Compilation**: Execute LaTeX build chain (e.g., `pdflatex` -> `bibtex` -> `pdflatex`).
2. **Format Verification**: Ensure constraints (page count, font size, margin limits) for the target venue (e.g., IEEEtran, NeurIPS) are met.
3. **Error Recovery**: Automatically parse and fix LaTeX build errors or missing references.
4. **Output**: A clean, submission-ready `main.pdf`.

Exit criteria:
- Clean PDF generation with no outstanding warnings or formatting violations.

## 5. Decision Framework

| Decision Area | Preferred Option | Alternative | Selection Rule |
|---|---|---|---|
| AI Pattern Detection | Per-section humanizer pass | End-of-pipeline single pass | Per-section prevents AI hallmarks from compounding and polluting subsequent context. |
| Writing Sequence | Intro -> Methods -> Results -> Related Work | Linear (Intro -> Related -> ...) | Writing 'Related Work' later ensures better positioning against the paper's actual contribution. |
| Citation Validation | Automated BibTeX script validation | Manual LLM cross-checking | Scripts prevent hallucinated citations and ensure strict LaTeX compilation compatibility. |
| Prose Constraints | Strict rule enforcement (No colons, no generic transition words) | Default LLM generation | Enforcing strict stylistic rules is the only reliable way to pass human scrutiny. |

## 6. Validation Strategy

### Functional Validation
- Verify that every technical term is explained in plain English prior to its first use.
- Verify that every LaTeX formula includes a comprehensive "where" clause defining all symbols.
- Ensure all figures and tables appear on the same page or immediately following their first text reference.

### Failure Validation
- Check behavior when LaTeX compilation throws an error (must enter auto-fix loop).
- Verify graceful degradation if an academic database (e.g., CrossRef) rate limits the search.

### Contract/Security/Performance Validation
- Validate that the AI writing vocabulary strictly avoids the 25+ banned words (e.g., *landscape, vibrant, delve, nuanced, tapestries*).
- Confirm zero usage of em-dashes, colons, or semicolons in standard prose.

## 7. Benchmarks and SLO Targets
- AI Detection Score: `< 5%` false-positive probability on standard AI-detectors.
- Citation Integrity: `100%` of `\cite{}` commands resolve successfully in `references.bib`.
- Prose Burstiness: Sentence length variance must be high; no >4 consecutive sentences of identical length.
- Compile Success Rate: `100%` successful PDF generation for final output.

## 8. Risks and Controls
- Risk: Hallucinated citations or unverified claims.
  - Control: Integrate citation-management scripts and novelty-assessment cross-referencing.
- Risk: "Sterile" or overly-robotic academic tone.
  - Control: Enforce narrative flow rules (Story Flow rule) and utilize creative-prose critique skills to ensure readability across expertise levels.
- Risk: Scope creep causing disjointed storylines between intro and conclusion.
  - Control: Utilize the dependency mapping created generated during the STRUCTURE phase.

## 9. Agent Execution Checklist
- [ ] Determine paper metadata (Title, Authors, Target Venue, Format).
- [ ] Initialize master configuration mapping (Skill Auto-Trigger Map).
- [ ] Execute **Phase 1 (RESEARCH)**: Generate `research_notes.md`.
- [ ] Execute **Phase 2 (STRUCTURE)**: Build dependent outline with quality checkpoints.
- [ ] Execute **Phase 3 (WRITING)**: Draft sections, apply per-section humanizers, validate `\cite{}`.
- [ ] Execute **Phase 4 (QUALITY)**: Run devil's advocate review, simulated peer review, and global AI scan.
- [ ] Execute **Phase 5 (COMPILE)**: Build LaTeX, format check, and output PDF.

## 10. Reuse Notes
Adapt only:
- Target venue templates (e.g., swapping IEEEtran for a generic two-column conference format).
- Compilation commands based on local environment (Local LaTeX vs. Dockerized compilation).
- The stringency of the devil's advocate review depending on the target publication tier (Q1 journal vs. Workshop paper).
