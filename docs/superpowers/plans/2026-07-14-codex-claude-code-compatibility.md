# Codex and Claude Code Compatibility Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:executing-plans` to implement this plan task-by-task. Do not use subagents unless the user explicitly authorizes delegation.

**Goal:** Make `astronomy-paper-summarize` installable and usable as a Codex plugin and a Claude Code plugin while preserving its existing Copilot CLI behavior and astronomy summarization workflow.

**Architecture:** Store one canonical Agent Skill under `skills/astronomy-paper-summarize/`, including all agent prompt templates and references. Add thin, platform-specific manifests at the plugin root and replace the Copilot-specific dispatch example with a platform-neutral orchestration contract that maps to Codex `spawn_agent`/`wait_agent` and Claude Code `Agent`.

**Tech Stack:** Markdown Agent Skills, JSON plugin manifests, YAML frontmatter, native Codex and Claude Code subagent tools.

## Global Constraints

- Make minimal semantic changes: preserve all astronomy prompts, scoring logic, report templates, staging-file rules, and output naming.
- Preserve existing Copilot CLI compatibility through `package.json`.
- Maintain one canonical copy of every skill, agent prompt, and reference document; do not duplicate platform-specific workflow trees.
- Do not add packages, tools, scripts, hooks, MCP servers, apps, icons, or marketplace catalogs.
- Do not run tests or platform smoke tests; the user will run all testing manually afterward.
- Do not install Codex or Claude Code tooling.
- Do not commit changes unless the user gives separate explicit approval.
- Run all implementation commands in WSL2 from `/home/zzyu/skills/astronomy-paper-summarize`.
- Use `python3`, never `python`, if Python is needed during implementation.
- Remove temporary files or directories created during implementation before handing off.

---

## Target File Structure

```text
astronomy-paper-summarize/
├── .claude-plugin/
│   └── plugin.json
├── .codex-plugin/
│   └── plugin.json
├── docs/
│   └── superpowers/
│       ├── plans/
│       └── specs/
├── skills/
│   └── astronomy-paper-summarize/
│       ├── SKILL.md
│       ├── agents/
│       │   ├── connection_synthesizer_agent.md
│       │   ├── critical_evaluator_agent.md
│       │   ├── deep_reader_agent.md
│       │   ├── methodology_analyst_agent.md
│       │   ├── paper_intake_agent.md
│       │   ├── relevance_assessor_agent.md
│       │   ├── report_compiler_agent.md
│       │   └── rough_skimmer_agent.md
│       └── references/
│           ├── ads_api_protocol.md
│           ├── arxiv_api_protocol.md
│           ├── astronomical_pipeline.md
│           ├── config_guide.md
│           ├── deep_summary_template.md
│           └── rough_overview_template.md
├── LICENSE
└── package.json
```

The existing historical design and plan documents remain at their current paths. Do not rewrite them to reflect the new layout; this plan records the compatibility migration.

---

### Task 1: Move the Canonical Skill Into the Shared Standard Layout

**Files:**

- Move: `SKILL.md` → `skills/astronomy-paper-summarize/SKILL.md`
- Move: `agents/` → `skills/astronomy-paper-summarize/agents/`
- Move: `references/` → `skills/astronomy-paper-summarize/references/`

**Interfaces:**

- Consumes: The current root skill, eight prompt templates, and six reference documents.
- Produces: One self-contained Agent Skill directory discoverable by both Codex and Claude Code.

- [ ] **Step 1: Create the canonical skill directory**

Create `skills/astronomy-paper-summarize/` using normal filesystem operations. Do not create a second copy of any existing content.

- [ ] **Step 2: Move the entry point**

Move the root `SKILL.md` to `skills/astronomy-paper-summarize/SKILL.md` without changing its content in this step.

- [ ] **Step 3: Move the prompt templates**

Move the entire root `agents/` directory to `skills/astronomy-paper-summarize/agents/`. Preserve every filename and file body.

- [ ] **Step 4: Move the references**

Move the entire root `references/` directory to `skills/astronomy-paper-summarize/references/`. Preserve every filename and file body.

- [ ] **Step 5: Check the working tree for accidental copies**

Inspect `git status --short`. The old paths should appear as deletions and the new paths as additions or renames. There must be only one `SKILL.md` for this plugin and one copy of each agent and reference file.

Do not run tests.

---

### Task 2: Preserve Copilot CLI Discovery and Synchronize Release Metadata

**Files:**

- Modify: `package.json`
- Modify: `skills/astronomy-paper-summarize/SKILL.md`

**Interfaces:**

- Consumes: Existing Copilot CLI package metadata and skill frontmatter.
- Produces: Copilot discovery rooted at `./skills` and synchronized version `1.1.0`.

- [ ] **Step 1: Update the package version**

Change the top-level `package.json` version from `1.0.0` to `1.1.0`.

- [ ] **Step 2: Point Copilot at the shared skills directory**

Change only the skills directory field to:

