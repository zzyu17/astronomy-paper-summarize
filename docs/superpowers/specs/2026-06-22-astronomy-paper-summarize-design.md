# Astronomy Paper Summarize — Design Specification

**Date**: 2026-06-23
**Status**: Draft (awaiting review)
**Version**: 1.1.0

---

## 1. Overview

`astronomy-paper-summarize` is a skill that transforms astronomy research papers into structured, actionable summaries. It offers two summarization modes — **Rough Overview** (rapid triage) and **Deep Summary** (comprehensive critical analysis) — and two execution modes — **Subagent-Driven** (fresh sub-agent per agent, recommended) and **Inline Execution** (sequential checklist in-session) — powered by an 8-agent team whose behavior is guided by the user's own research background.

The skill is designed for astronomers and astrophysics researchers who need to efficiently process large volumes of literature, separating signal from noise and connecting each paper's insights back to their own research project.

### 1.1 Source Material

The skill is derived from two prompt sets in `Prompts for Summarizing Literature.md`:

| Prompt Set | Purpose | Core Philosophy |
|------------|---------|-----------------|
| **Rough Overview** | Rapid literature triage — extract core information (research question, methodology, 1-3 key findings, relevance) in minutes | Brevity, relevance-first, jargon simplification |
| **Deep Summary** | Comprehensive critical analysis — methodological deep dive, critical evaluation, research connection, terminology glossary, new direction generation | Rigor, synthesis, actionable insights |

All key information from both prompt sets is preserved in the skill design. The skill extends them with a multi-agent architecture, persistent configuration, paper-type-adaptive analysis, and cross-agent portability.

### 1.2 Design Philosophy

- **Follow ARS patterns**: Multi-agent team with phase boundaries, agent-to-agent handoffs, frontmatter metadata, and reference-document separation — matching the architecture of `academic-research-skills`
- **User's research as lens**: Every analysis is filtered through the user's own research topic, goal, and methodology (stored in persistent YAML config), ensuring summaries are always targeted rather than generic
- **Astronomy-native, not astronomy-hardcoded**: Domain features (e.g., astronomical research pipeline phases, terminology glossaries, paper-type detection) are presented as **context knowledge** for agents to adapt per-paper, rather than rigid mapping requirements
- **Portable across AI agents**: Config stored as plain YAML files, output as standard Markdown — readable by Copilot CLI, Claude Code, Codex, or any LLM-based agent

---

## 2. Architecture

### 2.1 High-Level Flow

```
User paper + config
        │
        ▼
┌─────────────────────────────────┐
│ Paper Intake Agent              │
│ • Load/create config            │
│ • Mode: rough / deep / both     │
│ • Execution: subagent / inline  │
└──────┬──────────────────────────┘
       │
       ├── Rough Mode ──────────────────────────┐
       │                                         │
       ▼                                         ▼
┌──────────────┐  ┌──────────────────┐  ┌─────────────────────┐
│ Rough        │  │ Relevance        │  │ Deep Reader         │
│ Skimmer      │  │ Assessor         │  │ Agent               │
│ Agent        │  │ Agent            │  │ (Structured Summary) │
└──────┬───────┘  └────────┬─────────┘  └──────────┬──────────┘
       │                    │                       │
       │                    │              ┌────────┴────────┐
       │                    │              ▼                 ▼
       │                    │    ┌─────────────────┐  ┌──────────────────┐
       │                    │    │ Methodology     │  │ Critical         │
       │                    │    │ Analyst Agent   │  │ Evaluator Agent  │
       │                    │    └────────┬────────┘  └────────┬─────────┘
       │                    │             │                     │
       │                    │             └──────┬──────────────┘
       │                    │                    ▼
       │                    │    ┌──────────────────────────┐
       │                    │    │ Connection Synthesizer   │
       │                    │    │ Agent (Glossary + Gaps)  │
       │                    │    └────────────┬─────────────┘
       │                    │                 │
       └────────────────────┴─────────────────┘
                            │
             ┌──────────────┴──────────────┐
             │  Execution Mode Selected     │
             │  by User in Phase 1          │
             ├──────────────┬──────────────┤
             │ Subagent     │ Inline       │
             │ (dispatch    │ (sequential  │
             │  sub-agents) │  checklist)  │
             └──────┬───────┴──────┬───────┘
                    │              │
                    ▼              ▼
             ┌──────────────────────────┐
             │ Report Compiler Agent    │
             │ (MD → PDF)               │
             └──────────────────────────┘
```

