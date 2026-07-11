# agentic-support-coding

A [Claude Code](https://claude.com/claude-code) **plugin** that bundles the **Open Elements coding best practices** as reusable skills. It gives any Open Elements project the same baseline of Java and frontend coding conventions.

This plugin is a focused extract of the coding skills from [`claude-base`](https://github.com/OpenElementsLabs/claude-base); install it when you want the Open Elements coding conventions without the full base configuration.

## What's included

### Java

| Skill | What it does |
| --- | --- |
| `modern-java` | Write idiomatic modern Java (11–25) using newer language features and APIs (records, sealed types, pattern matching, virtual threads, structured concurrency, modern crypto, …) instead of legacy patterns. |
| `java-api-design` | Design and review public Java APIs: implementation hiding, type selection (interfaces/records/enums/annotations), design patterns (Factory, SPI, Facade, Builder, Strategy), generics, modularization, and long-term stability. |

### Frontend

| Skill | What it does |
| --- | --- |
| `frontend-design` | Create distinctive, production-grade frontend interfaces with high design quality — typography, color, motion, and composition that avoid generic AI aesthetics. |
| `web-frontend-builder` | Build web frontend applications with the Open Elements stack (Next.js, React, TypeScript, Tailwind CSS, shadcn/ui, pnpm). |
| `web-accessibility` | Audit and improve web accessibility following WCAG 2.1 guidelines (ARIA, keyboard navigation, color contrast, alt text, focus management). |

`.mcp.json` ships two MCP servers used by these skills: **shadcn** (browse/install shadcn/ui components) and **maven-central** (look up Maven dependency versions).

## Installation

Install from inside Claude Code:

```
/plugin marketplace add OpenElementsLabs/agentic-support-coding
/plugin install agentic-support-coding@open-elements
```

Then restart Claude Code (or run `/reload-plugins`).

Skills are **namespaced** under the plugin, e.g. `/agentic-support-coding:modern-java`, `/agentic-support-coding:web-frontend-builder`.

## A note on cross-references

The `web-frontend-builder` skill points to the `typescript-best-practices` and `open-elements-brand-guidelines` skills, which live in [`agentic-support-open-elements`](https://github.com/OpenElementsLabs/agentic-support-open-elements) but are **not** shipped here. Those references are intentional prose pointers. Install that plugin as well for the complete, self-referential set.

## Keeping up to date

Because the plugin is versioned, updates arrive when the `version` in `plugin.json` is bumped. Pull the latest release with:

```
/plugin marketplace update open-elements
```

## Repository layout

```
agentic-support-coding/
├── .claude-plugin/
│   ├── plugin.json          # plugin manifest
│   └── marketplace.json     # marketplace catalog (this repo is its own marketplace)
├── skills/                  # all skills, flat (Claude Code discovers them here)
├── conventions/             # shared convention docs referenced by skills
├── .mcp.json                # shadcn + maven-central MCP servers
└── CHANGELOG.md
```

## For maintainers

- **Releasing:** bump `version` in **both** `.claude-plugin/plugin.json` and the `agentic-support-coding` entry in `.claude-plugin/marketplace.json`, update `CHANGELOG.md`, and tag the release. Validate with `claude plugin validate ./ --strict`.

## License

This project is licensed under the Apache License 2.0 — see [LICENSE](LICENSE) for details. The `web-accessibility` skill is adapted from [web-quality-skills](https://github.com/addyosmani/web-quality-skills) by Addy Osmani (MIT) and the `frontend-design` skill from [Anthropic's skills](https://github.com/anthropics/skills) (Apache-2.0); each retains its upstream license in its own frontmatter.
