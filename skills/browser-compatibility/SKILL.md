---
name: browser-compatibility
license: Apache-2.0
metadata:
  source: https://github.com/OpenElementsLabs/agentic-support-coding
  author: Open Elements
description: Keep an Open Elements web frontend working in every supported browser, including hardened ones such as LibreWolf or Firefox with strict privacy settings. Covers the browserslist plus eslint-plugin-compat lint setup, the ban on unguarded localStorage/sessionStorage, the privacy settings that commonly break apps, and Playwright cross-browser smoke tests (Chromium, Firefox, WebKit and a hardened Firefox profile) run in CI against the Docker stack with the mock OIDC server. Use when setting up a new frontend, adding CI to one, touching browser storage or other browser APIs, or when a user reports that the app does not work in a particular browser.
---

# Browser Compatibility

An Open Elements app has to work in every browser the users actually run. That includes
privacy-focused browsers: LibreWolf and hardened Firefox profiles ship strict settings by default.
They break apps in ways that never show up in a developer's Chrome.

Two layers catch these problems. They are cheap to set up, so every frontend has both:

1. **Lint** catches known incompatibilities while you write code: browser APIs a target browser
   lacks, and unguarded storage access.
2. **Cross-browser smoke tests** in CI catch everything else. They log in and open every main page
   in Chromium, Firefox, WebKit and a hardened Firefox, and fail on any runtime error.

Paths below assume the standard two-application layout (`frontend/`, `backend/`, Docker Compose
with a mock OIDC server in `docker-compose.override.yml`). Adjust them for other layouts.

## What hardened browsers change

LibreWolf's defaults, and the settings privacy-minded Firefox users turn on, differ from a stock
browser in these ways. Write code that survives all of them:

| Setting | Effect | Consequence for the app |
|---|---|---|
| `privacy.resistFingerprinting` | time zone is UTC, locale is `en-US`, timers are coarse, screen size is spoofed | never derive the user's time zone or language from the browser alone; offer a setting. Date maths must not assume local time is the user's time |
| `network.cookie.cookieBehavior = 5` (dynamic first-party isolation) | cookies and storage are partitioned per top-level site | no auth or state that relies on third-party cookies or cross-site iframes |
| Enhanced Tracking Protection "strict" | known trackers and fingerprinting scripts are blocked | do not load app-critical code or fonts from third-party CDNs; self-host fonts (`next/font` does) |
| `dom.security.https_only_mode` | every `http://` URL is upgraded or blocked (local addresses exempt) | no `http://` links, images or API calls in production |
| `network.http.referer.XOriginTrimmingPolicy = 2` | cross-origin `Referer` is the origin only | never depend on a full cross-origin referrer |
| `webgl.disabled`, canvas extraction prompts | WebGL is missing, canvas readback is blocked | charts as SVG, not canvas/WebGL, unless there is a fallback |
| Site data blocked or `dom.storage.enabled = false` | `localStorage` throws `SecurityError` or is `null` | **every storage access must be guarded** (see below) |
| Clear cookies and site data on exit | the session ends when the browser closes | the login must be quick to repeat; never keep important state only in storage |

## Lint: `browserslist` and `eslint-plugin-compat`

Declare the supported browsers once in `frontend/package.json`. Next.js reads the same list for its
transpilation targets, and the lint plugin checks API usage against it:

```json
"browserslist": ["defaults", "not op_mini all", "Firefox ESR", "not dead"]
```

- `defaults` alone is not enough: Firefox ESR is the baseline for many enterprise and Linux users,
  so it is listed explicitly.
- `not op_mini all` removes Opera Mini. It supports neither `fetch` nor `URLSearchParams`, so
  without this line every API call is flagged.

Add the plugin with an exact version: `pnpm add -D -E eslint-plugin-compat`. Then enable it, together
with the storage ban, in `frontend/eslint.config.mjs`. The example below fits the flat-compat setup
that `create-next-app` generates:

```js
import compatPlugin from "eslint-plugin-compat";

const eslintConfig = [
  ...compat.extends("next/core-web-vitals", "next/typescript"),
  // Flags browser APIs that a target in the browserslist (incl. Firefox ESR) lacks.
  { ...compatPlugin.configs["flat/recommended"], files: ["src/**/*.{ts,tsx}"] },
  {
    // Storage is missing or throws in hardened browsers; use safeStorage from @open-elements/ui.
    files: ["src/**/*.{ts,tsx}"],
    ignores: ["src/**/__tests__/**", "src/test/**"],
    rules: {
      "no-restricted-globals": [
        "error",
        { name: "localStorage", message: "Use safeStorage from @open-elements/ui." },
        { name: "sessionStorage", message: "Storage can throw in hardened browsers; guard it." },
      ],
      "no-restricted-properties": [
        "error",
        { object: "window", property: "localStorage", message: "Use safeStorage from @open-elements/ui." },
        { object: "window", property: "sessionStorage", message: "Guard storage access." },
      ],
    },
  },
];
```

