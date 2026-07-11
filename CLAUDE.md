# Agentic Support Coding — Project Rules

This repository **is** a Claude Code plugin: it bundles the Open Elements coding best-practices skills and distributes them through its own marketplace.

## Purpose of this project

The goal is to give any Open Elements project a ready-made set of coding best practices as reusable skills. This plugin is a focused extract of the coding skills from [`claude-base`](https://github.com/OpenElementsLabs/claude-base); install it when you want the Open Elements coding conventions (Java and frontend) without the full base configuration. When working on this repository, keep in mind that every skill you write here will be used across many different projects.

## Repository Structure

The repository root is the plugin root (and the marketplace root).

- `.claude-plugin/plugin.json` — plugin manifest (name, version, metadata)
- `.claude-plugin/marketplace.json` — marketplace catalog (lists this plugin, `source: "./"`)
- `skills/` — reusable Claude Code skills, flat (`skills/<name>/SKILL.md`); discovered automatically
- `conventions/` — shared convention documents referenced by skills via `${CLAUDE_PLUGIN_ROOT}/conventions/…`
- `.mcp.json` — MCP servers shipped with the plugin
- `README.md` — public-facing documentation for users of this plugin
- `CHANGELOG.md` — release history

## Skills

**Java:**

- `modern-java` — write idiomatic modern Java (11–25) using newer language features and APIs instead of legacy patterns.
- `java-api-design` — design and review public Java APIs: implementation hiding, type selection, design patterns, modularization, and long-term stability.

**Frontend:**

- `frontend-design` — create distinctive, production-grade frontend interfaces with high design quality, avoiding generic AI aesthetics.
- `web-frontend-builder` — build web frontend applications with the Open Elements stack (Next.js, React, TypeScript, Tailwind CSS, shadcn/ui, pnpm).
- `web-accessibility` — audit and improve web accessibility following WCAG 2.1 guidelines.

The `web-frontend-builder` skill references the `typescript-best-practices` and `open-elements-brand-guidelines` skills, which live in [`agentic-support-open-elements`](https://github.com/OpenElementsLabs/agentic-support-open-elements) and are **not** bundled here. Those references are intentional prose pointers; install that plugin for the full set.

## Distribution

The plugin is installed from inside Claude Code:

```
/plugin marketplace add OpenElementsLabs/agentic-support-coding
/plugin install agentic-support-coding@open-elements-coding
```

Skills are namespaced under the plugin, e.g. `/agentic-support-coding:modern-java`.

**Releasing:** bump `version` in **both** `.claude-plugin/plugin.json` and the `agentic-support-coding` entry in `.claude-plugin/marketplace.json`, update `CHANGELOG.md`, tag the release, and validate with `claude plugin validate ./ --strict`. Users only receive an update when the version is bumped.

A plugin-root `CLAUDE.md` is not loaded as context, so ship guidance as skills or `conventions/` — not in this file.

## Writing Guidelines

### For skills (`skills/<name>/SKILL.md`)

- Each skill must have valid frontmatter with a `name` (kebab-case, matching its directory) and a `description`, plus a clear H1 title.
- Keep skills focused on a single task. Do not combine unrelated actions into one skill.
- Reference bundled files (conventions, assets) with `${CLAUDE_PLUGIN_ROOT}/…`, never with `../` traversal or absolute paths — the plugin is copied to a cache on install.
- Preserve the `metadata` frontmatter block (`source`, `author`) so provenance tooling keeps working. For skills adapted from third parties, keep the upstream attribution (`source`, `author`, `license`) intact.

## Quality Standards

- All files must be valid Markdown with no formatting errors.
- Use consistent heading levels (H1 for title, H2 for sections, H3 for subsections).
- No trailing whitespace, no tabs for indentation in Markdown files.
- Run `claude plugin validate ./ --strict` after structural changes.

## What NOT to include

- No project-specific configurations (build scripts, CI pipelines, IDE settings).
- No absolute file paths or machine-specific references.
- No secrets, credentials, or environment-specific values.
