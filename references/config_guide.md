---
name: config_guide
description: "Reference documentation for .astro-paper/config.yaml — research background configuration and PDF converter management"
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

The `pdf_converter_rough` and `pdf_converter_deep` fields store the shell command used to convert Markdown summaries to PDF. They are managed automatically by the `report_compiler_agent`, but can also be edited manually.

### Automatic Management

**Initial state**: Both fields start empty (`""`).

**First conversion**: When the `report_compiler_agent` runs its first PDF conversion for each mode, it uses `pandoc` as the default converter:
```
pandoc --pdf-engine=xelatex -V geometry:margin=1in <markdown-file> -o <pdf-file>
```
If the conversion succeeds, the command (with actual filenames) is stored in the config for that mode. If the default command fails (e.g., `xelatex` not installed), the agent will try alternatives and prompt the user for help.

**Re-conversion**: On subsequent runs, the agent uses the stored command. If it fails, the agent falls back to the pandoc default, updates the config entry, and notifies the user.

### Manual Editing

The stored commands use actual filenames (not placeholders), so they can be run directly from the paper directory. The `pdf_converter_rough` or `pdf_converter_deep` fields can also be edited to use a different converter:

```yaml
output:
  pdf_converter_rough: "pandoc --pdf-engine=weasyprint paper-summaries/My Paper-rough-overview.md -o My Paper-rough-overview.pdf"
  pdf_converter_deep: "pandoc --pdf-engine=xelatex -V geometry:margin=1in paper-summaries/My Paper-deep-summary.md -o My Paper-deep-summary.pdf"
```

**To re-convert PDFs** after editing the Markdown summaries, run the stored command directly from the paper directory — no placeholder substitution needed.

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
3. Save approved values to `../.astro-paper/config.yaml` for future use

## Updating Config

The user can update their research background at any time during a summarization session. The agent should prompt: "Would you like to update your research background?" when appropriate.

## Cross-Agent Portability

`.astro-paper/config.yaml` is plain YAML — readable by Copilot CLI, Claude Code, Codex, or any LLM agents. No tool-specific format or dependencies.
