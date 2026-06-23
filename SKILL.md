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

## Mode Selection & Routing

When activated, follow this flow:

1. **Trigger detected** → Load this skill context
2. **Read `agents/paper_intake_agent.md`** → Execute intake as the first agent
3. **paper_intake_agent handles:**
   - `cd` into the paper's project directory (all paths relative from here)
   - Load/create `../.astro-paper/config.yaml` (parent of paper directory)
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

## Execution Mode Dispatch

### Output Discipline (Both Modes)

**CRITICAL: All analysis agents write output to staging files, never to the conversation.**

Analysis agents (`rough_skimmer`, `relevance_assessor`, `deep_reader`, `methodology_analyst`, `critical_evaluator`, `connection_synthesizer`) write their full output to `./paper-summaries/.staging/<agent_name>.md` and return ONLY a brief 1-line confirmation. The `report_compiler_agent` assembles final output from staging files via bash `cat` + heredocs — staging content never enters the conversation context.

### Subagent-Driven (Default)

For each agent in the active phase sequence, dispatch as a sub-agent:

```
task({
  agent_type: "general-purpose",
  name: "<agent-name>",
  prompt: [full content of agents/<agent_name>.md +
           paper text (for analysis agents) +
           research background from config +
           previous staging file paths (agents reference them if needed)]
})
```

The sub-agent writes its output to `paper-summaries/.staging/<agent_name>.md` and returns only a brief confirmation. The main session never loads the full content.

After all analysis agents complete, dispatch `report_compiler_agent` which assembles via bash.

### Inline Execution

For each agent in the active phase sequence, work through sequentially:

1. Read the agent template from `agents/<agent_name>.md`
2. Execute the agent's instructions — it writes output to `paper-summaries/.staging/<agent_name>.md`
3. Confirm the staging file was written (brief check)
4. Move to the next agent
5. After all analysis agents, execute `report_compiler_agent` instructions for bash assembly

**Warning:** If running `both` mode on a long paper, monitor context usage. Warn the user if approaching context limits.

### Fallback

If Subagent-Driven mode is selected but the platform does not support sub-agent dispatch, fall back to Inline Execution and inform the user.
