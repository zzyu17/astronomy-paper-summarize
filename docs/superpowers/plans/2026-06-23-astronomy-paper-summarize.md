# Astronomy Paper Summarize — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a prompt-only astronomy paper summarization skill with 8 agents, 6 reference docs, two summarization modes (rough/deep), and two execution modes (subagent-driven/inline).

**Architecture:** Multi-agent team with phase boundaries — each agent is a self-contained prompt template with YAML frontmatter. Entry point `SKILL.md` detects triggers and delegates to `paper_intake_agent` for config/mode/execution routing. Phase 2a (rough) runs two agents, Phase 2b (deep) runs four agents, Phase 3 (compile) runs one agent for Markdown assembly + PDF conversion.

**Tech Stack:** Prompt-only — no Python/Node.js code. Agent templates in Markdown with YAML frontmatter. Reference docs in Markdown. `pandoc` for PDF conversion (user-managed dependency). ADS & arXiv APIs for optional metadata enrichment.

**Spec:** `docs/superpowers/specs/2026-06-22-astronomy-paper-summarize-design.md` (v1.1.0)

**Source prompts:** `/mnt/d/大学/论文/prompts/Prompts for Summarizing Literature.md`

---

## File Map

All 16 files to create:

```
astronomy-paper-summarize/
├── package.json
├── SKILL.md
├── agents/
│   ├── paper_intake_agent.md
│   ├── rough_skimmer_agent.md
│   ├── relevance_assessor_agent.md
│   ├── deep_reader_agent.md
│   ├── methodology_analyst_agent.md
│   ├── critical_evaluator_agent.md
│   ├── connection_synthesizer_agent.md
│   └── report_compiler_agent.md
└── references/
    ├── rough_overview_template.md
    ├── deep_summary_template.md
    ├── astronomical_pipeline.md
    ├── ads_api_protocol.md
    ├── arxiv_api_protocol.md
    └── config_guide.md
```

---

## Task 1: Plugin Metadata (`package.json`)

**Files:**
- Create: `package.json`

Purpose: Minimal plugin metadata identifying this skill to Copilot CLI's plugin system.

- [ ] **Step 1: Create `package.json`**

```json
{
  "name": "astronomy-paper-summarize",
  "version": "1.0.0",
  "description": "Multi-agent astronomy paper summarization — rough overview and deep summary with research-background-aware analysis",
  "author": "",
  "license": "MIT",
  "keywords": ["astronomy", "astrophysics", "paper-summary", "literature-review"],
  "skills": {
    "directory": "."
  }
}
```

- [ ] **Step 2: Verify file exists**

Run: `cat package.json | python3 -m json.tool`
Expected: Valid JSON, no parse errors.

---

## Task 2: Entry Point (`SKILL.md`)

**Files:**
- Create: `SKILL.md`
- Reference: `spec §1, §2, §9, §10`

Purpose: Skill entry point — trigger detection, mode routing, execution mode selection, agent dispatch. Uses YAML frontmatter metadata convention.

- [ ] **Step 1: Create `SKILL.md` with frontmatter and overview**

```markdown
---
name: astronomy-paper-summarize
description: "Multi-agent astronomy paper summarization skill. 8-agent team for rough overview (rapid triage) and deep summary (comprehensive critical analysis) with research-background-aware relevance scoring. 4 modes: rough, deep, both, config. Triggers on: summarize paper, paper summary, rough overview, deep summary, astro paper, astronomy paper, analyze paper, break down paper, what does this paper say, 论文总结, 天文学论文."
metadata:
  version: "1.0.0"
  last_updated: "2026-06-23"
  status: active
  task_type: open-ended
  required_dependencies:
    - pandoc
  optional_dependencies:
    - ADS API token
---

# Astronomy Paper Summarize

Transforms astronomy research papers into structured, actionable summaries powered by an 8-agent team whose behavior is guided by the user's own research background.

**Two summarization modes:**
- **Rough Overview** — Rapid literature triage: research question, methodology, 1-3 key findings, relevance score, quick takeaway. 2 agents, ~2-3 min.
- **Deep Summary** — Comprehensive critical analysis: structured summary, methodological deep dive, critical evaluation, research connection, terminology glossary, new directions. 5 agents, ~5-8 min.

**Two execution modes:**
- **Subagent-Driven** (default) — Each agent dispatched as a fresh sub-agent with isolated context. Better for long/complex papers.
- **Inline Execution** — Agents executed sequentially in-session as structured checklists. Better for short papers or quick triage.

> **Routing discipline:** This skill assumes the user wants to summarize an astronomy paper. If the user's intent is ambiguous (e.g., they might want original research or peer review instead), clarify before proceeding.
```

- [ ] **Step 2: Append trigger conditions section**

```markdown
## Trigger Conditions

### Trigger Keywords (case-insensitive)

**Paper summarization:**
- "summarize paper", "summarize this paper", "paper summary", "summarize pdf"
- "rough overview", "deep summary", "paper deep dive"
- "analyze paper", "review this paper for me"
- "what does this paper say", "break down this paper"

**Astronomy-specific:**
- "astro paper", "astronomy paper", "astrophysics paper"
- "exoplanet paper", "cosmology paper", "galaxy paper"

### Anti-Triggers

This skill is specifically for **summarizing** astronomy papers. It should NOT activate when the user wants to conduct original research, write a paper, or perform formal peer review — those are separate workflows.

### Detection Logic

If the user mentions any trigger keyword AND provides or references a paper (PDF path, arXiv ID, title, URL), activate this skill. If no paper is provided, ask the user to provide one.
```

- [ ] **Step 3: Append mode selection and routing flow**

```markdown
## Mode Selection & Routing

When activated, follow this flow:

1. **Trigger detected** → Load this skill context
2. **Read `agents/paper_intake_agent.md`** → Execute intake as the first agent
3. **paper_intake_agent handles:**
   - Load/create `.astro-paper/config.yaml`
   - Prompt for research background if missing (Core Research Topic, Primary Goal, Proposed Methodology)
   - Generic fallback if user skips: auto-extract from abstract, present for approval
   - Ask user: **Rough**, **Deep**, or **Both**?
   - Ask user: **Subagent-Driven** (default) or **Inline Execution**?
   - Extract paper metadata via ADS → arXiv → manual fallback
4. **Route based on answers:**
   - **Rough**: Phase 2a → Phase 3
   - **Deep**: Phase 2b → Phase 3
   - **Both**: Phase 2a → Phase 2b → Phase 3

## Agent Team (8 Agents)

| # | Agent | Role | Phase | Mode |
|---|-------|------|-------|------|
| 1 | `paper_intake_agent` | Config + metadata + routing | Phase 1 | Both |
| 2 | `rough_skimmer_agent` | Rapid skimming, core extraction, jargon simplification | Phase 2a | Rough |
| 3 | `relevance_assessor_agent` | Multi-dimensional relevance scoring, implications | Phase 2a | Rough |
| 4 | `deep_reader_agent` | Structured summary (I→M→R→D→C) | Phase 2b | Deep |
| 5 | `methodology_analyst_agent` | Technical methodology, paper-type-adaptive extraction | Phase 2b | Deep |
| 6 | `critical_evaluator_agent` | Critical analysis: strengths, limitations, robustness | Phase 2b | Deep |
| 7 | `connection_synthesizer_agent` | Research connection, glossary, references | Phase 2b | Deep |
| 8 | `report_compiler_agent` | Markdown assembly, PDF conversion, naming | Phase 3 | Both |

## Phase Boundaries

| Phase | Agents | Deliverable |
|-------|--------|-------------|
| **Phase 1 — Intake** | `paper_intake_agent` | Config loaded/created, mode + execution mode selected, paper metadata + full text |
| **Phase 2a — Rough** | `rough_skimmer_agent` → `relevance_assessor_agent` | Rough Overview Markdown content |
| **Phase 2b — Deep** | `deep_reader_agent` → `methodology_analyst_agent` → `critical_evaluator_agent` → `connection_synthesizer_agent` | Deep Summary Markdown content |
| **Phase 3 — Compile** | `report_compiler_agent` | Final Markdown files + PDFs |

When `both` mode: Phase 2a → Phase 2b → Phase 3.
```

- [ ] **Step 4: Append execution mode dispatch instructions**

````markdown
## Execution Mode Dispatch

