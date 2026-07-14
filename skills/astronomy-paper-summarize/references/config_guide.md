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

**Every PDF conversion**: The `report_compiler_agent` builds a fresh command for the current paper. After a successful conversion, it stores the command (with that paper's filenames) in the config, overwriting any previous entry.

- **First run or empty config**: Uses the pandoc default:
  ```
  pandoc --pdf-engine=xelatex -V 'mainfont=FreeSerif' -V geometry:margin=1in <markdown-file> -o <pdf-file>
  ```
- **Subsequent papers**: Reads the stored command to learn the user's preferred converter options (engine, font, geometry), but **always rebuilds with the current paper's filenames**. The stored filenames from a previous paper are never reused.
- **On failure**: Tries fallback engines (`weasyprint`, `wkhtmltopdf`). If all fail, saves Markdown only and reports errors.

**Important**: Because the config is shared across papers in a parent directory, the stored command always reflects the **most recently processed paper** only.

### Manual Editing & Re-conversion

The stored commands use actual filenames for the most recent paper. To re-convert PDFs after editing Markdown, run the stored command directly from the paper directory — no placeholder substitution needed. Note that if you process a new paper, the stored command is updated to that paper's filenames.

You can also manually edit the `pdf_converter_rough` or `pdf_converter_deep` fields to change the converter engine or options:

### Example Commands

Stored commands use actual filenames. Below are the equivalents before filename substitution:

| Converter | Command Pattern |
|-----------|----------------|
| **pandoc + xelatex** (default) | `pandoc --pdf-engine=xelatex -V 'mainfont=FreeSerif' -V geometry:margin=1in <md-file> -o <pdf-file>` |
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