Both rules are needed. `no-restricted-globals` catches `localStorage.getItem(…)`, and
`no-restricted-properties` catches `window.localStorage`.

The same storage rules belong in the Open Elements libraries themselves; there, the file that
implements the helper is listed under `ignores`.

## Browser storage

Persisted values such as a page size, the language or a collapsed panel are conveniences. A failed
read must mean "nothing stored". It must never crash a render or abort an event handler halfway
through.

- Use `safeStorage` from `@open-elements/ui`. Its `get`, `set` and `remove` never throw
  ([open-elements-ui#19](https://github.com/OpenElementsLabs/open-elements-ui/issues/19)).
- Until an app is on a release that has it, wrap every access in `try/catch`. Getting the
  `localStorage` property can itself throw, so do it inside the `try`:

  ```ts
  function readStored(key: string): string | null {
    try {
      return typeof window === "undefined" ? null : window.localStorage.getItem(key);
    } catch {
      return null; // Storage unavailable: treat as nothing stored.
    }
  }
  ```

- A `typeof window === "undefined"` check only covers server rendering. It does **not** protect
  against a browser that blocks storage.
- Write the unit test for the failure path too: `vi.spyOn(window, "localStorage", "get")` throwing
  a `DOMException("…", "SecurityError")`, and `vi.stubGlobal("localStorage", null)`.

## Cross-browser smoke tests with Playwright

The smoke test logs in through the mock OIDC server once and then opens every main page. A page
fails on an uncaught error, a `console.error` or a 5xx response. That one simple test catches most
real-world breakage: hydration crashes, unguarded browser APIs, broken proxies and failed API calls.

### Setup

Pin a Playwright release that is at least two weeks old, and keep the CI container tag in sync with
it:

```bash
pnpm add -D -E @playwright/test@<version>
```

Add `"e2e": "playwright test"` to the scripts and `playwright-report/` and `test-results/` to
`frontend/.gitignore`. Vitest only includes `src/**`, so the `e2e/` folder does not collide with it.

### `frontend/playwright.config.ts`

```ts
import { defineConfig, devices } from "@playwright/test";

/** Privacy defaults of LibreWolf / hardened Firefox profiles. */
const HARDENED_FIREFOX_PREFS = {
  "privacy.resistFingerprinting": true,
  "privacy.resistFingerprinting.letterboxing": true,
  "browser.contentblocking.category": "strict",
  "network.cookie.cookieBehavior": 5,
  "privacy.trackingprotection.enabled": true,
  "privacy.trackingprotection.socialtracking.enabled": true,
  "privacy.query_stripping.enabled": true,
  "network.http.referer.XOriginTrimmingPolicy": 2,
  "dom.security.https_only_mode": true,
  "webgl.disabled": true,
  "media.peerconnection.enabled": false,
};

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 1 : 0,
  // Four browsers in parallel against one stack overload small CI runners (requests stall).
  workers: process.env.CI ? 2 : undefined,
  reporter: process.env.CI ? [["list"], ["html", { open: "never" }]] : "list",
  use: {
    baseURL: process.env.E2E_BASE_URL ?? "http://localhost:<frontend-port>",
    trace: "retain-on-failure",
    screenshot: "only-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
    {
      name: "firefox-hardened",
      use: { ...devices["Desktop Firefox"], launchOptions: { firefoxUserPrefs: HARDENED_FIREFOX_PREFS } },
    },
  ],
});
```

### `frontend/e2e/smoke.spec.ts`

```ts
import { expect, test, type Page } from "@playwright/test";

const STATIC_PAGES = ["/", "/<resource>", "/admin", "/admin/api-keys", "/admin/users" /* … */];

function collectProblems(page: Page): string[] {
  const problems: string[] = [];
  page.on("pageerror", (error) => problems.push(`uncaught error: ${error.message}`));
  page.on("console", (message) => {
    if (message.type() === "error") {
      const { url, lineNumber } = message.location();
      problems.push(`console error: ${message.text()} (${url}:${lineNumber})`);
    }
  });
  page.on("response", (response) => {
    if (response.status() >= 500) problems.push(`HTTP ${response.status()}: ${response.url()}`);
  });
  return problems;
}

async function logIn(page: Page, appOrigin: string) {
  await page.goto("/");
  await page.waitForURL(/\/login/);
  await page.getByRole("button").click();
  // Mock OIDC login form: any user name works.
  await page.locator('input[name="username"]').fill("e2e");
  await page.locator('input[type="submit"]').click();
  await page.waitForURL((url) => url.origin === appOrigin && !url.pathname.startsWith("/login"));
}

/** Detail pages need IDs: take the first matching link on the list page (needs seeded data). */
async function firstDetailPath(page: Page, listPath: string, linkSelector: string) {
  await page.goto(listPath);
  const link = page.locator(`main ${linkSelector}`).first();
  await expect(link, `a detail link on ${listPath} (is demo data seeded?)`).toBeVisible();
  return (await link.getAttribute("href"))!;
}

test("every main page loads without errors", async ({ page, context, baseURL }) => {
  test.setTimeout(180_000);
  // Console output during login is not checked: it includes the mock OIDC server's own page.
  await logIn(page, new URL(baseURL!).origin);
  const detailPages = [await firstDetailPath(page, "/<resource>", 'a[href^="/<resource>/"]')];

  for (const path of [...STATIC_PAGES, ...detailPages]) {
    await test.step(path, async () => {
      // A fresh tab per page: requests the previous page still had in flight die with it.
      const tab = await context.newPage();
      const problems = collectProblems(tab);
      try {
        await tab.goto(path);
        await tab.waitForLoadState("networkidle");
        await expect(tab).not.toHaveURL(/\/login/);
        await expect(tab.locator("main")).toBeVisible();
        expect(problems, `problems on ${path}`).toEqual([]);
      } finally {
        await tab.close();
      }
    });
  }
});
```

Rules that keep this test deterministic. Each one comes from a real flaky run:

- **One tab per page.** If you navigate a single tab, Firefox aborts the previous page's
  in-flight fetches, and their `console.error("Failed to load …")` lands on the next page.
- **Do not check console output during login.** The mock OIDC login page loads its own fonts over
  `http://`, which the hardened profile blocks. That page is not app code.
- **Wait for the detail link, not for `networkidle`.** Lists render after the session and the data
  have loaded, which can be after the network goes idle once. Fail with a clear message when there
  is no data. Do not silently skip the detail pages.
- **Raise the test timeout.** A single login followed by many pages takes longer than the default
  30 seconds.
- **Do not match visible text** for the login button or headings. The hardened profile switches the
  locale to `en-US`, so the language differs between projects.

### CI: `.github/workflows/e2e.yml`

```yaml
name: E2E
on:
  push: { branches: [main] }
  pull_request: { branches: [main] }
permissions:
  contents: read
jobs:
  e2e:
    runs-on: ubuntu-latest
    timeout-minutes: 30
    steps:
      - uses: actions/checkout@v7
      - name: Start stack
        run: |
          cp .env.example .env
          docker compose up -d --build --wait
      - name: Wait for backend and frontend
        run: |
          for url in http://localhost:<backend-port>/api/health http://localhost:<frontend-port>/login; do
            timeout 120 bash -c "until curl -fsS -o /dev/null $url; do sleep 2; done"
          done
      - name: Seed demo data
        run: python3 scripts/seed-demo-data.py
      - uses: pnpm/action-setup@v6
        with:
          package_json_file: "frontend/package.json"
      - uses: actions/setup-node@v7
        with:
          node-version-file: "frontend/.nvmrc"
          cache: "pnpm"
          cache-dependency-path: "frontend/pnpm-lock.yaml"
      - name: Install dependencies and browsers
        working-directory: frontend
        run: |
          pnpm install --frozen-lockfile
          pnpm exec playwright install --with-deps chromium firefox webkit
      - name: Run smoke tests
        working-directory: frontend
        run: pnpm e2e
      - name: Upload Playwright report
        if: failure()
        uses: actions/upload-artifact@v7
        with:
          name: playwright-report
          path: |
            frontend/playwright-report
            frontend/test-results
          retention-days: 14
      - name: Stack logs
        if: failure()
        run: docker compose logs --no-color
```

The stack in CI is the same one developers run locally: the mock OIDC server from
`docker-compose.override.yml` and seeded demo data. If the project has no seed script yet, add one
that only uses the Python standard library. It gets a token from the mock issuer and creates a
handful of records through the API.

### Running locally

Playwright's Firefox build may not start on recent macOS versions: the browser exits with
"Could not find profile folder" or hangs at launch. Run the suite in the Linux image instead. It is
the same environment as CI, and it reaches the local stack through host networking (Docker Desktop
needs host networking enabled):

```bash
cd frontend && docker run --rm --network host --ipc=host -v "$PWD":/work -w /work mcr.microsoft.com/playwright:v<version>-noble ./node_modules/.bin/playwright test
```

## When a user reports "it doesn't work in browser X"

1. Ask for the browser console output (F12 → Console and Network), and for the step that fails:
   login, the first page after login, or one specific page.
2. Reproduce with the matching Playwright project. For LibreWolf or Brave-like setups, extend
   `HARDENED_FIREFOX_PREFS` with the reporter's non-default settings (`about:config`).
3. Fix the cause, then add the page or the failure mode to the smoke test so it stays fixed.

## Checklist

- [ ] `browserslist` in `frontend/package.json` includes `Firefox ESR` and excludes Opera Mini
- [ ] `eslint-plugin-compat` is enabled for `src/**`, and `pnpm lint` passes
- [ ] No direct `localStorage`/`sessionStorage` access outside a guarded helper (enforced by lint)
- [ ] No third-party scripts, fonts or `http://` URLs that the app needs to work
- [ ] `frontend/e2e/smoke.spec.ts` lists every main page, including at least one detail page per resource
- [ ] `playwright.config.ts` has the four projects, including `firefox-hardened`
- [ ] `.github/workflows/e2e.yml` runs on pull requests against the Docker stack with seeded data
