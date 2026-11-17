---
name: open-elements-app-layout
license: Apache-2.0
metadata:
  source: https://github.com/OpenElementsLabs/agentic-support-coding
  author: Open Elements
description: Build the standard Open Elements application shell and view flow on @open-elements/nextjs-app-layer and @open-elements/ui — route groups, the thin page.tsx plus <route>-client.tsx split, component co-location, the dark-sidebar/light-content surface rules, sidebar composition, admin pages, i18n, the OIDC wiring and the backend proxy. Use whenever adding a page, view, sidebar entry or admin screen to an Open Elements app (open-crm, open-tasks, open-transcript, application-skeleton), or when starting a new one. For a greenfield Next.js project without the Open Elements libraries, use web-frontend-builder instead.
---

# Open Elements App Layout

The shell every Open Elements application shares, as it actually exists in `open-crm`, `open-tasks`
and `open-transcript`. This skill covers **structure** — where files go, what wraps what, which
surface gets which colours. For visual design quality use `frontend-design`; for the generic
Next.js/shadcn stack without the Open Elements libraries use `web-frontend-builder`.

The premise is the two-application layout: `backend/` and `frontend/` are fully independent
applications, no shared code, wired only over HTTP. Everything below is the frontend half.

## Packages

| Package | Role |
|---|---|
| `@open-elements/nextjs-app-layer` | root layout, OIDC auth factories, backend proxy, logout, login page, admin page factories |
| `@open-elements/ui` | design system, `LanguageProvider`, `Sidebar`, `NavItem`, primitives, brand stylesheet |
| `next-auth` (5.x) | peer dependency of the app layer — never wired by hand |
| `@base-ui/react`, `radix-ui`, `lucide-react` | peer dependencies of the UI library |

