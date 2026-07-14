# Native Marketplace Catalogs Design

## Goal

Add native Codex and Claude Code marketplace catalogs for `astronomy-paper-summarize`, with both catalogs fetching the plugin from its GitHub repository.

## Scope

Create exactly two files:

- `.agents/plugins/marketplace.json` for Codex.
- `.claude-plugin/marketplace.json` for Claude Code.

Keep `.plugin/marketplace.json`, both platform plugin manifests, the canonical skill, and all package metadata unchanged.

## Architecture

Each platform receives its native marketplace schema. Both catalogs describe the same `astronomy-paper-summarize` plugin at version `1.1.0`, but use the source descriptor required by that platform.

The catalogs reference GitHub rather than the local marketplace root. This avoids Codex's current limitation with a local plugin source of `"./"` while keeping the repository layout unchanged.

## Codex Catalog

Location: `.agents/plugins/marketplace.json`

The catalog will contain:

- Marketplace name and display name: `astronomy-paper-summarize` / `Astronomy Paper Summarize`.
- One plugin entry named `astronomy-paper-summarize`.
- Git source type `url` with `https://github.com/zzyu17/astronomy-paper-summarize.git`.
- Installation policy `AVAILABLE`.
- Authentication policy `ON_INSTALL`.
- Category `Productivity`.

The Codex catalog will not duplicate plugin version, author, description, or interface metadata already owned by `.codex-plugin/plugin.json`.

## Claude Code Catalog

Location: `.claude-plugin/marketplace.json`

The catalog will contain:

- The Claude Code marketplace JSON Schema URL.
- Marketplace name `astronomy-paper-summarize`.
- Owner name `Zhenyu Zhang`.
- A concise marketplace description.
- One plugin entry named `astronomy-paper-summarize`.
- GitHub source type `github` with repository shorthand `zzyu17/astronomy-paper-summarize`.
- Category `productivity`.

The plugin entry will not duplicate version, author, description, homepage, repository, license, or keywords already owned by `.claude-plugin/plugin.json`. With Claude Code's default strict mode, the plugin manifest remains authoritative.

## Data Flow

1. A user registers the appropriate marketplace repository.
2. Codex or Claude Code reads its native catalog.
3. The catalog entry directs the client to the GitHub repository.
4. The client fetches the repository and reads its native `plugin.json` manifest.
5. The manifest exposes the shared skill under `skills/astronomy-paper-summarize/`.

## Failure Handling

- Marketplace registration or installation fails clearly if GitHub is unavailable or the repository is inaccessible.
- The source descriptors do not pin a commit, branch, or tag; the repository's default branch is used.
- Plugin release updates remain controlled by the `1.1.0` version in each platform's `plugin.json`. Future releases must bump those manifest versions.

## Verification Boundary

Implementation will include JSON and diff inspection only. It will not register either marketplace, install either plugin, run platform smoke tests, install Claude Code, or commit changes. The user will perform runtime testing manually.