### Subagent-Driven (Default)

For each agent in the active phase sequence, dispatch as a sub-agent. For example, in Copilot CLI, dispatch sub-agents with:

```
task({
  agent_type: "general-purpose",
  name: "<agent-name>",
  prompt: [full content of agents/<agent_name>.md +
           paper text (for analysis agents) +
           research background from config +
           accumulated outputs from prior agents in the phase]
})
```

Sub-agent dispatch wrapper differs across platforms and should be adapted depending on the running platform. Collect output after each agent completes. Pass accumulated context to the next agent.

### Inline Execution

For each agent in the active phase sequence, work through sequentially:

1. Read the agent template from `agents/<agent_name>.md`
2. Execute the agent's instructions as a structured task in the current session
3. Collect and persist the output
4. Move to the next agent

**Warning:** If running `both` mode on a long paper, monitor context usage. Warn the user if approaching context limits.

### Fallback

If Subagent-Driven mode is selected but the platform does not support sub-agent dispatch, fall back to Inline Execution and inform the user.
````

- [ ] **Step 5: Verify file structure**

Run: `wc -l SKILL.md`
Expected: ~120-150 lines.

---

## Task 3: Output Templates (`references/rough_overview_template.md`, `references/deep_summary_template.md`)

**Files:**
- Create: `references/rough_overview_template.md`
- Create: `references/deep_summary_template.md`
- Reference: `spec §5.2, §5.3`

Purpose: Output format templates that the `report_compiler_agent` uses for assembly. Content derived from original prompt sets.

- [ ] **Step 1: Create `references/rough_overview_template.md`**

```markdown
---
name: rough_overview_template
description: "Output template for the rough overview report — used by report_compiler_agent during Phase 3 assembly"
version: "1.0.0"
---

# Rough Overview Template

The `report_compiler_agent` assembles output from `rough_skimmer_agent` and `relevance_assessor_agent` into this structure.

## Template

```markdown
# [Paper Title] — Rough Overview

---

## 1. Core Information Summary

| Element | Details |
|---------|---------|
| **Research Question** | [What specific problem or objective does the paper address?] |
| **Methodology** | [High-level: primary data sources, observational tools, analytical methods, theoretical models, simulation techniques] |
| **Core Findings** | 1. [Most significant result 1]<br>2. [Most significant result 2]<br>3. [Most significant result 3] |
| **Stated Limitations** | [Key limitations or open questions explicitly stated by authors; omit if none prominent] |

## 2. Relevance Assessment

**Relevance Score**: [1-10] / 10
**Classification**: [Must-Read / Maybe / Not-Essential]
**Explanation**: [1-sentence rationale]

