---
name: chayen-frontend
description: Web UI knowledge for builder and verifier bots - Angular, React, Next.js, Vue, Nuxt, and Vite apps, plus PWA, Capacitor, Ionic, Electron, and Tauri shells. Covers components, state, forms, design systems, every UI state (loading, empty, error, permission, offline), accessibility, i18n, frontend performance, error handling, and a no-generic-look visual check. Use with chayen-build or chayen-verify. Not for server-only code or native UI (SwiftUI, Flutter, React Native, WPF, Unity).
---

# Frontend

Build or check UI that matches the project's framework version, design
system, and conventions, with every state handled and accessible.

**Operating mode.** Under `chayen-sdlc-loop`: do not ask; take behaviour from
the spec and patterns from the nearest analogous screen, record
assumptions, and declare risky changes. If that skill is not loaded: never
invent API fields, permissions, or UX rules; never put secrets in client
code.

## Detect first

Read the owning app's `package.json`, lock file, and config before writing.

| Signal | Framework notes |
|---|---|
| `@angular/core` | standalone or NgModules as the project uses; signals only if the version and code already use them; `ng build`, `ng test` or the configured runner |
| `react` without `next` | router, state (Redux, Zustand, React Query), forms (React Hook Form, Formik) as installed |
| `next` | App Router or Pages Router as present; server vs client components (`'use client'`); no browser APIs during server render |
| `vue` without `nuxt` | Options or Composition API as the code uses; Pinia or Vuex as installed |
| `nuxt` | auto-imports, `useFetch`/`useAsyncData`, server routes are backend work |
| `@capacitor/*`, `@ionic/*`, `electron`, `@tauri-apps/*`, service worker | shell present; platform builds need the SDKs |

Use only APIs available in the installed versions. Do not upgrade as a side
effect.

## Reuse before creating

Design system components and tokens, form helpers, HTTP services or hooks
or composables, state stores, icon set, and the styling approach (CSS
modules, Tailwind, SCSS, styled components). A new UI library is a
dependency change (`chayen-build` rules).

## Every data-bound view has its states

| State | Required behaviour |
|---|---|
| Loading (first) | skeleton or spinner in the region, `aria-busy="true"`, focus not moved |
| Loading (refresh) | keep old data visible with a subtle indicator |
| Empty | distinct message; first-use differs from "no results for these filters" (offer clear filters) |
| Error | plain message, Retry button, no server internals; keep previous data |
| Retry | manual re-runs the same request; automatic only for idempotent reads with bounded backoff |
| 403 | "you cannot access this" view, no Retry; hide or disable actions, server still enforces |
| 401 | the app's re-auth flow, return to the intended route, keep unsaved input when possible |
| 404 | not-found view with a way back |
| Offline | banner with what still works and pending changes, `role="status"` |
| Submitting | submit disabled, progress shown, no double submit |
| Success | visible confirmation, focus or navigation as existing flows do |
| Partial | show what loaded; failed section has its own retry |

Test each applicable state with the project's network mocking (MSW,
`HttpTestingController`, Playwright route interception).

## Accessibility is part of done

Semantic elements first (`button`, `a`, `label`, headings in order), every
input labelled, keyboard reachable and operable, visible focus, focus
managed on dialogs and route changes, errors announced (`role="alert"` for
action errors), colour contrast WCAG AA, no information by colour alone,
images with meaningful `alt` or empty `alt` when decorative.

## Visual check: no generic generated look

Every visible change avoids: two-hue gradients and gradient text, glows and
heavy shadows, emoji as icons, tracked uppercase eyebrow labels, hype or
greeting copy ("Welcome back! 🚀"), decorative looping animation, one-off
spacing and colours outside the tokens, and identical card grids where the
content is not uniform. Keep the frame quiet and give emphasis to the one
piece of content that matters. Use the product's own words. Fix these in
the code you touch; report others outside the change. Global theme or token
changes need the spec.

## Other concerns

| Concern | Rules |
|---|---|
| Forms | validation from the spec mirrored client-side, server is authoritative; errors next to fields and summarized; preserve input on failure |
| State | server state in the query/cache layer the project uses; avoid duplicating it in global stores |
| i18n | all user text through the project's i18n; no concatenated sentences; locale-aware dates, numbers, currency; RTL if the project supports it |
| Performance | lazy-load routes and heavy components; no new large dependency for a small helper; images sized and lazy; check bundle report if the project has one |
| Errors | use the project's boundary or interceptor; report to its error tracker; no secrets or tokens in logs |
| Security | no `innerHTML`, `dangerouslySetInnerHTML`, `v-html`, or `bypassSecurityTrust*` with untrusted data; no secrets in bundles or env files shipped to the client |
| Shells | keep Electron `contextIsolation` and sandbox on, no Node integration in renderers; do not widen Tauri permissions; native permissions and signing are spec-level changes |

## Bug path

Reproduce with a failing component test or a Playwright step; find the cause
(stale state, effect dependencies, change detection, hydration mismatch,
cache, lifecycle); fix minimally; keep the test.

## Checks before handing off

| Check | Evidence |
|---|---|
| Build, typecheck, lint | project commands PASS |
| Component and unit tests | new tests listed and PASS |
| States | each applicable state tested or listed as not verified |
| Accessibility | lint rules (for example `eslint-plugin-jsx-a11y`, Angular ESLint template a11y) PASS; manual notes for what tooling cannot check |
| Visual | screenshot or Playwright evidence when the project supports it; otherwise "not verified in browser" |
| Shell | sync or package command PASS, or NOT_RUN with the missing SDK named |

## Failure handling

| Situation | Action |
|---|---|
| Installed version lacks the planned API | Use the supported pattern; note the upgrade as a new intent |
| No component test setup | Verify with build and typecheck; propose the setup as a new intent; do not scaffold one inside a feature PR |
| API contract unknown or differs from spec | `blocked`; never invent fields |
| SSR or hydration errors | look for browser-only APIs during server render, dates, random values; report the cause |
| Platform SDK or signing missing | do the web part; platform build NOT_RUN with exact commands |
