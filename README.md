# Astronomy Paper Summarize

`astronomy-paper-summarize` turns astronomy and astrophysics papers into structured Markdown and PDF reports. It uses eight role-specific agents for intake, rapid triage, technical reading, critical evaluation, research connection, and report assembly.

The plugin works with GitHub Copilot CLI, Codex, Claude Code, and other agent platforms that can load `SKILL.md` skills.

## What it does

Choose one of three output modes:

| Mode | Best for | Report contents |
| --- | --- | --- |
| Rough | Screening papers quickly | Research question, methods, 1 to 3 findings, relevance score, project implications, and a short takeaway |
| Deep | Studying a paper closely | Structured summary, methodology, critical evaluation, research connections, glossary, useful references, and possible next steps |
| Both | Keeping a quick reference and a full analysis | Produces both reports in the same run |

The analysis can use your research topic, goal, and planned methodology as a lens. This makes relevance scores and research connections specific to your project instead of generic to the paper's field.

## Requirements

You need a supported agent platform and a local copy of the paper as PDF, plain text, Markdown, or pasted text.

For PDF input and output, install:

- `pdftotext` from Poppler to extract text from PDFs.
- `pandoc` to create PDF reports.
- A Pandoc-compatible PDF engine. The default command uses XeLaTeX with the FreeSerif font. The plugin can also try WeasyPrint or wkhtmltopdf.

If PDF conversion is unavailable, the plugin still keeps the Markdown report.

A [NASA ADS API token](https://ui.adsabs.harvard.edu/user/settings/token) is optional. It provides richer publication metadata and citation counts. Set it before starting your agent platform:

```bash
export ADS_API_TOKEN="your-token-here"
```

Without a token, the plugin falls back to the arXiv API and then to metadata extracted from the paper.

## Install

### GitHub Copilot CLI

Register this repository as a marketplace, then install the plugin:

```bash
copilot plugin marketplace add zzyu17/astronomy-paper-summarize
copilot plugin install astronomy-paper-summarize@astronomy-paper-summarize
```

Inside an interactive GitHub Copilot CLI session, you can use:
```text
/plugin marketplace add zzyu17/astronomy-paper-summarize
/plugin install astronomy-paper-summarize@astronomy-paper-summarize
```

Then run `/restart` or start a new session with `/clear`.

### Codex

Register the marketplace and install the plugin:

```bash
codex plugin marketplace add zzyu17/astronomy-paper-summarize
codex plugin add astronomy-paper-summarize@astronomy-paper-summarize
```

Start a new Codex session after installation so the plugin is discovered.

### Claude Code

Register the marketplace and install the plugin:

```bash
claude plugin marketplace add zzyu17/astronomy-paper-summarize
claude plugin install astronomy-paper-summarize@astronomy-paper-summarize
```

Inside an interactive Claude Code session, you can use:

```text
/plugin marketplace add zzyu17/astronomy-paper-summarize
/plugin install astronomy-paper-summarize@astronomy-paper-summarize
/reload-plugins
```

### Other agent platforms

The canonical skill is in [`skills/astronomy-paper-summarize`](skills/astronomy-paper-summarize). To use it with another platform:

1. Clone this repository:

   ```bash
   git clone https://github.com/zzyu17/astronomy-paper-summarize.git
   ```

2. Add the repository's `skills` directory to the platform's skill search path, or copy the entire `skills/astronomy-paper-summarize` directory into the platform's user or project skill directory. Keep its `agents` and `references` subdirectories with `SKILL.md`.
3. Start a new session and ask the platform to summarize an astronomy paper. If the platform has no subagent support, select inline execution when prompted.

Exact skill directories and reload behavior vary by platform.

## First use

Give the agent a local paper path and say what kind of summary you want:

```text
Summarize /home/me/papers/example-paper.pdf.
```

```text
Give me a rough overview of C:\Users\me\papers\example-paper.pdf.
```

```text
Create both a rough overview and a deep summary for this astronomy paper.
```

On the first run for a research project, the intake step asks for:

- Your core research topic.
- Your primary research goal.
- Your proposed methodology.
- Rough, deep, or both output.
- Subagent-driven or inline execution.

Subagent-driven execution is the default and is better suited to long papers. Inline execution runs the same workflow sequentially in the current session and works on platforms without subagent support.

## Output files

Markdown reports go into `paper-summaries/` inside the paper's directory. PDFs go beside the original paper:

```text
paper-directory/
├── original-paper.pdf
├── Paper Title-rough-overview.pdf
├── Paper Title-deep-summary.pdf
└── paper-summaries/
    ├── Paper Title-rough-overview.md
    └── Paper Title-deep-summary.md
```

Only files for the selected mode are created. The plugin removes its temporary staging directory after it assembles the reports.

## Research configuration

The plugin stores shared project settings in:

```text
<parent-of-paper-directory>/.astro-paper/config.yaml
```

Paper directories under the same parent share this configuration. A typical file looks like:

```yaml
research_background:
  core_topic: "Planets in the Neptunian desert"
  primary_goal: "Understand their formation and evolution"
  proposed_methodology: "Statistical analysis of TESS planet populations"

output:
  pdf_converter_rough: ""
  pdf_converter_deep: ""
```

After a successful PDF conversion, the plugin saves the latest working conversion command in the corresponding `pdf_converter` field. You can edit the research background or converter settings between runs.

## Troubleshooting

### The plugin is not discovered

Check that the marketplace was registered before the plugin was installed. Then restart the platform or reload its plugins. For a generic installation, confirm that the platform can see the complete `skills/astronomy-paper-summarize` directory.

### PDF text extraction fails

Confirm that `pdftotext` is installed and available on `PATH`. Scanned PDFs usually need OCR first; alternatively, provide a text or Markdown copy of the paper.

### PDF generation fails

The Markdown report should still be available in `paper-summaries/`. Check that `pandoc` and the configured PDF engine are installed. You can change the converter command in `.astro-paper/config.yaml` and run it manually from the paper directory.

### ADS metadata is unavailable

Check that `ADS_API_TOKEN` is visible in the environment where the agent platform was started. The workflow can continue without ADS by using arXiv or metadata from the paper itself.

### The platform cannot create subagents

Choose inline execution. It uses the same agent instructions in sequence within the current session.

## License

This project is licensed under [CC BY-NC 4.0](LICENSE).