### Project Implications
1. [Key insight 1 for user's research]
2. [Key insight 2 for user's research]
3. [Key insight 3 for user's research]

## 3. Quick Takeaway

> [Single sentence summarizing the most critical piece of information for the user's research]

```

## Assembly Rules

1. All fields are required unless marked optional ("omit if none prominent")
2. Jargon is simplified inline — no separate glossary in rough mode
3. Brevity is paramount: bullet points, short sentences
4. Relevance score uses the 5-factor weighted framework from `relevance_assessor_agent`
5. Paper title is sanitized for filesystem safety (remove `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`)

- [ ] **Step 2: Create `references/deep_summary_template.md`**

```markdown
---
name: deep_summary_template
description: "Output template for the deep summary report — used by report_compiler_agent during Phase 3 assembly"
version: "1.0.0"
---

# Deep Summary Template

The `report_compiler_agent` assembles output from `deep_reader_agent`, `methodology_analyst_agent`, `critical_evaluator_agent`, and `connection_synthesizer_agent` into this structure.

## Template

```markdown
# [Paper Title] — Deep Summary

---

## 1. Structured Paper Summary

### Introduction
[Research background, key questions, and objectives]

### Methods
[Core methodological framework, data sources, sample selection, analysis pipelines, statistical/computational tools]

### Results
[Primary findings and key evidence supporting the conclusions]

### Discussion
[Interpretation of results, connections to prior work, identified limitations]

### Conclusion
[Core takeaways and proposed future research directions]

## 2. Methodological Deep Dive

### Simplified Framework
[What the method does and why it was chosen over alternatives]

### Key Technical Details
[Specific observational designs, data analysis pipelines, statistical methods, robustness checks, critical parameters, datasets, software, code libraries worth referencing]

*Note: Technical details are adapted to detected paper type — observational / theoretical / computational.*

## 3. Critical Evaluation

### Strengths
[Major novel contributions, robust evidence, innovative methodological designs]

### Limitations
[Potential weaknesses in methodology/assumptions, uncertainties, caveats — both explicit and implicit]

### Evidence Robustness
[How reliable and generalizable the evidence is; adequacy of robustness checks; alternative explanations considered]

## 4. Research Connection & Relevance

### Hypothesis Alignment
[How the paper supports, challenges, or complements the user's research hypotheses]

### Methodological Inspiration
[Specific ways the paper's approaches can be adopted, modified, extended, or challenged in the user's research]

### Gap Filling
[Unaddressed research gaps and open questions the user's research can target]

### New Research Directions
1. [Novel research question or follow-up study — concrete and actionable]
2. [Novel research question or follow-up study — concrete and actionable]
3. [Novel research question or follow-up study — concrete and actionable]

## 5. Key Terminology & References

### Key Technical Terms
| Term | Definition & Context |
|------|---------------------|
| **[Term 1]** | [Concise yet clear definition, central to understanding the paper and critical to the user's research] |
| **[Term 2]** | [...] |

### High-Impact References
| # | Reference | Relevance to the User's Research |
|---|-----------|---------------------------|
| 1 | [Author (Year), *Journal*, DOI] | [1-sentence explanation of relevance] |
| 2 | ... | ... |

## 6. Overall Research Implication Summary

[A concise, high-level paragraph summarizing how this paper's insights should influence the design, execution, and/or interpretation of the user's research]

```

## Assembly Rules

1. All sections are required
2. Structured Summary (§1) follows Astrobites template: engaging lede → background → methods → results → implications
3. Methodological Deep Dive (§2) adapts detail level to detected paper type (observational/theoretical/computational)
4. Critical Evaluation (§3) maintains constructive tone — critical but aimed at empowering research progress
5. Research Connection (§4) is the most critical section — every insight must be anchored to the user's config
6. Key Terminology (§5) provides concise yet clear definitions; 3-5 high-impact references with 1-sentence relevance each
7. Paper title is sanitized for filesystem safety

- [ ] **Step 3: Verify both files**

Run: `wc -l references/rough_overview_template.md references/deep_summary_template.md`
Expected: ~60-80 lines each.

---

## Task 4: Domain & API Reference Docs

**Files:**
- Create: `references/astronomical_pipeline.md`
- Create: `references/ads_api_protocol.md`
- Create: `references/arxiv_api_protocol.md`
- Create: `references/config_guide.md`
- Reference: `spec §4, §6, §7`

Purpose: Context knowledge for agents and user-facing documentation.

- [ ] **Step 1: Create `references/astronomical_pipeline.md`**

```markdown
---
name: astronomical_pipeline
description: "7-phase astronomical research pipeline — context knowledge for methodology_analyst_agent and deep-reading agents"
version: "1.0.0"
---

# The 7-Phase Astronomical Research Pipeline

Modern astronomical research typically flows through seven interconnected phases. This pipeline is **context knowledge** for agents — not a rigid mapping. Each paper may cover some, many, or all phases. Rich feedback loops exist between all phases.

## Pipeline Phases

| Phase | Description | Key Activities | Typical Outputs |
|-------|-------------|---------------|-----------------|
| **1. Theory & Physical Modeling** | Mathematical/analytical framework | Deriving physical equations, building conceptual models (ΛCDM, core accretion, MRI), making predictions | Equations, scaling relations, analytical predictions |
| **2. Numerical Simulation** | Computational implementation | N-body, hydro, MHD, radiative transfer; large simulation suites (IllustrisTNG, EAGLE, CAMELS) | Simulated catalogs, mock observations, parameter spaces |
| **3. Observational Design & Proposal** | Planning observations | Target selection, feasibility calculations, instrument selection, exposure time estimation, proposal writing | Observing proposals, target lists, feasibility reports |
| **4. Data Acquisition** | Collecting data | Imaging, spectroscopy, interferometry, time-domain surveys (TESS, JWST, ALMA, VLA, SDSS, LSST) | Raw data files (FITS, HDF5) |
| **5. Data Reduction & Calibration** | Processing raw data | Bias/flat/dark correction, wavelength/flux calibration, astrometric calibration, source extraction | Reduced data products, catalogs |
| **6. Statistical Inference & Analysis** | Extracting physical parameters | MCMC, nested sampling, SED fitting, machine learning classification, Bayesian inference, population statistics | Parameter posteriors, HR diagrams, mass functions |
| **7. Interpretation & Model Comparison** | Connecting to physics | Bayes factors, posterior predictive checks, theory-data comparison, physical interpretation | Scientific conclusions, theory constraints |

## Key Insight

All phases are **bidirectionally connected**. The pipeline is a web, not a line:
- Simulations (Phase 2) inform observational design (Phase 3) and interpret results (Phase 7)
- Observations (Phase 4) constrain theory (Phase 1) and drive new simulations (Phase 2)
- Statistical methods (Phase 6) evolve based on data characteristics (Phase 5) and theoretical requirements (Phase 1)

## Subfield Variations

| Subfield | Typical Phase Emphasis |
|----------|----------------------|
| **Exoplanets** | 3-4-5-6 (observation-heavy, transit/radial velocity pipelines) |
| **Cosmology** | 1-2-6-7 (theory + simulation + statistical inference) |
| **Galaxy Evolution** | 2-4-5-6 (simulation + multi-wavelength surveys) |
| **Stellar Astrophysics** | 4-5-6-7 (spectroscopy + stellar models) |
| **High-Energy Astrophysics** | 4-5-6 (time-domain + multi-messenger) |

## Agent Usage

When the `methodology_analyst_agent` analyzes a paper:
1. Recognize which phases the paper covers (most papers cover 2-4 phases)
2. Use phase awareness when explaining methods — help the user understand where this work fits
3. Do NOT force every paper into all seven phases
4. Do NOT use this as a rigid checklist — it's background context
```

- [ ] **Step 2: Create `references/ads_api_protocol.md`**

````markdown
---
name: ads_api_protocol
description: "ADS API usage guide for paper metadata enrichment — used by paper_intake_agent during Phase 1"
version: "1.0.0"
---

# ADS API Protocol

The [Astrophysics Data System (ADS) API](https://api.adsabs.harvard.edu/v1/search/query) serves as the **primary** metadata source for paper intake.

## Endpoint

```
https://api.adsabs.harvard.edu/v1/search/query
```

## Authentication

Requires an ADS API token. The user sets this as an environment variable:
```bash
export ADS_API_TOKEN="your-token-here"
```

Get a token at: https://ui.adsabs.harvard.edu/user/settings/token

## Key Fields Returned

| Field | Description |
|-------|-------------|
| `title` | Paper title |
| `abstract` | Full abstract |
| `author` | Author list |
| `keyword` | Author-provided and ADS-assigned keywords |
| `year` | Publication year |
| `citation_count` | Number of citations |
| `arxiv_class` | arXiv category (e.g., astro-ph.EP) |
| `bibcode` | ADS bibliographic code (unique identifier) |
| `doi` | Digital Object Identifier |
| `pub` | Journal name |

## Query Patterns

### Pattern 1: Search by arXiv ID
```
GET ?q=arxiv:{arxiv_id}&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

### Pattern 2: Search by Title
```
GET ?q=title:"{paper_title}"&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

### Pattern 3: Search by Bibcode
```
GET ?q=bibcode:{bibcode}&fl=title,abstract,author,keyword,year,citation_count,arxiv_class,bibcode,doi,pub
```

## Rate Limits

- 5,000 queries per day (standard token)
- Higher limits available for institutional tokens

## Special Operators

| Operator | Usage | Description |
|----------|-------|-------------|
| `citations()` | `citations(bibcode:2020ApJ...905L..11B)` | Papers citing the given paper |
| `references()` | `references(bibcode:2020ApJ...905L..11B)` | Papers referenced by the given paper |
| `similar()` | `similar(bibcode:2020ApJ...905L..11B)` | Similar papers based on text analysis |
| `trending()` | `trending(astronomy)` | Currently trending papers |

## Error Handling

| Condition | Action |
|-----------|--------|
| No API token | Skip ADS, fall back to arXiv API |
| HTTP 401/403 | Report authentication error, fall back to arXiv API |
| HTTP 429 (rate limit) | Fall back to arXiv API |
| HTTP 5xx / timeout | Fall back to arXiv API |
| Query returns zero results | Try title search; if still zero, fall back to arXiv API |

## Integration Note

The paper's **full text** always comes from the user-provided file. ADS API provides metadata and abstract enrichment only — it does not replace the need for the full paper text.
````

- [ ] **Step 3: Create `references/arxiv_api_protocol.md`**

````markdown
---
name: arxiv_api_protocol
description: "arXiv API usage guide for paper metadata — used by paper_intake_agent as first fallback when ADS is unavailable"
version: "1.0.0"
---

# arXiv API Protocol

## Endpoint

```
https://export.arxiv.org/api/query
```

## Response Format

Atom 1.0 XML feed. A match yields one or more `<entry>` elements; a miss yields a feed with zero `<entry>` elements.

## Rate Limit

~3 seconds between requests (per arXiv terms of use). No polite-pool or higher-tier mechanism.

## Query Patterns

### Pattern 1: arXiv ID Lookup
```
GET ?id_list={arxiv_id}
```
Example: `?id_list=2301.12345`

### Pattern 2: Title Search
```
GET ?search_query=ti:"{title}"&max_results=5
```
Example: `?search_query=ti:"The Neptunian Desert Across Stellar Types"&max_results=5`

## Key Fields per Entry

| XML Path | Description |
|----------|-------------|
| `<entry><title>` | Paper title (arXiv may insert internal whitespace — collapse to single spaces) |
| `<entry><summary>` | Abstract |
| `<entry><published>` | ISO-8601 timestamp (first 4 digits = year) |
| `<entry><author><name>` | Author names |
| `<entry><arxiv:primary_category>` | Primary arXiv category |

## Error Handling

| Condition | Action |
|-----------|--------|
| Empty feed (zero `<entry>`) | Treat as miss — fall back to manual metadata entry |
| HTTP 429 (rate limit) | Back off 2 seconds, retry up to 3 times; after exhaustion, fall back to manual |
| HTTP 5xx | Fall back to manual metadata entry immediately |
| Network timeout (30s) | Fall back to manual metadata entry |
| Malformed XML | Fall back to manual metadata entry |

## Integration Note

arXiv API is the **first fallback** after ADS API. It provides metadata and abstract only — the paper's full text comes from the user-provided file.
````

- [ ] **Step 4: Create `references/config_guide.md`**

````markdown
---
name: config_guide
description: "User-facing documentation for .astro-paper/config.yaml — research background configuration"
version: "1.0.0"
---

# Research Background Configuration Guide

The `.astro-paper/config.yaml` file stores the user's research background, enabling all summarization agents to filter and connect paper insights specifically to the user's research project.

## Location

```
<parent-directory>/.astro-paper/config.yaml
```

The file is created in the **parent directory of the user's paper project directory**. All papers in the same project share one configuration.

**Example**: If the user's paper is at `/mnt/d/papers/neptunian-desert/neptunian-desert-paper-1/neptunian-desert-paper-1.pdf`, the config lives at `/mnt/d/papers/neptunian-desert/.astro-paper/config.yaml`. All papers under `/mnt/d/papers/neptunian-desert/` share this one config.

## Schema

```yaml
# Astronomy Paper Summarize — Research Background Configuration
# Last updated: YYYY-MM-DD

research_background:
  core_topic: "The user's research topic (e.g., Planets in Neptunian Desert)"
  primary_goal: "What the user aims to achieve (e.g., To understand formation mechanisms...)"
  proposed_methodology: "The user's planned approach (e.g., Statistical analysis of TESS data...)"

output:
  pdf_converter_rough: ""
  pdf_converter_deep: ""
```

## Fields

| Field | Description | Example |
|-------|-------------|---------|
| `core_topic` | The specific research area or phenomenon the user studies | "Planets in Neptunian Desert" |
| `primary_goal` | What the user ultimately wants to understand or discover | "To draw statistical patterns and understand formation/evolution mechanisms" |
| `proposed_methodology` | The user's planned approach — data sources, methods, tools | "Large-scale TESS data, statistical analysis of exoplanet populations" |
| `pdf_converter_rough` | Command to convert rough overview Markdown to PDF | Populated after first successful conversion |
| `pdf_converter_deep` | Command to convert deep summary Markdown to PDF | Populated after first successful conversion |

## How Config is Used

- **Relevance scoring**: The user's `core_topic`, `primary_goal`, and `proposed_methodology` are the reference point for scoring each paper's relevance to the user's research
- **Research connection**: The `connection_synthesizer_agent` maps paper insights directly onto the user's `primary_goal` and `proposed_methodology`
- **PDF conversion**: The `report_compiler_agent` uses stored `pdf_converter_*` commands for reliable re-conversion

## PDF Converter Commands

The `pdf_converter_rough` and `pdf_converter_deep` fields store the shell command used to convert Markdown summaries to PDF. They are managed automatically by the `report_compiler_agent`, but the user can also edit them manually.

### Automatic Management

**Initial state**: Both fields start empty (`""`).

**First conversion**: When the `report_compiler_agent` runs its first PDF conversion for each mode, it uses `pandoc` as the default converter:
```
pandoc --pdf-engine=xelatex -V geometry:margin=1in <markdown-file> -o <pdf-file>
```
If the conversion succeeds, the command (with actual filenames) is stored in the config for that mode. If the default command fails (e.g., `xelatex` not installed), the agent will try alternatives and prompt the user for help.

**Re-conversion**: On subsequent runs, the agent uses the stored command. If it fails, the agent falls back to the pandoc default, updates the config entry, and notifies the user.

### Manual Editing

The stored commands use actual filenames (not placeholders), so the user can run them directly from the paper directory. The user can also edit the `pdf_converter_rough` or `pdf_converter_deep` fields to use a different converter:

```yaml
output:
  pdf_converter_rough: "pandoc --pdf-engine=weasyprint paper-summaries/My Paper-rough-overview.md -o My Paper-rough-overview.pdf"
  pdf_converter_deep: "pandoc --pdf-engine=xelatex -V geometry:margin=1in paper-summaries/My Paper-deep-summary.md -o My Paper-deep-summary.pdf"
```

**To re-convert PDFs** after editing the Markdown summaries, run the stored command directly from the paper directory.

### Example Commands

Stored commands use actual filenames. Below are the equivalents before filename substitution:

| Converter | Command Pattern |
|-----------|----------------|
| **pandoc + xelatex** (default) | `pandoc --pdf-engine=xelatex -V geometry:margin=1in <md-file> -o <pdf-file>` |
| **pandoc + weasyprint** | `pandoc --pdf-engine=weasyprint <md-file> -o <pdf-file>` |
| **pandoc + wkhtmltopdf** | `pandoc --pdf-engine=wkhtmltopdf <md-file> -o <pdf-file>` |

## Generic Fallback

If the user skips config creation, the `paper_intake_agent` will:
1. Auto-extract approximate values from the paper's abstract
2. Present them for the user's review/refinement
3. Save approved values to `.astro-paper/config.yaml` for future use

## Updating Config

The user can update their research background at any time during a summarization session. The agent should prompt: "Would you like to update your research background?" when appropriate.

## Cross-Agent Portability

`.astro-paper/config.yaml` is plain YAML — readable by Copilot CLI, Claude Code, Codex, or any editor. No tool-specific format or dependencies.
````

- [ ] **Step 5: Verify all four files**

Run: `wc -l references/astronomical_pipeline.md references/ads_api_protocol.md references/arxiv_api_protocol.md references/config_guide.md`
Expected: ~60-80 lines each.

---

## Task 5: Phase 1 — `paper_intake_agent.md`

**Files:**
- Create: `agents/paper_intake_agent.md`
- Reference: `spec §2.2, §3.2 Agent 1, §4, §7`

Purpose: Unified entry point — config management, paper content acquisition, mode + execution selection, routing.

- [ ] **Step 1: Create `agents/paper_intake_agent.md`**

```markdown
---
name: paper_intake_agent
description: "Unified entry point for astronomy paper summarization — research background config, paper metadata extraction, mode selection, execution mode selection, and routing to analysis phases"
model: inherit
phase: 1
mode: both
---

# Paper Intake Agent

## Role Definition

You are the Paper Intake Agent — the unified entry point for the astronomy paper summarization skill. You handle three responsibilities in sequence: (1) research background configuration, (2) paper content acquisition and metadata extraction, (3) mode + execution mode selection and routing.

## Phase 1 Flow

### Step 1: Check for Existing Config

Look for `.astro-paper/config.yaml` in the **parent directory of the paper project directory**. If found, load and validate the three required fields:
- `research_background.core_topic`
- `research_background.primary_goal`
- `research_background.proposed_methodology`

If config exists and all fields are populated, display them to the user and ask: "Use this research background? (yes / update)"

### Step 2: Config Creation / Update

If config is missing, stale, or the user wants to update, prompt for the three fields one at a time:

1. **Core Research Topic**: "What is your core research topic? (e.g., 'Planets in Neptunian Desert', 'Galaxy cluster mass profiles')"
2. **Primary Goal**: "What is your primary research goal? (e.g., 'To understand formation and evolution mechanisms that create the Neptunian desert')"
3. **Proposed Methodology**: "What methodology do you plan to use? (e.g., 'Statistical analysis of TESS exoplanet data', 'Weak lensing mass reconstruction')"

Write the config to `.astro-paper/config.yaml` in the parent directory of the paper project directory.

### Step 3: Generic Fallback (if user skips config)

If the user chooses to skip providing research background:

1. Read the paper's abstract (from user-provided file or API)
2. Auto-extract approximate values for all three fields by analyzing the abstract's content
3. Present to the user: "Based on your paper, I've inferred the following research background. Please refine or approve:"
   - Show the three auto-extracted fields
   - Ask: "Approve as-is, refine specific fields, or skip entirely?"
4. If user approves or refines: save to `.astro-paper/config.yaml`
5. If user skips entirely: proceed with auto-extracted values in-memory (do not save)

### Step 4: Paper Content Acquisition

Use the **ADS → arXiv → manual** fallback chain:

1. **Try ADS API first** (if token available):
   - Query by arXiv ID (if user provided) or paper title
   - Extract: title, authors, year, abstract, citation count, bibcode, doi, arXiv class, keywords
   - Reference: `references/ads_api_protocol.md`

2. **Fall back to arXiv API** (if ADS unavailable):
   - Query by arXiv ID or title search
   - Extract: title, authors, year, abstract, primary category
   - Reference: `references/arxiv_api_protocol.md`

3. **Final fallback — manual extraction**:
   - If both APIs unavailable, extract metadata from the user-provided PDF/text
   - Read the first page: extract title, authors, abstract
   - Prompt user to verify or fill in missing fields

**Important**: The paper's **full text** always comes from the user-provided file (PDF, text, or pasted content). APIs supply metadata and abstract only.

### Step 5: Mode Selection

Ask the user:
"Which summarization mode would you like?"

Options:
- **Rough Overview** — Rapid triage: research question, methodology, 1-3 findings, relevance score (2 agents, ~2-3 min)
- **Deep Summary** — Comprehensive analysis: structured summary, methodology deep dive, critical evaluation, research connection, glossary (5 agents, ~5-8 min)
- **Both** — Rough overview first for quick triage, then deep summary for full analysis

### Step 6: Execution Mode Selection

Ask the user:
"How should I execute the analysis?"

Options:
- **Subagent-Driven** (default, recommended) — Each agent runs as a fresh sub-agent with isolated context. Better for long/complex papers.
- **Inline Execution** — Agents run sequentially in this session as a checklist. Better for short papers or quick triage.

### Step 7: Route to Analysis Phases

Based on selections, pass control to the next phase with the following context:
- Paper full text
- Paper metadata (from API or extraction)
- Research background (from config or fallback)
- Selected modes (rough/deep/both)
- Selected execution mode (subagent/inline)

**Routing rules:**
- **Rough only**: Phase 2a (`rough_skimmer_agent` → `relevance_assessor_agent`) → Phase 3 (`report_compiler_agent`)
- **Deep only**: Phase 2b (`deep_reader_agent` → `methodology_analyst_agent` → `critical_evaluator_agent` → `connection_synthesizer_agent`) → Phase 3
- **Both**: Phase 2a → Phase 2b → Phase 3

## Output

At the end of Phase 1, you must have produced:
1. Loaded/created research background config (from file or fallback)
2. Paper metadata (title, authors, year, abstract, identifiers)
3. Paper full text (from user's file)
4. Selected summarization mode(s)
5. Selected execution mode

These are passed as context to all downstream agents.

## Error Handling

| Scenario | Response |
|----------|----------|
| No paper provided | Ask user: "Please provide a paper — PDF path, arXiv ID, title, or paste the text." |
| PDF cannot be parsed (corrupted, scanned) | Ask user: "I can't extract text from this PDF. Please provide the arXiv ID, paste the abstract, or share a text version." |
| No API available (ADS + arXiv both down) | Extract metadata from paper text; prompt user to verify |
| Config directory not writable | Store config in-memory only; warn user |
| User cancels at mode selection | Exit gracefully: "Summarization cancelled. You can restart anytime." |
```

- [ ] **Step 2: Verify file**

Run: `grep -c "^##\|^###\|^-" agents/paper_intake_agent.md`
Expected: clear section structure with flow steps.

---

## Task 6: Phase 2a Agents (Rough Mode)

**Files:**
- Create: `agents/rough_skimmer_agent.md`
- Create: `agents/relevance_assessor_agent.md`
- Reference: `spec §3.2 Agents 2-3`, original prompts §1-6 (Rough Overview)

Purpose: Rapid literature triage — core extraction + relevance scoring.

- [ ] **Step 1: Create `agents/rough_skimmer_agent.md`**

```markdown
---
name: rough_skimmer_agent
description: "Rapid paper skimming for core information extraction with inline jargon simplification"
model: inherit
phase: 2a
mode: rough
dependencies:
  - paper full text (from paper_intake_agent)
  - research_background (from config/fallback)
---

# Rough Skimmer Agent

## Role Definition

You are the Rough Skimmer Agent — a specialized rapid literature interpretation assistant focused on astronomical papers. Your purpose is distilling core information from a quick reading without delving into technical details. You help the user triage their reading list efficiently.

## Core Principles

1. **Brevity is paramount** — bullet points, short sentences. Every word must earn its place.
2. **Focus on the core** — extract only the primary research question, main methods, and most significant 1-3 findings.
3. **No deep dives** — high-level descriptions only. Do not explain technical intricacies.
4. **Explicit information only** — report what the paper states, not what you infer.
5. **Jargon simplification** — translate technical terminology into more understandable concepts where possible, without losing essential meaning. Do this inline within the summary text (plain-language equivalents, brief analogies). No separate glossary.

## Output Sections

Produce the following sections in order:

### 1. Core Information Summary

| Element | Content |
|---------|---------|
| **Research Question** | What specific problem or objective does the paper address? One sentence. |
| **Methodology** | High-level description: primary data sources, observational tools, analytical methods, theoretical models, simulation techniques. 2-4 bullet points. |
| **Core Findings** | 1-3 most significant results or conclusions. Exclude secondary outcomes. Numbered list. |
| **Stated Limitations** | Key limitations or open questions explicitly stated by the authors. If none are prominent, omit this row. |

### 2. Paper Type Detection

Briefly note the detected paper type (observational / theoretical / computational / mixed). This helps downstream agents adapt their analysis.

**Detection signals:**
- "observed", "detected", "survey", "telescope" → Observational
- "model", "theory", "predict", "analytic" → Theoretical
- "simulation", "N-body", "hydro", "MHD" → Computational

If signals are mixed, note all applicable types.

## Constraints

- Do NOT produce a dedicated terminology glossary (that's deep mode only)
- Do NOT evaluate paper quality or critique methodology (that's deep mode: `critical_evaluator_agent`)
- Do NOT connect findings to the user's research (that's `relevance_assessor_agent`)
- Do NOT infer limitations the authors don't state
- Keep entire output under ~300 words

## Output Format

Raw Markdown content for the "Core Information Summary" section. This will be assembled by `report_compiler_agent` into the `rough_overview_template.md` structure.
```

- [ ] **Step 2: Create `agents/relevance_assessor_agent.md`**

```markdown
---
name: relevance_assessor_agent
description: "Multi-dimensional relevance scoring against user's research project with actionable implications"
model: inherit
phase: 2a
mode: rough
dependencies:
  - paper metadata + full text (from paper_intake_agent)
  - research_background (from config/fallback)
  - rough_skimmer_agent output (Core Information Summary)
---

# Relevance Assessor Agent

## Role Definition

You are the Relevance Assessor Agent. You evaluate how relevant the paper is to the user's specific research project using a weighted 5-factor scoring framework, then produce concrete, actionable implications and a quick-takeaway summary.

## Core Principles

1. **Score against the user's research, not the field** — a Nobel-worthy cosmology paper scores low if the user studies exoplanets.
2. **Evidence-based scoring** — explain your score with specific references to paper content.
3. **Actionable implications** — every implication should suggest something the user can do (adopt a method, check a result, avoid a pitfall).
4. **Honest triage** — don't inflate scores to make papers seem useful. A low-relevance paper with a clear "why" is more helpful than artificial relevance.

## Relevance Scoring Framework

Score on a **1-10** scale using weighted dimensions:

| Dimension | Weight | How to Assess |
|-----------|--------|---------------|
| **Topic Match** | 35% | Keyword/title overlap with user's `core_topic`. arXiv category alignment. Phenomenon studied. |
| **Methodology Match** | 25% | Same telescope/instrument, same wavelength regime, same analysis technique, similar data type (transit, RV, imaging, spectroscopic). Compare to user's `proposed_methodology`. |
| **Target Match** | 20% | Same object class, same redshift range, same stellar population, same parameter space. |
| **Impact** | 10% | Citation count (if available), journal tier, author prominence in the subfield. |
| **Timeliness** | 10% | Recency (last 3 years vs older), connection to active debates, relevance to upcoming surveys. |

**Scoring bands:**
- **9-10**: Direct match — same topic, method, and targets as user's research
- **7-8**: Related subfield — overlapping topic or method, different targets
- **5-6**: Overlapping — some shared concepts but different focus
- **1-4**: Not relevant — minimal connection to user's research

## Output Sections

### 1. Relevance Assessment

**Relevance Score**: [1-10] / 10
**Classification**: [Must-Read / Maybe / Not-Essential]
**Explanation**: One sentence explaining the score with specific reference to paper content.

### 2. Project Implications

1-3 concrete, actionable insights for the user's research:
- What should the user know from this paper?
- What might they adopt, modify, or avoid?
- Each implication must reference the user's `primary_goal` or `proposed_methodology`.

### 3. Quick Takeaway

A single sentence summarizing the most critical piece of information for the user's research. This is the "if you read nothing else, know this" summary.

## Constraints

- All assessments must reference the user's research background
- Each implication must be actionable (not just "this is interesting")
- Score honestly — not all papers are highly relevant, and that's useful information
- Keep entire output under ~200 words

## Output Format

Raw Markdown content for the "Relevance Assessment" and "Quick Takeaway" sections. This will be assembled by `report_compiler_agent` into the `rough_overview_template.md` structure.
```

- [ ] **Step 3: Verify both files**

Run: `wc -l agents/rough_skimmer_agent.md agents/relevance_assessor_agent.md`
Expected: ~60-100 lines each.

---

## Task 7: Phase 2b Agents Batch 1 — Deep Reader + Methodology Analyst

**Files:**
- Create: `agents/deep_reader_agent.md`
- Create: `agents/methodology_analyst_agent.md`
- Reference: `spec §3.2 Agents 4-5`, original prompts §1-6 (Deep Summary)

Purpose: Structured paper summary + technical methodology deconstruction.

- [ ] **Step 1: Create `agents/deep_reader_agent.md`**

```markdown
---
name: deep_reader_agent
description: "Structured comprehensive paper summary following Astrobites 14-year-validated template (Introduction → Methods → Results → Discussion → Conclusion)"
model: inherit
phase: 2b
mode: deep
dependencies:
  - paper full text + metadata (from paper_intake_agent)
  - research_background (from config/fallback)
---

# Deep Reader Agent

## Role Definition

You are the Deep Reader Agent — a seasoned academic in astronomy specializing in comprehensive paper analysis. You produce a structured, 5-section summary following the Astrobites template (validated over 14 years of astronomy paper blogging). Your output provides the foundational understanding that downstream agents (`methodology_analyst_agent`, `critical_evaluator_agent`, `connection_synthesizer_agent`) build upon.

## Core Principles

1. **Follow the Astrobites pattern**: engaging lede → background → methods → results → implications.
2. **Maintain technical accuracy** — correctly interpret specialized methods, parameters, and terminology without misrepresentation.
3. **Adapt level of detail** based on paper type (observational/theoretical/computational).
4. **Be comprehensive but focused** — cover all five sections without excessive detail. Aim for ~500-800 words total.

## Output Sections

### 1. Introduction

- Research background: What is the broader context of this work?
- Key questions: What specific questions does the paper aim to answer?
- Objectives: What are the stated goals?

### 2. Methods

- Core methodological framework: What approach was taken?
- Data sources: What observations, simulations, or datasets were used?
- Sample selection: What was studied and why?
- Analysis pipelines: Key steps in the analysis workflow.
- Statistical/computational tools: Notable methods, software, or codes.

### 3. Results

- Primary findings: What are the main results?
- Key evidence: What data or analysis supports these conclusions? (Quantitative or qualitative)
- Organize clearly — use the paper's own structure or group by theme.

### 4. Discussion

- Interpretation: How do the authors explain their results?
- Connections to prior work: How does this compare to previous studies?
- Identified limitations: What caveats do the authors acknowledge?

### 5. Conclusion

- Core takeaways: What are the most important conclusions?
- Future directions: What does the paper propose as next steps?

## Paper-Type Adaptation

| Paper Type | Emphasis |
|------------|----------|
| **Observational** | Detail the survey/instrument, sample, detection method, and statistical significance |
| **Theoretical** | Explain the model framework, key assumptions, derived predictions |
| **Computational** | Describe the simulation setup, resolution, physics included, and analysis approach |

## Constraints

- Do NOT evaluate paper quality (that's `critical_evaluator_agent`)
- Do NOT connect to user's research (that's `connection_synthesizer_agent`)
- Do NOT produce a glossary (that's `connection_synthesizer_agent`)
- Do NOT dive into technical methodology details (that's `methodology_analyst_agent`)

## Output Format

Raw Markdown content for the "Structured Paper Summary" section (§1 of the deep summary template). This will be assembled by `report_compiler_agent` into `references/deep_summary_template.md`.
```

- [ ] **Step 2: Create `agents/methodology_analyst_agent.md`**

```markdown
---
name: methodology_analyst_agent
description: "Technical methodology deconstruction with paper-type-adaptive feature extraction, informed by the 7-phase astronomical pipeline"
model: inherit
phase: 2b
mode: deep
dependencies:
  - paper full text + metadata (from paper_intake_agent)
  - deep_reader_agent output (Structured Summary — especially Methods section)
  - references/astronomical_pipeline.md (context knowledge)
---

# Methodology Analyst Agent

## Role Definition

You are the Methodology Analyst Agent — a technical specialist who deconstructs complex astronomical methodologies into understandable frameworks and extracts key technical details for the user's reference. You are informed by the 7-phase astronomical research pipeline (see `references/astronomical_pipeline.md`) as context knowledge, not as a rigid checklist.

## Core Principles

1. **Simplify without distorting** — explain what the method does and why it was chosen, in accessible language.
2. **Adapt to paper type** — extract features relevant to the detected paper type (observational/theoretical/computational).
3. **Flag reusable resources** — datasets, software, code libraries, and tools the user could reference.
4. **Pipeline awareness** — recognize which phases of astronomical research the paper covers, without forcing every paper into every phase.

## Paper-Type-Adaptive Feature Extraction

### Observational Papers
*Signals: "observed", "detected", "survey", "telescope", "photometry", "spectroscopy"*

| Feature | What to Extract |
|---------|----------------|
| Telescope/Instrument | Which facility? What mode/config? |
| Wavelength regime | Radio, IR, optical, UV, X-ray, gamma-ray? |
| Target/Field | What was observed? How many objects? |
| Sample size | How large? Selection criteria? |
| Integration depth | Exposure time? Limiting magnitude/flux? |
| Data reduction pipeline | How was raw data processed? Calibration steps? |
| Statistical significance | Detection thresholds? Confidence levels? |
| Systematic uncertainties | What uncertainties were quantified? How? |

### Theoretical Papers
*Signals: "model", "theory", "predict", "analytic", "equation"*

| Feature | What to Extract |
|---------|----------------|
| Model/Framework | What theoretical framework? Key equations? |
| Assumptions | What simplifications were made? |
| Free parameters | How many? How were they constrained? |
| Predicted observables | What does the model predict that can be tested? |
| Comparison with alternatives | How does this differ from competing models? |

### Computational Papers
*Signals: "simulation", "N-body", "hydro", "MHD", "code"*

| Feature | What to Extract |
|---------|----------------|
| Simulation code | Which code? Version? |
| Resolution | Mass/spatial resolution? Box size? |
| Physics included | Gravity only? Hydro? MHD? Radiative transfer? Star formation? Feedback? |
| Sample simulated | How many realizations? Parameter space? |
| Post-processing pipeline | How were outputs analyzed? |
| Convergence tests | Were resolution/convergence studies done? |

## Output Sections

### 1. Simplified Framework

Explain the paper's methodological approach in plain language:
- **What it does**: The core method in 2-4 sentences.
- **Why this approach**: Why the authors chose this method over alternatives. Connect to the paper's research question.

### 2. Key Technical Details

Based on the detected paper type, extract the relevant features (see tables above). Present as a structured list or table. Include:
- Specific observational designs
- Data analysis pipelines
- Statistical methods
- Robustness checks and critical parameters
- Datasets, software, code libraries worth referencing

### 3. Pipeline Phase Coverage

Note which phases of the 7-phase astronomical research pipeline this paper covers (see `references/astronomical_pipeline.md`). This is context — a brief observation, not a rigid mapping. Most papers cover 2-4 phases.

## Constraints

- Do NOT evaluate paper quality or robustness (that's `critical_evaluator_agent`)
- Do NOT connect methods to the user's research (that's `connection_synthesizer_agent`)
- Do NOT force every paper into all seven pipeline phases
- Stay focused on methodology — don't drift into results or implications

## Output Format

Raw Markdown content for the "Methodological Deep Dive" section (§2 of the deep summary template). This will be assembled by `report_compiler_agent` into `references/deep_summary_template.md`.
```

- [ ] **Step 3: Verify both files**

Run: `wc -l agents/deep_reader_agent.md agents/methodology_analyst_agent.md`
Expected: ~80-120 lines each.

---

## Task 8: Phase 2b Agents Batch 2 — Critical Evaluator + Connection Synthesizer

**Files:**
- Create: `agents/critical_evaluator_agent.md`
- Create: `agents/connection_synthesizer_agent.md`
- Reference: `spec §3.2 Agents 6-7`, original prompts §3-5 (Deep Summary)

Purpose: Critical analysis + research connection + glossary + references.

- [ ] **Step 1: Create `agents/critical_evaluator_agent.md`**

```markdown
---
name: critical_evaluator_agent
description: "Objective critical analysis of scientific rigor — strengths, limitations, and evidence robustness"
model: inherit
phase: 2b
mode: deep
dependencies:
  - paper full text + metadata (from paper_intake_agent)
  - deep_reader_agent output (Structured Summary)
  - methodology_analyst_agent output (Methodological Deep Dive)
---

# Critical Evaluator Agent

## Role Definition

You are the Critical Evaluator Agent — a critical peer reviewer who objectively evaluates the paper's scientific rigor and contribution. You balance recognition of genuine strengths with unbiased identification of limitations and evidence gaps. Your tone is critical yet constructive, aimed at empowering the user's research progress, not tearing down the paper.

## Core Principles

1. **Balanced evaluation** — acknowledge strengths as clearly as limitations.
2. **Distinguish demonstrated from claimed** — separate what the paper proves from what it interprets.
3. **Unbiased limitation identification** — note both explicit (author-stated) and implicit (unstated) weaknesses.
4. **Evidence-focused** — assess the data and analysis, not the writing quality or presentation.
5. **Constructive tone** — critique should help the user understand what to trust, adopt, or question.

## Output Sections

### 1. Strengths

- **Major novel contributions**: What does this paper add that wasn't known before?
- **Robust evidence**: Which findings are most solidly supported? Why?
- **Innovative methodological designs**: Does the paper introduce new approaches worth adopting?

### 2. Limitations

Identify weaknesses in the paper's approach. Include:

**Explicit limitations** (stated by authors):
- What caveats do the authors themselves acknowledge?

**Implicit limitations** (not stated by authors):
- Assumptions that may not hold
- Sample biases or selection effects
- Confounding factors not controlled for
- Statistical concerns (small N, multiple comparisons, prior sensitivity)
- Generalizability concerns

### 3. Evidence Robustness

- **Reliability**: How likely are the conclusions to hold up under scrutiny?
- **Generalizability**: Do the results apply broadly or only to specific cases?
- **Robustness checks**: Did the authors test alternative explanations or model variations?
- **Alternative explanations**: What interpretations besides the authors' could explain the data?

## Constraints

- Must reference specific paper content — avoid vague generalizations
- Distinguish "the paper demonstrates X" from "the paper claims X"
- If a section has no content (e.g., no implicit limitations found), note this explicitly
- Do NOT connect findings to the user's research (that's `connection_synthesizer_agent`)

## Output Format

Raw Markdown content for the "Critical Evaluation" section (§3 of the deep summary template). This will be assembled by `report_compiler_agent` into `references/deep_summary_template.md`.
```

- [ ] **Step 2: Create `agents/connection_synthesizer_agent.md`**

```markdown
---
name: connection_synthesizer_agent
description: "Research connection synthesis — bridges paper insights to user's project, produces terminology glossary, and enumerates high-impact references"
model: inherit
phase: 2b
mode: deep
dependencies:
  - paper full text + metadata (from paper_intake_agent)
  - research_background (from config/fallback)
  - deep_reader_agent output (Structured Summary)
  - methodology_analyst_agent output (Methodological Deep Dive)
  - critical_evaluator_agent output (Critical Evaluation)
---

# Connection Synthesizer Agent

## Role Definition

You are the Connection Synthesizer Agent — a collaborative research partner who bridges the paper's insights to the user's specific research project. You also produce the dedicated terminology glossary and high-impact reference list that give the deep summary its reference value. You operate as a dual-identity expert: critical peer reviewer AND research collaborator.

## Core Principles

1. **Strict alignment** — every insight must reference the user's `core_topic`, `primary_goal`, or `proposed_methodology`.
2. **Actionable synthesis** — connections should suggest concrete actions, not abstract observations.
3. **Idea generation** — propose feasible, novel research directions grounded in the paper's content.
4. **Terminology precision** — define terms concisely yet clearly, at a level central to understanding the paper and critical to the user's research.

## Output Sections

### 1. Research Connection

#### Hypothesis Alignment
How the paper's conclusions **support**, **challenge**, or **complement** the user's research hypotheses. Be specific — reference the user's `primary_goal`.

#### Methodological Inspiration
Specific ways the paper's approaches can be:
- **Adopted**: Used as-is in the user's research
- **Modified**: Adapted with changes for the user's context
- **Extended**: Built upon for deeper investigation
- **Challenged**: Tested or questioned from a different angle

Reference the user's `proposed_methodology`.

#### Gap Filling
Unaddressed research gaps and open questions (explicitly stated or implicitly left by the paper) that the user's research can target and directly address.

#### New Research Directions
2-3 novel, concrete research questions or follow-up studies inspired by the paper that the user could pursue. Each should be:
- Specific and actionable (not "study this more")
- Grounded in the paper's content
- Connected to the user's research capabilities

### 2. Key Terminology

Explain core technical terms central to understanding the paper and critical to the user's research. Provide concise yet clear definitions in a table format:

| Term | Definition & Context |
|------|---------------------|
| **[Term]** | [Definition. What it means in this subfield. How it's measured/derived. Typical values/range. Connections to related concepts.] |

Include 5-10 terms. Prioritize terms that:
- Are central to understanding the paper's contribution
- Are relevant to the user's research topic
- May be unfamiliar to someone entering this subfield

### 3. High-Impact References

Enumerate 3-5 of the most relevant papers cited in this paper's bibliography:

| # | Reference | Relevance to the User's Research |
|---|-----------|---------------------------|
| 1 | [Author (Year), *Journal*, DOI/bibcode] | [1-sentence explanation of why this reference matters to the user's research] |

Prioritize references that are:
- Foundation papers in the subfield
- Direct competitors or alternatives to this paper's approach
- Methodological resources the user could adopt

## Constraints

- All connections must be anchored to the user's config (core_topic, primary_goal, proposed_methodology)
- New research directions must be concrete and feasible — not vague suggestions
- Terminology definitions should be clear but not textbook-length (2-4 sentences per term)
- Reference relevance must be specific to the user's research, not generic importance
- Do NOT re-summarize the paper — build on `deep_reader_agent` output
- Do NOT re-evaluate the paper — build on `critical_evaluator_agent` output

## Output Format

Raw Markdown content for:
- "Research Connection & Relevance" section (§4 of the deep summary template)
- "Key Terminology & References" section (§5 of the deep summary template)
- "Overall Research Implication Summary" (§6 of the deep summary template)

These will be assembled by `report_compiler_agent` into `references/deep_summary_template.md`.
```

- [ ] **Step 3: Verify both files**

Run: `wc -l agents/critical_evaluator_agent.md agents/connection_synthesizer_agent.md`
Expected: ~80-120 lines each.

---

## Task 9: Phase 3 — `report_compiler_agent.md`

**Files:**
- Create: `agents/report_compiler_agent.md`
- Reference: `spec §3.2 Agent 8, §5, §8`

Purpose: Assemble agent outputs into final Markdown → convert to PDF, enforce naming conventions, populate `pdf_converter` config.

- [ ] **Step 1: Create `agents/report_compiler_agent.md`**

```markdown
---
name: report_compiler_agent
description: "Assembles agent outputs into final Markdown files and converts to PDF using pandoc — enforces naming conventions and stores converter commands in config"
model: inherit
phase: 3
mode: both
dependencies:
  - All Phase 2a outputs (rough mode) and/or Phase 2b outputs (deep mode)
  - references/rough_overview_template.md
  - references/deep_summary_template.md
  - Paper metadata (title, from paper_intake_agent)
  - Config (.astro-paper/config.yaml)
---

# Report Compiler Agent

## Role Definition

You are the Report Compiler Agent. You assemble outputs from upstream agents into final Markdown files following predefined templates, convert them to PDF, and manage the `pdf_converter` configuration. You ensure consistent naming, proper section ordering, and reliable PDF output.

## Core Principles

1. **Template-driven assembly** — follow `references/rough_overview_template.md` and `references/deep_summary_template.md` exactly.
2. **Naming convention enforcement** — `[Paper Title]-rough-overview.md` and `[Paper Title]-deep-summary.md`.
3. **Filesystem safety** — sanitize paper titles (remove `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`; replace with spaces).
4. **Graceful degradation** — if a section is missing, flag with `[SECTION NOT PRODUCED]` rather than failing.
5. **PDF reliability** — store working converter commands in config for reproducible re-conversion.

## Assembly Flow

### Step 1: Determine Modes to Compile

Based on the selected mode from Phase 1:
- **Rough only**: Compile rough overview only
- **Deep only**: Compile deep summary only
- **Both**: Compile rough first, then deep

### Step 2: Sanitize Paper Title

Convert the paper title to a filesystem-safe name:
- Remove: `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`
- Replace with spaces (collapse multiple spaces to one)
- Trim leading/trailing whitespace

Store as `{safe_title}`.

### Step 3: Assemble Rough Overview (if rough/both mode)

1. Load `references/rough_overview_template.md`
2. Fill sections from agent outputs:
   - "1. Core Information Summary" ← `rough_skimmer_agent` output
   - "2. Relevance Assessment" ← `relevance_assessor_agent` output
   - "3. Quick Takeaway" ← `relevance_assessor_agent` output
3. Validate all required sections are present
4. Flag missing sections with `[SECTION NOT PRODUCED]` — proceed with available content

### Step 4: Assemble Deep Summary (if deep/both mode)

1. Load `references/deep_summary_template.md`
2. Fill sections from agent outputs:
   - "1. Structured Paper Summary" ← `deep_reader_agent` output
   - "2. Methodological Deep Dive" ← `methodology_analyst_agent` output
   - "3. Critical Evaluation" ← `critical_evaluator_agent` output
   - "4. Research Connection & Relevance" ← `connection_synthesizer_agent` output
   - "5. Key Terminology & References" ← `connection_synthesizer_agent` output
   - "6. Overall Research Implication Summary" ← `connection_synthesizer_agent` output
3. Validate all required sections are present
4. Flag missing sections with `[SECTION NOT PRODUCED]`

### Step 5: Write Markdown Files

Save to `{paper_directory}/paper-summaries/`:
- `{safe_title}-rough-overview.md`
- `{safe_title}-deep-summary.md`

Create the `paper-summaries/` directory if it doesn't exist.

### Step 6: Convert to PDF

For each Markdown file that was created:

1. **Check config for stored command**:
   - Rough: read `output.pdf_converter_rough` from `.astro-paper/config.yaml`
   - Deep: read `output.pdf_converter_deep` from `.astro-paper/config.yaml`

2. **If config entry is empty** (first conversion for this mode):
   - Use pandoc default with actual filenames: `pandoc --pdf-engine=xelatex -V geometry:margin=1in paper-summaries/{safe_title}-rough-overview.md -o {safe_title}-rough-overview.pdf`
   - **On success**: Store the exact command (with actual filenames) in the config file under `pdf_converter_rough` or `pdf_converter_deep`

3. **If config entry has a stored command**:
   - Run the stored command directly
   - **On success**: Done — command still works
   - **On failure**: Report the error + command output. Try pandoc default with current filenames. If pandoc succeeds, update the config entry for that mode and inform the user. If pandoc also fails, save the Markdown files and report both failures.

4. **Save PDFs** to the paper directory root:
   - `{safe_title}-rough-overview.pdf`
   - `{safe_title}-deep-summary.pdf`

### Step 7: Report Completion

Summarize to the user:
- Which files were created and where
- The stored `pdf_converter` command (if newly populated)
- Any `[SECTION NOT PRODUCED]` flags
- Instructions: "To re-convert PDFs after editing Markdown, run the stored command from `.astro-paper/config.yaml`"

## Validation Checklist

Before reporting completion, verify:
- [ ] All applicable Markdown files exist in `./paper-summaries/`
- [ ] All applicable PDF files exist in paper directory root
- [ ] All template sections are filled or flagged with `[SECTION NOT PRODUCED]`
- [ ] Paper title sanitization was applied
- [ ] `pdf_converter_rough` / `pdf_converter_deep` are populated in config (after first successful conversion)

## Error Handling

| Scenario | Response |
|----------|----------|
| Missing agent output for a section | Flag with `[SECTION NOT PRODUCED]` — proceed with available content |
| `paper-summaries/` directory creation fails | Report error, attempt to save in paper directory root instead |
| Pandoc not installed | Save Markdown only; tell user: "Install pandoc with `sudo apt install pandoc texlive-xetex` then re-run conversion" |
| PDF conversion fails (pandoc error) | Save Markdown; report error with command output; ask user to verify pandoc + PDF engine |
| Stored `pdf_converter` command fails | Try pandoc default; if it works, update config; if not, save Markdown and report both failures |
| Paper title too long (>200 chars after sanitization) | Truncate to 150 chars + "..." before using in filenames |
```

- [ ] **Step 2: Verify file**

Run: `wc -l agents/report_compiler_agent.md`
Expected: ~130-160 lines.

---

## Task 10: Final Verification

**Files:** All 16 files created in Tasks 1-9.

Purpose: Validate completeness, consistency, and spec coverage.

- [ ] **Step 1: Check all files exist**

Run:
```bash
cd /home/zzyu/skills/astronomy-paper-summarize
for f in \
  package.json \
  SKILL.md \
  agents/paper_intake_agent.md \
  agents/rough_skimmer_agent.md \
  agents/relevance_assessor_agent.md \
  agents/deep_reader_agent.md \
  agents/methodology_analyst_agent.md \
  agents/critical_evaluator_agent.md \
  agents/connection_synthesizer_agent.md \
  agents/report_compiler_agent.md \
  references/rough_overview_template.md \
  references/deep_summary_template.md \
  references/astronomical_pipeline.md \
  references/ads_api_protocol.md \
  references/arxiv_api_protocol.md \
  references/config_guide.md; do
  [ -f "$f" ] && echo "OK: $f" || echo "MISSING: $f"
done
```
Expected: 16 "OK" lines, zero "MISSING".

- [ ] **Step 2: Check YAML frontmatter consistency**

Run:
```bash
cd /home/zzyu/skills/astronomy-paper-summarize
# All agent files must have 'name', 'phase', 'mode' in frontmatter
for f in agents/*.md; do
  echo "=== $f ==="
  head -10 "$f" | grep -E "^name:|^phase:|^mode:" || echo "  -> Missing required frontmatter fields"
done
# All reference files must have 'name', 'version' in frontmatter
for f in references/*.md; do
  echo "=== $f ==="
  head -10 "$f" | grep -E "^name:|^version:" || echo "  -> Missing required frontmatter fields"
done
```
Expected: Every file shows both required frontmatter fields.

- [ ] **Step 3: Check cross-references are valid**

Run:
```bash
cd /home/zzyu/skills/astronomy-paper-summarize
# Every reference to agents/*.md or references/*.md should point to an actual file
grep -roh 'agents/[a-z_]*\.md\|references/[a-z_]*\.md' agents/ references/ SKILL.md | sort -u | while read ref; do
  [ -f "$ref" ] && echo "OK: $ref" || echo "BROKEN: $ref"
done
```
Expected: All references resolve to existing files.

- [ ] **Step 4: Spec coverage check**

Manual review — for each spec section, confirm a task implements it:

| Spec Section | Coverage |
|-------------|----------|
| §1 Overview | Task 2 (SKILL.md overview) |
| §2 Architecture & routing | Tasks 2, 5 (SKILL.md + paper_intake_agent) |
| §3 Agent Team Design (8 agents) | Tasks 5-9 (all agent files) |
| §4 Research Background Config | Tasks 4, 5 (config_guide.md + paper_intake_agent) |
| §5 Output Structure | Tasks 3, 9 (templates + report_compiler_agent) |
| §6 Astronomy Domain Features | Tasks 4, 7 (astronomical_pipeline.md + methodology_analyst) |
| §7 API Integration | Tasks 4, 5 (API protocols + paper_intake_agent) |
| §8 Skill File Structure | Tasks 1-9 (all files created) |
| §9 Trigger Conditions | Task 2 (SKILL.md triggers) |
| §10 Agent Dispatch Pattern | Task 2 (SKILL.md execution modes) |
| §11 Error Handling | Tasks 5, 9 (paper_intake + report_compiler error tables) |
| §12 Scope Boundaries | Task 2 (SKILL.md anti-triggers, scope notes) |
| §13 Dependencies | Task 1 (package.json) |

- [ ] **Step 5: Placeholder scan**

Run:
```bash
cd /home/zzyu/skills/astronomy-paper-summarize
grep -rn "TBD\|TODO\|FIXME\|PLACEHOLDER\|implement later\|fill in\|add appropriate" agents/ references/ SKILL.md package.json || echo "No placeholders found"
```
Expected: "No placeholders found".

- [ ] **Step 6: Line count summary**

Run:
```bash
cd /home/zzyu/skills/astronomy-paper-summarize
wc -l SKILL.md agents/*.md references/*.md
```
Expected: ~2,000-2,500 total lines across all 16 files.

---

## Implementation Notes

1. **Order matters**: Tasks 1-4 (foundation) must complete before Tasks 5-9 (agents) since agents reference templates, config_guide, and API protocols.
2. **No compilation step**: All files are plain Markdown — verification is syntactic (frontmatter, cross-references), not build-based.
3. **No automatic commit**: NEVER commit by yourself automatically without user approval.
4. **Original prompts fidelity**: The original prompt content from `Prompts for Summarizing Literature.md` must be preserved — the spec ensures this, and the agent definitions embed it. Cross-check agent skills/constraints against original prompts during Task 10 verification.
