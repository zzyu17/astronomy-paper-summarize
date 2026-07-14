# Native Marketplace Catalogs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add native Codex and Claude Code marketplace catalogs that install `astronomy-paper-summarize` from its GitHub repository.

**Architecture:** Create one catalog at each platform's conventional repository path. The Codex catalog uses its URL-backed Git source descriptor and required policy fields; the Claude Code catalog uses its GitHub repository source descriptor and leaves the existing strict-mode plugin manifest authoritative for plugin metadata.

**Tech Stack:** JSON, Codex plugin marketplace schema, Claude Code plugin marketplace schema

## Global Constraints

- Create exactly `.agents/plugins/marketplace.json` and `.claude-plugin/marketplace.json`.
- Keep `.plugin/marketplace.json`, `.codex-plugin/plugin.json`, `.claude-plugin/plugin.json`, the canonical skill, and package metadata unchanged.
- Fetch the plugin from `https://github.com/zzyu17/astronomy-paper-summarize.git` for Codex and `zzyu17/astronomy-paper-summarize` for Claude Code.
- Do not pin a branch, tag, or commit; use the repository default branch.
- Do not duplicate plugin version, author, description, homepage, repository, license, keywords, or interface metadata in individual catalog entries.
- Do not register or install either marketplace, run platform smoke tests, install packages or tools, or commit changes.
- Verification is limited to JSON parsing and diff inspection.
- Implement inline on the existing `master` branch, as explicitly authorized by the user.

## Upstream References

- Codex marketplace schema and required policy fields: https://github.com/openai/codex/blob/main/codex-rs/skills/src/assets/samples/plugin-creator/references/plugin-json-spec.md
- Codex URL-backed plugin source resolver: https://github.com/openai/codex/blob/main/codex-rs/core-plugins/src/marketplace.rs
- Claude Code marketplace example: https://github.com/anthropics/claude-code/blob/main/.claude-plugin/marketplace.json
- Claude Code official plugin marketplace documentation: https://code.claude.com/docs/en/plugin-marketplaces

---

### Task 1: Add the Codex marketplace catalog

**Files:**
- Create: `.agents/plugins/marketplace.json`

**Interfaces:**
- Consumes: `.codex-plugin/plugin.json` at the root of the GitHub repository after Codex fetches the URL source.
- Produces: A Codex marketplace named `astronomy-paper-summarize` with one installable plugin entry.

- [x] **Step 1: Create the Codex catalog**

Create `.agents/plugins/marketplace.json` with exactly:

```json
{
  "name": "astronomy-paper-summarize",
  "interface": {
    "displayName": "Astronomy Paper Summarize"
  },
  "plugins": [
    {
      "name": "astronomy-paper-summarize",
      "source": {
        "source": "url",
        "url": "https://github.com/zzyu17/astronomy-paper-summarize.git"
      },
      "policy": {
        "installation": "AVAILABLE",
        "authentication": "ON_INSTALL"
      },
      "category": "Productivity"
    }
  ]
}
```

- [x] **Step 2: Parse and inspect the Codex catalog**

Run:

```bash
python3 -m json.tool .agents/plugins/marketplace.json
```

Expected: exit status `0` and formatted JSON containing the URL source, both policy fields, and category `Productivity`.

### Task 2: Add the Claude Code marketplace catalog

**Files:**
- Create: `.claude-plugin/marketplace.json`

**Interfaces:**
- Consumes: `.claude-plugin/plugin.json` at the root of the GitHub repository under Claude Code's default strict mode.
- Produces: A Claude Code marketplace named `astronomy-paper-summarize` with one GitHub-backed plugin entry.

- [x] **Step 1: Create the Claude Code catalog**

Create `.claude-plugin/marketplace.json` with exactly:

```json
{
  "$schema": "https://json.schemastore.org/claude-code-marketplace.json",
  "name": "astronomy-paper-summarize",
  "owner": {
    "name": "Zhenyu Zhang"
  },
  "description": "Multi-agent astronomy paper summarization for Claude Code.",
  "plugins": [
    {
      "name": "astronomy-paper-summarize",
      "source": {
        "source": "github",
        "repo": "zzyu17/astronomy-paper-summarize"
      },
      "category": "productivity"
    }
  ]
}
```

- [x] **Step 2: Parse and inspect the Claude Code catalog**

Run:

```bash
python3 -m json.tool .claude-plugin/marketplace.json
```

Expected: exit status `0` and formatted JSON containing the Claude Code marketplace schema URL, GitHub source, owner, description, and category `productivity`.

### Task 3: Inspect the scoped changes

**Files:**
- Inspect: `.agents/plugins/marketplace.json`
- Inspect: `.claude-plugin/marketplace.json`
- Preserve: `.plugin/marketplace.json`
- Preserve: `.codex-plugin/plugin.json`
- Preserve: `.claude-plugin/plugin.json`
- Preserve: `package.json`

**Interfaces:**
- Consumes: The two marketplace catalogs from Tasks 1 and 2.
- Produces: Evidence that the two files parse and that no out-of-scope file was changed during this implementation.

- [x] **Step 1: Check patch whitespace**

Run:

```bash
git diff --check -- .agents/plugins/marketplace.json .claude-plugin/marketplace.json
```

Expected: exit status `0` with no output.

- [x] **Step 2: Inspect the new catalogs**

Run:

```bash
git diff --no-index -- /dev/null .agents/plugins/marketplace.json
git diff --no-index -- /dev/null .claude-plugin/marketplace.json
```

Expected: each command displays one newly added catalog matching the exact JSON specified above. Exit status `1` is expected from `git diff --no-index` because differences exist.

- [x] **Step 3: Confirm no extra implementation files were added**

Run:

```bash
git status --short -- .agents/plugins/marketplace.json .claude-plugin/marketplace.json
```

Expected:

```text
?? .agents/plugins/marketplace.json
?? .claude-plugin/marketplace.json
```

Do not stage or commit the files.