### 2.2 Mode Routing

The `paper_intake_agent` serves as the unified entry point that handles **research background configuration**, **mode selection**, and **execution mode selection**:

1. **Check for existing config**: Look for `.astro-paper/config.yaml` in the **parent directory of the paper project directory** (see §4.1)
2. **If config missing or stale**: Prompt user for Core Research Topic, Primary Goal, Proposed Methodology (3 fields)
3. **Mode selection**: Ask user to choose `rough`, `deep`, or `both`
4. **Execution mode selection**: Ask user to choose **Subagent-Driven** (dispatch each agent as a fresh sub-agent) or **Inline Execution** (process agents sequentially in the current session)
5. **Generic fallback** (if user skips config): Skim paper abstract to auto-extract topic/goal/methodology, present to user for refinement or approval, then proceed
6. **Dispatch**: Route to the appropriate agent pipeline based on mode + execution mode selection

The skill uses standard `Skill` tool invocation — no Copilot CLI slash commands or `extension.mjs` needed. The intake agent handles all routing internally.

#### Execution Mode Options

| Mode | Mechanism | Pros | Cons |
|------|-----------|------|------|
| **Subagent-Driven** (default) | Each agent dispatched as a fresh sub-agent with isolated context | Clean context per agent; parallel-safe; better for large papers | Higher latency (sub-agent startup); more token usage |
| **Inline Execution** | Orchestrator reads each agent template and executes instructions sequentially in the current session | Lower latency; fewer context switches; better for short papers | Shared context window; no parallelism; risk of context overflow on long papers |

### 2.3 Phase Boundaries

| Phase | Mode | Agents | Deliverable |
|-------|------|--------|-------------|
| **Phase 1 — Intake** | Both | `paper_intake_agent` | Config loaded/created, mode + execution mode selected, paper metadata extracted |
| **Phase 2a — Rough** | Rough | `rough_skimmer_agent` → `relevance_assessor_agent` | Rough Overview Markdown content |
| **Phase 2b — Deep** | Deep | `deep_reader_agent` → `methodology_analyst_agent` → `critical_evaluator_agent` → `connection_synthesizer_agent` | Deep Summary Markdown content |
| **Phase 3 — Compile** | Both | `report_compiler_agent` | Final Markdown files + PDF conversion |

When running `both` mode: Phase 2a runs first (rough overview for quick triage), then Phase 2b (deep analysis for full understanding), then Phase 3 compiles both.

**Execution mode affects how agents in Phases 2-3 are invoked**:
- **Subagent-Driven**: Each agent is dispatched as an isolated sub-agent. After each sub-agent completes, the orchestrator collects output and passes to the next phase.
- **Inline Execution**: The orchestrator reads each agent template from `agents/` and works through its instructions sequentially in the current session. Agent templates serve as structured checklists rather than requiring process isolation.

---

## 3. Agent Team Design (8 Agents)

### 3.1 Agent Roster

| # | Agent | Role | Phase | Mode |
|---|-------|------|-------|------|
| 1 | `paper_intake_agent` | Research background config + paper metadata extraction + mode routing | Phase 1 | Both |
| 2 | `rough_skimmer_agent` | Rapid skimming, core information extraction, inline jargon simplification | Phase 2a | Rough |
| 3 | `relevance_assessor_agent` | Multi-dimensional relevance scoring, project implications, takeaway synthesis | Phase 2a | Rough |
| 4 | `deep_reader_agent` | Structured paper summary (Introduction→Methods→Results→Discussion→Conclusion) following Astrobites template | Phase 2b | Deep |
| 5 | `methodology_analyst_agent` | Technical methodology deconstruction, paper-type-adaptive feature extraction | Phase 2b | Deep |
| 6 | `critical_evaluator_agent` | Critical analysis: strengths, limitations, evidence robustness, biases | Phase 2b | Deep |
| 7 | `connection_synthesizer_agent` | Research connection + dedicated terminology glossary + high-impact references | Phase 2b | Deep |
| 8 | `report_compiler_agent` | Markdown assembly → PDF conversion, naming convention enforcement | Phase 3 | Both |