Pin exact versions. Both `@open-elements` packages must be listed in `transpilePackages` (see
[Build configuration](#build-configuration)).

## Route structure

```
src/
  app/
    layout.tsx                  OERootLayout + translations + globals.css
    globals.css
    login/page.tsx              OUTSIDE (app) — no sidebar
    api/
      auth/[...nextauth]/route.ts
      [...path]/route.ts        backend proxy
      logout/route.ts
    (app)/                      authenticated area, sidebar layout
      layout.tsx
      page.tsx                  landing
      <segment>/
        page.tsx                thin Server Component
        <segment>-client.tsx    "use client", all logic
        __tests__/<segment>-client.test.tsx
      admin/<segment>/page.tsx
  auth.ts
  middleware.ts
  components/                   SHARED components only (≥ 2 consumers)
  lib/
    api.ts                      hand-written typed client
    i18n/{de,en,index}.ts
    types.ts
  test/test-utils.tsx           renderWithProviders
```

`/login` lives outside the `(app)` group deliberately: it must render without the sidebar, and a
route group is the only way to give it a different layout.

## The view flow

**Every route is a thin Server Component plus a Client Component that does the work.**

```tsx
// src/app/(app)/companies/page.tsx
import { CompaniesClient } from "./companies-client";

export default function CompaniesPage() {
  return <CompaniesClient />;
}
```

```tsx
// src/app/(app)/companies/companies-client.tsx
"use client";
// hooks, state, effects, fetches — everything
```

This split is not stylistic. Server Components cannot use hooks or effects; Client Components cannot
call `auth()`. Keeping `page.tsx` thin means the role gate can live in the server half while the
client half stays unit-testable in isolation, with no server runtime in the test.

**Data is fetched in the client, not in `page.tsx`.** The backend is not reachable during
`next build`, so a Server Component that fetches at render time bakes an error state into the
statically prerendered page. Fetching from the client sidesteps this entirely. If a page genuinely
must fetch on the server, it needs `export const dynamic = "force-dynamic"` — `open-crm` sets this
defensively on its pages, `open-tasks` does not set it at all, and both work because their pages are
thin wrappers.

### Component co-location

| Used by | Lives in |
|---|---|
| exactly one route | next to that route's `page.tsx` |
| two or more routes, or a generic primitive | `src/components/` |

A component starts co-located and is promoted to `src/components/` only when a second route imports
it. The same rule applies to helper modules: a `webhook-status.ts` used only by
`webhooks-client.tsx` belongs beside it, not in `lib/`.

## Root layout

```tsx
// src/app/layout.tsx
import type { Metadata } from "next";
import { OERootLayout } from "@open-elements/nextjs-app-layer/layout";
import { translations } from "@/lib/i18n";
import "./globals.css";

export const metadata: Metadata = { title: "…", description: "…" };

export default function RootLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return <OERootLayout translations={translations}>{children}</OERootLayout>;
}
```

`OERootLayout` renders `<html>` and `<body>`, loads Montserrat and Lato, and mounts the full provider
stack (`SessionProvider`, `LanguageProvider`, `AppLayerTranslationProvider`, `ApiClientProvider`).
**Do not declare your own `<html>`/`<body>`** and do not assemble the providers by hand.

Note what this implies: `OERootLayout` mounts a `SessionProvider` that polls `/api/auth/session`
every two minutes. It is not usable before the auth routes exist — adopt the layout and the auth
wiring in the same change, never the layout first.

### `globals.css` — all five parts are required

```css
@import "@open-elements/ui/styles/brand.css";
@import "tailwindcss";

@source "../../node_modules/@open-elements/ui/src";
@source "../../node_modules/@open-elements/nextjs-app-layer/src";

body {
  font-family: var(--font-body);
  color: var(--color-oe-black);
  background-color: var(--color-oe-white);
}

h1, h2, h3, h4, h5, h6 {
  font-family: var(--font-heading);
}
```

Two failure modes, both silent:

- **Without the `@source` directives**, Tailwind never sees the class names used *inside* the two
  packages, so the shipped components render unstyled. Nothing errors.
- **Without the `body` and heading rules**, `OERootLayout` still loads the fonts and exposes
  `--font-heading` / `--font-body`, but nothing applies them: the app runs in the system font with the
  browser's default colours instead of the brand tokens. This also breaks colour inheritance in the
  sidebar (see below), so the symptom shows up somewhere else entirely.

### Required public asset

`SidebarHeader` renders `<img src="/oe-logo-landscape-dark.svg">` — an absolute path served from the
**consuming application's** `public/`. The library does not ship the file. Copy it from an existing
Open Elements app; without it the header shows a broken image next to "Developed by".

## The content area must be offset, not laid out in a row

The library's desktop sidebar is `hidden md:flex md:w-64 md:flex-col md:fixed md:inset-y-0`. It is
**`position: fixed`**, therefore out of flow and occupying no space in any parent.

```tsx
// src/app/(app)/layout.tsx — correct
export default function AppLayout({ children }: Readonly<{ children: React.ReactNode }>) {
  return (
    <TooltipProvider>
      <AppSidebar />
      <main className="bg-oe-white h-screen overflow-y-auto md:ml-64">
        <div className="p-6 md:p-8">{children}</div>
      </main>
      <Toaster richColors position="top-right" />
    </TooltipProvider>
  );
}
```

```tsx
// wrong — flex-1 fills the whole width and the fixed sidebar overlaps its first 16rem
<div className="flex min-h-screen">
  <AppSidebar />
  <main className="flex-1 p-8">{children}</main>
</div>
```

The offset is `md:ml-64` and nothing else: no flex wrapper, no grid. `h-screen overflow-y-auto` on
`<main>` makes the content area scroll rather than the page, which is what keeps the full-height
fixed sidebar stable.

## Surfaces and colours

The app has exactly **two** kinds of surface, and choosing classes for the wrong one produces
components that render *invisibly* rather than badly — which is why it survives review.

| Surface | Background | Use |
|---|---|---|
| Sidebar: desktop aside, mobile header, mobile drawer | `bg-oe-dark` (`#020144`) | `text-oe-white`, `text-oe-white/70`, `border-oe-white/10`, `bg-oe-white/10` |
| Everything else: `<main>`, cards, dialogs, forms | `bg-oe-white` | `text-oe-black`, `text-oe-black/70`, `border-oe-gray-light`, `bg-oe-gray-light` |

Both directions have gone wrong in production code:

- White-on-navy classes **omitted** in the sidebar's `bottomChildren` → black text on dark navy.
- A list written with `border-oe-white/20` and `bg-oe-white/10` rendered on the light content area →
  white borders on white, so an input looked non-editable.

Three rules follow.

**`opacity-70` is not a colour.** It dims whatever was inherited, which on the wrong surface is the
wrong colour to start with. Write `text-oe-white/70` or `text-oe-black/70`.

**Plain markup passed into a library component that owns a dark surface must state its own colour.**
The library's own children each set theirs and its containers set none, so anything you pass in
inherits the `body` colour. Today this affects `Sidebar`'s `bottomChildren`
([open-elements-ui#12](https://github.com/OpenElementsLabs/open-elements-ui/issues/12) proposes a
correct default; until it ships, set the colour).

**An interactive control needs a visible border on the light surface.** `border-oe-gray-mid` plus a
focus ring; a rounded box with no border reads as static text, not as an input.

The full palette comes from `brand.css`: `oe-dark`, `oe-black`, `oe-white`, `oe-gray-mid`,
`oe-gray-light`, and `oe-blue` / `oe-green` / `oe-red` / `oe-yellow` each with `-dark`, `-light` and
`-lighter` variants. Do not introduce colours outside it.

## Sidebar composition

```tsx
"use client";

export function AppSidebar() {
  const t = useTranslations();
  const pathname = usePathname();

  return (
    <Sidebar
      appTitle={t.app.title}
      homeHref="/companies"
      developedByText={t.app.developedBy}
      menuLabel={t.sidebar.menu}
      user={{ userName: …, avatarUrl: …, roles: …, onLogout: … }}
      bottomChildren={<AdminGroup />}
    >
      <NavItem href="/companies" icon={<Building2 className="h-5 w-5" />}
               label={t.nav.companies} active={pathname.startsWith("/companies")} />
    </Sidebar>
  );
}
```

- **`children`** are the primary navigation: `NavItem`, or `CollapsibleGroup` wrapping indented
  `NavItem`s. The mobile drawer closes itself on `NavItem` click — the library clones the children to
  add that, which only works for actual `NavItem` elements.
- **`bottomChildren`** is the secondary block, conventionally the admin group.
- **`user`** drives the user section, avatar and logout. It is optional; omit it and the section is
  simply absent.
- **`active`** is computed by the caller from `usePathname()`. Prefer `startsWith` for sections and
  an exact match where a sibling route would otherwise also match — `/tasks` must not light up on
  `/tasks/archive`.
- The **language switch is rendered by the library**. Do not add one.

## Admin pages

**Use the library factory when one exists.** One line, and the role gate comes with it:

```tsx
// src/app/(app)/admin/status/page.tsx
import { createServerStatusPage } from "@open-elements/nextjs-app-layer";
import { auth } from "@/auth";

export default createServerStatusPage({ auth, homeRoute: "/tasks" });
```

Factories exist for audit logs, users, server status, bearer token, API keys and webhooks
(`createAuditLogsPage`, `createUsersPage`, `createServerStatusPage`, `createBearerTokenPage`,
`createApiKeysPage`, `createWebhooksPage`), each with a matching `*Client` and `*PageMeta` export.

**For your own admin page**, gate in the Server Component:

```tsx
export default async function ThingPage() {
  const session = await auth();
  if (!session?.roles?.includes(ROLE_IT_ADMIN)) return <ForbiddenPage />;
  return <ThingClient />;
}
```

Never repeat the role check inside the client. Note that `open-tasks`' `docs/frontend.md` still
describes a `withAdminGuard()` helper — it no longer exists; the factories replaced it.

## Internationalisation

```
src/lib/i18n/
  de.ts      the German tree
  en.ts      the English tree, identical key paths
  index.ts   merge, typing, typed useTranslations()
```

```ts
// de.ts
import { de as uiDe } from "@open-elements/ui";

export const de = {
  ...uiDe,        // library first
  app: { … },     // application keys win on collision
  nav: { … },
} as const;
```

```ts
// index.ts
import { type Language, useTranslations as generic, useLanguage } from "@open-elements/ui";

type DeepStringify<T> = { readonly [K in keyof T]: T[K] extends string ? string : DeepStringify<T[K]> };
export type Translations = DeepStringify<typeof de>;
export const translations: Record<Language, Translations> = { de, en };
export function useTranslations(): Translations { return generic<Translations>(); }
```

**Spread `...uiDe` first.** `open-crm` and `open-tasks` both spread it last, which makes library keys
override application keys — so an app cannot override a library string even when it wants to. Nothing
collides today, which is why neither has noticed.

No user-facing string is ever hard-coded in JSX. `en.ts` must carry the same key paths as `de.ts`; a
test comparing the two key sets is cheap and catches every omission.

## Auth wiring

Six small files, all thin calls into the library. Nothing here is hand-written.

```ts
// src/auth.ts
import "@open-elements/nextjs-app-layer/server/next-auth-types";
import { createAppLayerAuth } from "@open-elements/nextjs-app-layer/server";

export const { handlers, auth, signIn, signOut, oidcIssuer } = createAppLayerAuth({
  issuer: process.env.OIDC_ISSUER_URI,
  clientId: process.env.OIDC_CLIENT_ID,
  clientSecret: process.env.OIDC_CLIENT_SECRET,
});
```

| File | Content |
|---|---|
| `src/app/api/auth/[...nextauth]/route.ts` | `export const { GET, POST } = handlers` |
| `src/app/api/[...path]/route.ts` | `createBackendProxyHandler({ backendUrl, auth })`, exported for `GET`, `POST`, `PUT`, `PATCH`, `DELETE` |
| `src/app/api/logout/route.ts` | `createLogoutHandler({ auth, oidcIssuer, authUrl })` |
| `src/app/login/page.tsx` | `createLoginPage({ appName, homeRoute })` |
| `src/middleware.ts` | `export { auth as middleware }` plus a matcher |

**Export all five HTTP methods from the proxy.** `open-crm` exports four and would answer `405` to a
`PATCH`, with nothing to point at the cause.

Session lifetime, refresh skew and OIDC timeouts belong to the library. Declare
`AUTH_SESSION_MAX_AGE_SECONDS`, `AUTH_SESSION_UPDATE_AGE_SECONDS`,
`AUTH_TOKEN_REFRESH_SKEW_SECONDS` and `OIDC_HTTP_TIMEOUT_MS` **blank** in `.env.example`: the library
treats an empty value as unset and applies its own default, so it stays the single source of truth
instead of the default being duplicated and drifting.

**Read required environment variables without a fallback.** Both reference apps write
`process.env.BACKEND_URL ?? "http://localhost:8080"`, which makes a misconfigured production
deployment silently proxy to its own container. Fail at startup with the variable named.

### The proxy is a route handler, not a rewrite

Do **not** put an `/api/:path*` rewrite in `next.config.ts`, and do **not** declare a `BACKEND_URL`
build argument in the Dockerfile or `build.args` in Compose. The catch-all route handler reads
`BACKEND_URL` at **request** time, and a route handler takes precedence over an `afterFiles` rewrite,
so a rewrite would be dead configuration. `BACKEND_URL` stays a runtime variable, which is what lets
one image be repointed at another backend.

Both reference apps still carry the vestigial build argument from the rewrite era
([open-tasks#53](https://github.com/OpenElementsLabs/open-tasks/issues/53)) and the
`fullstack-architecture` convention still documents the rewrite approach as **IMPORTANT**. Neither is
current — do not copy either.

**Caveat for binary payloads:** the proxy buffers the whole request body in memory
(`await req.arrayBuffer()`) and forwards neither `Range` nor arbitrary request headers. Fine for JSON,
unusable for file uploads or ranged media until
[nextjs-app-layer#10](https://github.com/OpenElementsLabs/nextjs-app-layer/issues/10) ships.

## Build configuration

```ts
// next.config.ts
const nextConfig: NextConfig = {
  output: "standalone",
  // Both packages publish ESM whose relative specifiers carry no extension, which Node's ESM
  // resolver rejects — Next must transpile them from source. See nextjs-app-layer#9.
  transpilePackages: ["@open-elements/ui", "@open-elements/nextjs-app-layer"],
};
```

`@open-elements/nextjs-app-layer` genuinely needs the entry: importing it under Node fails with
`ERR_MODULE_NOT_FOUND`. `@open-elements/ui` resolves on its own but is listed for symmetry. Keep the
comment and the issue link — the entries look redundant and get removed otherwise, and the breakage
surfaces in the Docker build rather than in `next dev`.

Vitest needs an alias for the same reason:

```ts
resolve: {
  alias: {
    "@": path.resolve(__dirname, "./src"),
    "@open-elements/nextjs-app-layer": path.resolve(
      __dirname, "./node_modules/@open-elements/nextjs-app-layer/src/index.ts"),
  },
},
```

## Naming

| Item | Convention | Example |
|---|---|---|
| file name | kebab-case | `audit-logs-client.tsx` |
| route client component | PascalCase, `Client` suffix | `AuditLogsClient` |
| test file | mirrors the source, in a sibling `__tests__/` | `audit-logs-client.test.tsx` |
| list route slug | kebab-case plural | `/api-keys`, `/audit-logs` |
| API function | verb + entity in `src/lib/api.ts` | `getAuditLogs`, `createApiKey` |
| page-size storage key | `pageSize.<resource>`, camelCase resource | `pageSize.auditLogs` |

## Tests

- Vitest picks up `src/**/*.test.{ts,tsx}` — anything outside `src/` is invisible to the runner.
- Tests live in a `__tests__/` folder beside the source.
- Render with `renderWithProviders` from `src/test/test-utils.tsx` when the component needs
  `LanguageProvider` or `SessionProvider`; most route clients do.
- Mock `@/lib/api` and, where read, `next/navigation` (`useRouter`, `usePathname`,
  `useSearchParams`).
- Mock `next-auth/react`: `SessionProvider` fetches `/api/auth/session` on mount and keeps a
  two-minute refetch timer, which otherwise makes every component test noisy.
- Run `pnpm test` and `pnpm exec tsc --noEmit`.
- The root `layout.tsx` cannot be unit-tested: `next/font/google` is a **0-byte module** at runtime
  (Next's SWC transform replaces the call at build time), so importing it in jsdom throws
  `TypeError: … is not a function`. Exclude that one file from coverage with the reason in the config,
  rather than mocking the font module to assert nothing.

- Cross-browser smoke tests (`frontend/e2e/`, Playwright) open every page in Chromium, Firefox,
  WebKit and a hardened Firefox profile in CI; set them up with the `browser-compatibility` skill.

## Adding a view — checklist

1. `src/app/(app)/<segment>/page.tsx` — thin Server Component, role gate here if needed.
2. `<segment>-client.tsx` with `"use client"` and all logic.
3. `__tests__/<segment>-client.test.tsx`.
4. API functions in `src/lib/api.ts`, types in `src/lib/types.ts`.
5. German **and** English strings in `src/lib/i18n/{de,en}.ts`.
6. A `NavItem` in the sidebar with an `active` predicate that does not also match sibling routes.
7. Colours for the **light** surface — this is the content area, not the sidebar.
8. Loading state as `Skeleton` rows; page size persisted under `pageSize.<resource>` through a guarded
   storage helper (`safeStorage` from `@open-elements/ui`), never raw `localStorage`.
9. The new route in `STATIC_PAGES` of `frontend/e2e/smoke.spec.ts` (plus its detail page, if any).

## Known library issues to be aware of

| Issue | Effect |
|---|---|
| [nextjs-app-layer#9](https://github.com/OpenElementsLabs/nextjs-app-layer/issues/9) | published ESM has extension-less relative imports → needs `transpilePackages` and a Vitest alias |
| [nextjs-app-layer#10](https://github.com/OpenElementsLabs/nextjs-app-layer/issues/10) | proxy buffers request bodies and drops `Range` → unusable for binary payloads |
| [open-elements-ui#19](https://github.com/OpenElementsLabs/open-elements-ui/issues/19) | `TablePagination` writes the page size to `localStorage` unguarded → changing the page size throws in browsers that block storage |
| [nextjs-app-layer#14](https://github.com/OpenElementsLabs/nextjs-app-layer/issues/14) | admin pages read the page size from `localStorage` unguarded → they crash in browsers that block storage |
| [open-elements-ui#12](https://github.com/OpenElementsLabs/open-elements-ui/issues/12) | `Sidebar` sets no text colour on its dark surfaces → plain `bottomChildren` render black on navy |
| [open-tasks#53](https://github.com/OpenElementsLabs/open-tasks/issues/53) | vestigial `BACKEND_URL` build argument, copied from the rewrite era |
