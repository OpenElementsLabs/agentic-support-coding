# Changelog

All notable changes to this plugin are documented here. The format is based on
[Keep a Changelog](https://keepachangelog.com/en/1.1.0/), and this project adheres to
[Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- Frontend skill `browser-compatibility`: `browserslist` (incl. Firefox ESR, without Opera Mini) with
  `eslint-plugin-compat`, a lint ban on unguarded `localStorage`/`sessionStorage`, the privacy
  settings of hardened browsers such as LibreWolf and what they break, and Playwright cross-browser
  smoke tests (Chromium, Firefox, WebKit and a `firefox-hardened` profile) with a GitHub Actions
  workflow against the Docker stack and the mock OIDC server. The smoke-test rules record the
  causes of flaky runs found while introducing it in `resource-planning`.

### Changed

- `web-frontend-builder` and `open-elements-app-layout` point to the new skill from their testing
  sections and checklists; `open-elements-app-layout` lists the unguarded-storage issues
  open-elements-ui#19 and nextjs-app-layer#14.

## [0.2.0] - 2026-09-07

### Added

- Frontend skill `open-elements-app-layout`: the application shell and view flow shared by
  `open-crm`, `open-tasks` and `open-transcript` — route groups, the thin `page.tsx` plus
  `<route>-client.tsx` split, the component co-location rule, the two-surface colour model
  (dark sidebar / light content), sidebar composition, admin page factories, i18n merge order,
  the OIDC wiring and the backend proxy.
  It records several defects found in the reference applications rather than reproducing them:
  the content area must be offset with `md:ml-64` because the sidebar is `position: fixed`;
  `globals.css` needs the `body`/heading rules or the brand fonts load but never apply;
  plain `bottomChildren` need an explicit text colour; `...uiDe` belongs first in the
  translation spread, not last; the `/api/*` proxy is a route handler, not a rewrite, so the
  `BACKEND_URL` build argument is obsolete.

## [0.1.0] - 2026-07-11

### Added

- Initial release: Claude Code plugin scaffolded as a focused extract of the coding skills from `claude-base`.
- Java skills: `modern-java`, `java-api-design`.
- Frontend skills: `frontend-design`, `web-frontend-builder`, `web-accessibility`.
- `conventions/software-quality.md`, referenced by the `web-frontend-builder` skill.
- `.mcp.json` shipping the `shadcn` and `maven-central` MCP servers.