### 3.2 Agent Details

#### Agent 1 — `paper_intake_agent`

**Role**: Unified entry point combining research background management, paper content acquisition, and mode routing.

**Paper Content Acquisition Strategy**:

The agent uses a **hybrid approach** — ADS API for structured metadata, combined with direct paper content from the user:

| Method | Pros | Cons |
|--------|------|------|
| **Direct PDF reading** | Full text available; no API dependency; works offline | PDF text extraction can fail on scanned/corrupted files; unstructured |
| **ADS API** | Rich structured metadata (title, authors, abstract, citations, bibcode, arXiv class); reliable; enables reference lookups | Requires API token; rate limited (5,000/day); abstract only (no full text); network dependency |
| **arXiv API** | Free access; full-text often available; covers most astro-ph papers | XML Atom format; rate-limit ~3s/request; no citation data |

**Recommended hybrid flow**:
1. If user provides arXiv ID or paper title → query ADS API for structured metadata (bibcode, citation count, abstract, keywords, arXiv category)
2. If ADS unavailable → fall back to arXiv API for metadata + abstract
3. If both APIs unavailable → extract metadata from user-provided PDF/text or manual input
4. The actual paper **full text** always comes from the user's provided file (PDF, text, or pasted content) — APIs supplement with metadata, not content

**Responsibilities**:
- Check for `.astro-paper/config.yaml` in the **parent directory of paper project directory**; load if exists
- If config missing or user requests update, prompt for 3 fields: Core Research Topic, Primary Goal, Proposed Methodology
- If user skips config: enter **Generic Fallback Mode** — extract approximate topic/goal/methodology from paper abstract, present for user approval/refinement (auto-extracted values are saved to config after user approves/refines)
- Present mode selection: `rough`, `deep`, or `both`
- Present execution mode selection: **Subagent-Driven** (default) or **Inline Execution**
- Extract paper metadata using the ADS → arXiv → manual fallback chain
- Dispatch to Phase 2a (rough), Phase 2b (deep), or both, using the chosen execution mode

**Input**: User-provided paper (path/arXiv ID/text/URL), `.astro-paper/config.yaml` (optional)
**Output**: Loaded config, selected mode + execution mode, paper metadata + full text

#### Agent 2 — `rough_skimmer_agent`

**Role**: Rapid information extraction for literature triage.

**Responsibilities**:
- Skim paper to extract: Research Question, Methodology (high-level), 1-3 Core Findings
- Identify authors' stated limitations/future work if explicitly present
- Translate technical terminology into more understandable concepts where possible, without losing essential meaning (inline in the summary text)