```json
"skills": {
  "directory": "./skills"
}
```

Keep all other package metadata unchanged.

- [ ] **Step 3: Synchronize skill frontmatter**

In `skills/astronomy-paper-summarize/SKILL.md`, change:

```yaml
metadata:
  version: "1.1.0"
  last_updated: "2026-07-14"
  status: active
  task_type: open-ended
```

Keep the skill name and trigger description unchanged.

Do not run tests.

---

### Task 3: Add the Codex Plugin Manifest

**Files:**

- Create: `.codex-plugin/plugin.json`

**Interfaces:**

- Consumes: The shared `skills/` directory and existing package metadata.
- Produces: A Codex plugin entry point that exposes the canonical astronomy skill.

- [ ] **Step 1: Create `.codex-plugin/plugin.json`**

Use this exact manifest:

```json
{
  "name": "astronomy-paper-summarize",
  "version": "1.1.0",
  "description": "Multi-agent astronomy paper summarization with rough overview and deep critical-analysis modes.",
  "author": {
    "name": "Zhenyu Zhang",
    "url": "https://github.com/zzyu17"
  },
  "homepage": "https://github.com/zzyu17/astronomy-paper-summarize",
  "repository": "https://github.com/zzyu17/astronomy-paper-summarize",
  "license": "CC-BY-NC-4.0",
  "keywords": [
    "astronomy",
    "astrophysics",
    "paper-summary",
    "literature-review",
    "paper-summarization",
    "astronomy-research",
    "ads",
    "arxiv"
  ],
  "skills": "./skills/",
  "interface": {
    "displayName": "Astronomy Paper Summarize",
    "shortDescription": "Multi-agent astronomy paper summarization",
    "longDescription": "Create rough overviews and deep critical summaries of astronomy papers with research-aware relevance scoring.",
    "developerName": "Zhenyu Zhang",
    "category": "Productivity",
    "capabilities": [
      "Read",
      "Write"
    ],
    "websiteURL": "https://github.com/zzyu17/astronomy-paper-summarize",
    "defaultPrompt": [
      "Summarize this astronomy paper."
    ]
  }
}
```

- [ ] **Step 2: Keep unsupported components absent**

Do not add `apps`, `mcpServers`, `hooks`, assets, or marketplace information. This remains a prompt-only, skill-backed plugin.

Do not run Codex validation or smoke tests; the user will validate it manually.

---

### Task 4: Add the Claude Code Plugin Manifest

**Files:**

- Create: `.claude-plugin/plugin.json`

**Interfaces:**

- Consumes: Claude Code's default `skills/` discovery and the shared package metadata.
- Produces: A Claude Code plugin entry point with the stable namespace `astronomy-paper-summarize`.

- [ ] **Step 1: Create `.claude-plugin/plugin.json`**

Use this exact manifest:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-plugin-manifest.json",
  "name": "astronomy-paper-summarize",
  "displayName": "Astronomy Paper Summarize",
  "version": "1.1.0",
  "description": "Multi-agent astronomy paper summarization with rough overview and deep critical-analysis modes.",
  "author": {
    "name": "Zhenyu Zhang",
    "url": "https://github.com/zzyu17"
  },
  "homepage": "https://github.com/zzyu17/astronomy-paper-summarize",
  "repository": "https://github.com/zzyu17/astronomy-paper-summarize",
  "license": "CC-BY-NC-4.0",
  "keywords": [
    "astronomy",
    "astrophysics",
    "paper-summary",
    "literature-review",
    "paper-summarization",
    "astronomy-research",
    "ads",
    "arxiv"
  ]
}
```

- [ ] **Step 2: Rely on default skill discovery**

Do not add a custom `skills` path. Claude Code should discover `skills/astronomy-paper-summarize/SKILL.md` from the conventional root `skills/` directory.

- [ ] **Step 3: Keep prompt templates out of Claude's plugin-agent registry**

Do not create a root `agents/` directory and do not point the Claude manifest at `skills/astronomy-paper-summarize/agents/`. These files remain workflow prompt templates loaded by the skill rather than independently invocable Claude plugin subagents.

Do not run Claude validation or smoke tests; the user will validate it manually.

---

### Task 5: Replace Platform-Specific Dispatch With a Shared Orchestration Contract

**Files:**

- Modify: `skills/astronomy-paper-summarize/SKILL.md`
- Modify only if needed for terminology consistency: `skills/astronomy-paper-summarize/agents/paper_intake_agent.md`

**Interfaces:**

- Consumes: Selected mode, selected execution mode, `PAPER_DIR`, the canonical skill directory, agent template paths, research background, and prior staging-file paths.
- Produces: The same sequential staging-file workflow through each platform's native subagent facility.

- [ ] **Step 1: Define the two independent path roots**

Add instructions near the beginning of execution dispatch that distinguish:

```text
SKILL_DIR = absolute directory containing this SKILL.md
PAPER_DIR = absolute directory containing the user's paper
```

State that bundled prompt templates and references always resolve from `SKILL_DIR`, while paper input, configuration, staging files, and generated reports always resolve from `PAPER_DIR`.

- [ ] **Step 2: Replace the `task({...})` example**

Remove the current Copilot-specific JavaScript-like `task({...})` block. Replace it with this platform mapping:

| Platform | Native subagent operation | Completion operation |
|---|---|---|
| Codex | `spawn_agent` with a concrete `task_name`, `fork_turns: "none"`, and the assembled prompt in `message` | `wait_agent` for the returned agent |
| Claude Code | `Agent` with the built-in `general-purpose` agent and the assembled prompt | Wait for the foreground Agent result |
| Copilot CLI or another compatible host | Its native general-purpose subagent operation | Wait for that worker to finish |

State that Claude Code's legacy `Task` alias must not be used in new instructions.

- [ ] **Step 3: Define the exact worker prompt contract**

For every dispatched worker, require the orchestrator to assemble a prompt containing, in this order:

```text
PAPER_DIR="<absolute paper directory>"
SKILL_DIR="<absolute directory containing SKILL.md>"

CRITICAL: Shell sessions are stateless. Every command that accesses paper data or output must start with:
cd "${PAPER_DIR}" &&

Bundled resources are read from absolute paths under:
${SKILL_DIR}/agents/
${SKILL_DIR}/references/

<full content of the selected agent template>

Paper text path relative to PAPER_DIR:
./paper-summaries/.staging/paper_fulltext.txt

<research background>
<paths of prerequisite staging files>
```

The template content must be read from `${SKILL_DIR}/agents/<agent_name>.md`. Reference documents required by a worker must be identified with absolute paths under `${SKILL_DIR}/references/`.

- [ ] **Step 4: Preserve sequential dependencies**

Keep the existing phase order unchanged. Spawn only the next agent whose prerequisite staging files have been produced. Do not parallelize dependent analysis agents.

- [ ] **Step 5: Preserve output discipline**

Retain the existing rules that analysis agents write full content only to staging files, return one-line confirmations, and never return full summaries to the parent conversation.

- [ ] **Step 6: Update inline execution resource paths**

Change inline-mode template-loading instructions to read agent templates from `${SKILL_DIR}/agents/<agent_name>.md` and reference material from `${SKILL_DIR}/references/<file>.md`. Keep all paper-directory commands prefixed with `cd "${PAPER_DIR}" &&`.

- [ ] **Step 7: Keep the fallback**

Preserve the rule that a host without a native subagent facility falls back to Inline Execution and informs the user.

- [ ] **Step 8: Adjust intake wording only if necessary**

If `paper_intake_agent.md` still names a platform-specific tool after the main dispatch rewrite, replace that wording with “the platform's native subagent operation.” Do not alter intake questions, configuration behavior, metadata acquisition, or routing.

Do not run any end-to-end summarization or platform tests.

---

### Task 6: Review Scope and Hand Off for User Testing

**Files:**

- Review only: all files changed by Tasks 1–5

**Interfaces:**

- Consumes: The completed compatibility edits.
- Produces: A clean implementation handoff without testing, installation, or commit side effects.

- [ ] **Step 1: Review the diff without executing tests**

Inspect the final diff and confirm that it contains only:

- Structural moves into `skills/astronomy-paper-summarize/`.
- The `package.json` discovery-path and version changes.
- Synchronized `SKILL.md` metadata.
- The Codex and Claude Code manifests.
- Platform-neutral dispatch and bundled-resource path instructions.

Do not run validators, smoke tests, plugin installations, or paper summarization.

- [ ] **Step 2: Check temporary artifacts**

Inspect temporary files and directories created during implementation. Remove those that are no longer needed; do not remove persistent project files.

- [ ] **Step 3: Report the untested handoff accurately**

Tell the user:

- Which files were moved, created, and modified.
- That no tests or platform smoke tests were run at their request.
- That Claude Code was not installed.
- That the changes remain uncommitted.
- The manual commands they may choose to run afterward, without executing them.

- [ ] **Step 4: Ask before any commit**

If the user wants a commit after manual testing, request explicit approval first. Use a concise message such as:

```text
feat: add Codex and Claude Code plugin support
```

Do not add generated-by or co-author attribution.

---

## Deferred Manual Testing

The user owns all testing after implementation. Suggested commands for the user's later manual run are documented here only and must not be executed as part of implementation:

```bash
python3 -m json.tool package.json
python3 -m json.tool .codex-plugin/plugin.json
python3 -m json.tool .claude-plugin/plugin.json
python3 /home/zzyu/.codex/skills/.system/skill-creator/scripts/quick_validate.py skills/astronomy-paper-summarize
python3 /home/zzyu/.codex/skills/.system/plugin-creator/scripts/validate_plugin.py .
claude plugin validate .
claude --plugin-dir .
```

Codex marketplace installation and Claude Code installation are intentionally outside this implementation plan. They require separate user action or approval.