**Constraints**:
- Brevity is paramount — bullet points, short sentences
- No deep methodological explanation
- No dedicated terminology glossary (that's deep mode only)
- Explicit information only — don't infer limitations the authors don't state

#### Agent 3 — `relevance_assessor_agent`

**Role**: Multi-dimensional relevance scoring against user's research project.

**Responsibilities**:
- Score paper relevance on 1-10 scale using weighted 5-factor framework:

| Dimension | Weight | Assessment Method |
|-----------|--------|-------------------|
| Topic Match | 35% | Keyword/title overlap with user's research topic, arXiv category match |
| Methodology Match | 25% | Same telescope/instrument, same wavelength regime, same analysis method |
| Target Match | 20% | Same object class, same redshift range, same stellar population |
| Impact | 10% | Citation count, journal tier, author prominence |
| Timeliness | 10% | Recency, connection to active debates |

- Scoring bands: 9-10 (direct match), 7-8 (related subfield), 5-6 (overlapping), 1-4 (not relevant)
- Generate 1-3 Project Implications — concrete takeaways for user's research
- Classify as Must-Read / Maybe / Not-Essential
- Produce single-sentence Quick Takeaway

#### Agent 4 — `deep_reader_agent`

**Role**: Structured comprehensive summary following the Astrobites 14-year-validated template.

**Responsibilities**:
- Produce a 5-section structured summary:
  1. **Introduction**: Research background, key questions, objectives
  2. **Methods**: Core methodological framework, data sources, sample selection, analysis pipelines, statistical/computational tools
  3. **Results**: Primary findings and key evidence (quantitative or qualitative) supporting conclusions
  4. **Discussion**: Interpretation of results, connections to prior work, identified limitations
  5. **Conclusion**: Core takeaways, proposed future research directions
- Follow Astrobites pattern: engaging lede → background → methods → results → implications
- Adapt level of detail based on paper type (observational/theoretical/computational)
- Maintain technical accuracy without misrepresentation

#### Agent 5 — `methodology_analyst_agent`

**Role**: Deep technical methodology deconstruction with paper-type-adaptive feature extraction.

**Responsibilities**:
- Produce simplified explanation of the paper's methodological framework: what it does and why chosen over alternatives
- Extract key technical details based on detected paper type:

| Paper Type | Detection Signals | Key Features to Extract |
|------------|-------------------|------------------------|
| **Observational** | "observed", "detected", "survey", "telescope" | Telescope/Instrument, wavelength regime, target/field, sample size, integration depth, data reduction pipeline, statistical significance, systematic uncertainties |
| **Theoretical** | "model", "theory", "predict", "analytic" | Model/framework, key equations, assumptions made, free parameters, predicted observables, comparison with alternatives |
| **Computational** | "simulation", "N-body", "hydro", "MHD" | Simulation code, resolution, physics included, sample simulated, post-processing pipeline, convergence tests |

- Reference the 7-phase astronomical pipeline as context knowledge (see §6.1), recognizing which phases the paper covers
- Flag datasets, software, code libraries worth referencing
- Note robustness checks and critical parameters

**Phase Boundary**: Single-phase agent — produces only the Methodological Deep Dive section. Must not drift into critical evaluation or research connection.

#### Agent 6 — `critical_evaluator_agent`

**Role**: Objective critical analysis of the paper's scientific rigor and contribution.

**Responsibilities**:
- Evaluate **Strengths**: Major novel contributions, robust evidence, innovative methodological designs
- Identify **Limitations**: Explicit (authors' stated) and implicit (unstated) weaknesses in methodology, assumptions, data, or interpretation
- Assess **Evidence Robustness**: Reliability and generalizability of conclusions, adequacy of robustness checks, alternative explanations considered
- Separate what the paper demonstrates from what it claims — distinguish demonstrated findings from interpretations
- Maintain constructive tone: critical but aimed at empowering research progress

#### Agent 7 — `connection_synthesizer_agent`

**Role**: Bridge the paper to the user's research project and produce dedicated terminology resources.

**Responsibilities**:

**Research Connection**:
- **Hypothesis Alignment**: How the paper's conclusions support, challenge, or complement user's research hypotheses
- **Methodological Inspiration**: Specific ways the paper's approaches can be adopted, modified, extended, or challenged
- **Gap Filling**: Unaddressed research gaps that the user's project can target
- **New Directions**: 2-3 novel, concrete research questions or follow-up studies

**Key Terminology & References**:
- Explain core technical terms central to understanding the paper and critical to the user's research with concise yet clear definitions
- Enumerate 3-5 high-impact references from the paper's bibliography with 1-sentence explanation of their relevance to the user's research topic

#### Agent 8 — `report_compiler_agent`

**Role**: Assemble agent outputs into final Markdown files and convert to PDF.

**Responsibilities**:
- Assemble rough overview content into `[Paper Title]-rough-overview.md`
- Assemble deep summary content into `[Paper Title]-deep-summary.md`
- Save Markdown files under `./paper-summaries/` subdirectory of the paper's directory
- Convert Markdown to PDF:
  - If `pdf_converter_rough` / `pdf_converter_deep` exists in config, use the stored command
  - If config entry is empty or stored command fails, use `pandoc` as default: `pandoc --pdf-engine=xelatex -V geometry:margin=1in <markdown-file> -o <pdf-file>`
  - After **first successful conversion** of each mode, store the working command in `.astro-paper/config.yaml` under `pdf_converter_rough` or `pdf_converter_deep`
  - If a previously-stored command fails on a subsequent run, try out a corrected command, update the config entry for that mode (rough/deep) and prompt the user
- Place PDFs in the paper directory root (alongside original paper): `./[Paper Title]-rough-overview.pdf` and `./[Paper Title]-deep-summary.pdf`
- Validate that all required sections are present in each output
- Flag any missing sections with `[SECTION NOT PRODUCED]` markers

---

## 4. Research Background Configuration

### 4.1 Config File

**Location**: `.astro-paper/config.yaml` under the **parent directory of paper project directory**. This ensures that the configuration is shared across all papers in the same research project.

**Schema**:
```yaml
# Astronomy Paper Summarize — Research Background Configuration
# Last updated: YYYY-MM-DD

research_background:
  core_topic: "Planets in Neptunian Desert"
  primary_goal: "To draw statistical patterns of the Neptunian desert and understand the formation and evolution mechanisms that create it"
  proposed_methodology: "Using large-scale observational data, particularly from TESS, to conduct statistical analyses of exoplanet populations"

output:
  pdf_converter_rough: ""
  pdf_converter_deep: ""
  # Populated AFTER first successful PDF conversion, not at config creation.
  # Two separate commands — one for rough overview, one for deep summary.
  # Updated on each subsequent conversion if the stored command breaks.
  # Stored with actual filenames for convenience.
  # Example: "pandoc --pdf-engine=xelatex -V geometry:margin=1in paper-summaries/My Paper-rough-overview.md -o My Paper-rough-overview.pdf"
```

**Initial state**: `pdf_converter_rough` and `pdf_converter_deep` are empty strings. They are populated by the `report_compiler_agent` after the first successful PDF conversion of each mode. If a stored command fails on a subsequent run, the agent tries out a corrected command, updates the config entry for that mode (rough/deep) and prompts the user.

### 4.2 Generic Fallback Mode

When the user skips background configuration:
1. `paper_intake_agent` skims the paper's abstract to extract approximate values for `core_topic`, `primary_goal`, and `proposed_methodology`
2. Presents the auto-extracted values to the user: "Based on your paper, I've inferred the following research background. Please refine or approve: ..."
3. User can refine (edit any field) or approve (accept as-is)
4. The approved values are used for all relevance scoring and research connections
5. The approved values are saved to `.astro-paper/config.yaml` in the parent directory of paper project directory for reuse with future papers
6. The user can update the config later within any summarization session

### 4.3 Cross-Agent Portability

The `.astro-paper/config.yaml` file is a plain YAML file with no tool-specific dependencies:
- **Copilot CLI**: Read via standard file read tools
- **Claude Code**: Read via Read tool
- **Codex**: Read via file read primitives
- **Any LLM agent**: Parse as standard YAML

The `pdf_converter_rough` and `pdf_converter_deep` fields are populated after first successful PDF conversion, not at config creation.

---

## 5. Output Structure

### 5.1 File Naming and Location

For a paper titled "The Neptunian Desert Across Stellar Types":

```
paper-directory/
├── paper.pdf                          # Original paper
├── The Neptunian Desert Across Stellar Types-rough-overview.pdf    # Rough overview PDF
├── The Neptunian Desert Across Stellar Types-deep-summary.pdf      # Deep summary PDF
└── paper-summaries/                   # Markdown source files
    ├── The Neptunian Desert Across Stellar Types-rough-overview.md
    └── The Neptunian Desert Across Stellar Types-deep-summary.md
```

**Rules**:
- Paper title is sanitized for filesystem safety (remove `/`, `\`, `:`, `*`, `?`, `"`, `<`, `>`, `|`; replace with space)
- Markdown files go in `./paper-summaries/` subdirectory (keeps source files organized)
- PDF files go in the paper directory root (alongside the original paper, for easy access)
- The `pdf_converter` command is stored in config for re-conversion if Markdown is modified afterwards

### 5.2 Rough Overview Sections

Based on the original prompt set, with refinements for clarity and structure:

```markdown
# [Paper Title] — Rough Overview

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

---

```

### 5.3 Deep Summary Sections

Based on the original prompt set, with refinements and the dedicated glossary:

```markdown
# [Paper Title] — Deep Summary

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
*(Adapted to detected paper type: observational / theoretical / computational)*

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
| **[Term 1]** | [Definition. What it means in this subfield. How it's measured. Typical values/range. Connections to related concepts. Notable uncertainties.] |
| **[Term 2]** | [...] |

### High-Impact References
| # | Reference | Relevance to the User's Research |
|---|-----------|---------------------------|
| 1 | [Author (Year), *Journal*, DOI] | [1-sentence explanation of relevance] |
| 2-5 | ... | ... |

## 6. Overall Research Implication Summary

[A concise, high-level paragraph summarizing how this paper's insights should influence the design, execution, and/or interpretation of the user's research]

---

```

---

## 6. Astronomy Domain Features

### 6.1 The 7-Phase Astronomical Research Pipeline

Presented as **context knowledge** for agents (especially `methodology_analyst_agent`), not as a rigid mapping. The pipeline describes how modern astronomical research typically flows, with rich feedback loops between all phases:

| Phase | Description | Key Activities |
|-------|-------------|---------------|
| 1. Theory & Physical Modeling | Mathematical/analytical framework | Deriving physical equations, building conceptual models (ΛCDM, core accretion, MRI) |
| 2. Numerical Simulation | Computational implementation | N-body, hydro, MHD, radiative transfer; simulation suites (e.g., CAMELS: 4,233 simulations) |
| 3. Observational Design & Proposal | Planning observations | Target selection, feasibility calculations, instrument selection, proposal writing |
| 4. Data Acquisition | Collecting data | Imaging, spectroscopy, interferometry, time-domain (TESS, JWST, ALMA, VLA) |
| 5. Data Reduction & Calibration | Processing raw data | Bias/flat correction, wavelength/flux calibration, source extraction |
| 6. Statistical Inference & Analysis | Extracting physical parameters | MCMC, nested sampling, SED fitting, ML classification, Bayesian inference |
| 7. Interpretation & Model Comparison | Connecting to physics | Bayes factors, posterior predictive checks, theory comparison |

**Key Insight**: All phases are bidirectionally connected — the pipeline is a web, not a line. Agents should recognize which phases a paper covers and use that awareness when explaining methods, without forcing every paper into all phases.

**Reference Document**: `references/astronomical_pipeline.md` — detailed explanation of each phase with subfield variations and feedback loops.

### 6.2 Jargon Simplification vs. Key Terminology Explanation

A clear separation between the two modes:

| | Rough Mode (`rough_skimmer_agent`) | Deep Mode (`connection_synthesizer_agent`) |
|---|---|---|
| **Approach** | Translate technical terminology into more understandable concepts inline, without losing essential meaning | Explain core technical terms with concise yet clear definitions, central to understanding the paper and critical to the user's research |
| **Format** | Inline simplification within summary text (plain-language equivalents, brief analogies) | Dedicated "Key Terminology" section (term + definition table) |

### 6.3 Paper-Type-Adaptive Feature Extraction

Domain features are activated based on detected paper type, not hardcoded:

- **Paper type detection**: Keyword signals from abstract/title (e.g., "observed" → observational; "model" → theoretical; "simulation" → computational)
- **Feature extraction**: The `methodology_analyst_agent` is parameterized by detected type (see Agent 5 in §3.2)

---

## 7. API Integration & Fallback Chain

### 7.1 ADS API (Primary)

The [ADS API](https://api.adsabs.harvard.edu/v1/search/query) serves as the **primary** metadata source for paper intake:

- **Endpoint**: `https://api.adsabs.harvard.edu/v1/search/query`
- **Key fields**: `title`, `abstract`, `author`, `keyword`, `year`, `citation_count`, `arxiv_class`, `bibcode`
- **Rate limit**: 5,000 queries/day
- **Special operators**: `citations()`, `references()`, `similar()`, `trending()`

**Reference Document**: `references/ads_api_protocol.md` (adapted from ARS deep-research)

### 7.2 arXiv API (First Fallback)

When ADS API is unavailable, fall back to arXiv API:

- **Endpoint**: `https://export.arxiv.org/api/query`
- **Format**: Atom 1.0 XML
- **Rate limit**: ~3s between requests
- **Key patterns**: ID lookup (`?id_list={arxiv_id}`), title search (`?search_query=ti:"{title}"`)

**Reference Document**: `references/arxiv_api_protocol.md` (ported and adapted from ARS deep-research)

### 7.3 Fallback Chain

```
ADS API (primary)
  ↓ unavailable
arXiv API (first fallback)
  ↓ unavailable
Manual metadata input (extract from PDF/user prompt)
```

The paper's **full text** always comes from the user-provided file — APIs supply metadata and abstract only.

---

## 8. Skill File Structure

```
astronomy-paper-summarize/
├── package.json                      # Plugin metadata
├── SKILL.md                          # Entry point — trigger detection, mode routing, execution mode selection, agent dispatch
│
├── agents/                           # 8 agent definition files
│   ├── paper_intake_agent.md
│   ├── rough_skimmer_agent.md
│   ├── relevance_assessor_agent.md
│   ├── deep_reader_agent.md
│   ├── methodology_analyst_agent.md
│   ├── critical_evaluator_agent.md
│   ├── connection_synthesizer_agent.md
│   └── report_compiler_agent.md
│
└── references/                       # Reference documents for agents
    ├── rough_overview_template.md    # Output template for rough overview
    ├── deep_summary_template.md      # Output template for deep summary
    ├── astronomical_pipeline.md      # 7-phase pipeline reference (context knowledge)
    ├── ads_api_protocol.md           # ADS API usage guide
    ├── arxiv_api_protocol.md         # arXiv API usage guide
    └── config_guide.md              # .astro-paper/config.yaml documentation

# .astro-paper/ is created at runtime in the parent directory of paper project directory, NOT in the skill directory.
```

**Not included** (per design decisions):
- ❌ `extension.mjs` — No Copilot CLI slash commands; mode routing is handled internally by `paper_intake_agent`
- ❌ `.github/` — No extension registration needed

---

## 9. Trigger Conditions

### 9.1 Trigger Keywords

The SKILL.md should detect these patterns (case-insensitive):

**Paper summarization**:
- "summarize paper", "summarize this paper", "paper summary", "summarize pdf"
- "rough overview", "deep summary", "paper deep dive"
- "analyze paper", "review this paper for me"
- "what does this paper say", "break down this paper"

**Astronomy-specific**:
- "astro paper", "astronomy paper", "astrophysics paper"
- "exoplanet paper", "cosmology paper", "galaxy paper"


### 9.2 Anti-Triggers

| Scenario | Use Instead |
|----------|-------------|
| Conducting original research | `deep-research` (ARS) |
| Writing a paper | `academic-paper` (ARS) |
| Peer reviewing a paper | `academic-paper-reviewer` (ARS) |
| Full research-to-publication pipeline | `academic-pipeline` (ARS) |

---

## 10. Agent Dispatch Pattern

The skill supports two execution modes, chosen by the user during Phase 1 intake. Agent templates in `agents/` are written to work with both modes — each template is a self-contained prompt that can be dispatched as a sub-agent or followed as a sequential checklist.

### 10.1 Subagent-Driven Mode (Default)

Agents are dispatched using sub-agent calls. This is the recommended mode for most papers, especially long or complex ones. The orchestrating agent (SKILL.md context or `paper_intake_agent`) loads agent templates from the `agents/` directory and dispatches them with paper content and research background injected. After each agent completes, the orchestrator collects output and passes it to the next agent in the phase sequence. For example, in Copilot CLI, sub-agents can be dispatched with:

```
task({
  agent_type: "general-purpose",
  name: "rough-skimmer",
  prompt: [content of agents/rough_skimmer_agent.md + paper text + config context]
})
```

**Advantages**: Clean context isolation per agent; better handling of long papers; each agent operates with full context window for its task.

When Subagent-Driven mode is selected on a platform that doesn't support sub-agent dispatch, the orchestrator should fall back to Inline Execution mode and inform the user.

### 10.2 Inline Execution Mode

The orchestrator reads each agent template and works through its instructions **sequentially in the current session**. No sub-agents are spawned:

1. Read agent template from `agents/<agent_name>.md`
2. Execute the agent's instructions as a structured task in-session
3. Collect and persist the output
4. Move to the next agent in the phase sequence

Agent templates serve as structured checklists — the orchestrator follows them step by step within the same conversation context.

**Advantages**: Lower latency (no sub-agent startup overhead); fewer context switches; better for short papers or quick triage.

**Risk**: On very long papers with `both` mode, the shared context window may overflow. The orchestrator should warn the user if the combined paper + agent content approaches context limits.

---

## 11. Error Handling

| Scenario | Response |
|----------|----------|
| Paper cannot be parsed (corrupted PDF, scanned image, no text layer) | Ask user to provide text extraction, arXiv ID, or paste abstract manually |
| No API available (no ADS token, arXiv down, local PDF only) | Extract metadata from paper title/first page; prompt user to verify or fill in manually |
| ADS API unavailable | Fall back to arXiv API or manual metadata entry |
| Config missing at runtime | Trigger Generic Fallback Mode (auto-extract from abstract) |
| `pdf_converter` command fails | Report error with command output; save Markdown files regardless; try out a corrected command, update the config entry for that mode (rough/deep) and prompt the user |
| Paper type detection ambiguous (mixed signals) | Default to balanced extraction covering all detected types; note ambiguity in output |
| Agent output missing a section | Flag with `[SECTION NOT PRODUCED]`; report compiler proceeds with available content |

---

## 12. Scope Boundaries

**In scope**:
- Single-paper summarization (rough overview, deep summary, or both)
- Research background configuration and management
- Paper-type-adaptive analysis (observational/theoretical/computational)
- Relevance scoring against user's research project
- Markdown + PDF output with standard naming conventions
- ADS API integration for metadata enrichment
- Cross-agent portability (Copilot CLI, Claude Code, Codex)

**Out of scope (for v1.0.0)**:
- Multi-paper batch processing
- Literature review across multiple papers
- English-only output; bilingual (Chinese/English) deferred to future version
- Direct paper download (user provides paper path or arXiv ID)
- Citation graph analysis or bibliometric networks
- Interactive reading mode (Q&A with paper)

---

## 13. Dependencies

- **Required**: [pandoc](https://pandoc.org/) — default PDF converter for Markdown → PDF conversion. The user must have `pandoc` installed with a PDF engine (e.g., xelatex, weasyprint, or wkhtmltopdf). Alternative converters can be configured via `.astro-paper/config.yaml`.
- **External APIs**: ADS API (primary), arXiv API (first fallback)
- **No Python/Node.js dependencies**: The skill is prompt-only; agent dispatch uses the platform's native sub-agent mechanism (Subagent-Driven mode) or sequential in-session processing (Inline Execution mode). API calls are made via platform-native tools (e.g., `web_fetch`, `curl`).

---

*Design spec version 1.1.0 — 2026-06-23*
