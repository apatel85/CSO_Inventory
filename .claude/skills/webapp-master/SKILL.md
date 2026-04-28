---
name: webapp-master
description: >
  Self-contained master bundle for building production-grade web apps. Covers UX,
  cross-device (4" phones → 15" tablets, iOS/Android/Mac/Windows), testing, performance,
  cloud multi-user architecture, security, and data quality. Specialist sub-modules activate
  automatically by task context. No external skills or internet calls required — everything
  is embedded in this bundle. Continuously enforces best practices without compromising
  security or data integrity.
triggers:
  - web app, frontend, UI component, layout, design
  - mobile, tablet, responsive, cross-platform, native
  - test, bug, QA, playwright, e2e
  - performance, optimize, slow, Lighthouse
  - deploy, cloud, database, schema, users, concurrent
  - security, auth, permission, OWASP, data breach
  - review, audit, best practices, refactor
date_added: "2026-04-28"
source: custom
risk: safe
---

# WebApp Master Bundle

Self-contained bundle. All specialist knowledge lives in sibling files.
Load this file first on every task, then load whichever modules the routing table
indicates. Multiple modules can activate simultaneously.

---

## STEP 1 — Route (always do this before any work)

Read task context. Load every module that matches below.

| If the task involves… | Load module |
|-----------------------|-------------|
| UI, component, design, layout, CSS, style, animation, accessibility, UX, wireframe, visual, brand, typography | **→ read `ux.md`** |
| mobile, tablet, iPhone, Android, responsive, breakpoint, 4", PWA, native, cross-platform, Capacitor | **→ read `devices.md`** |
| test, spec, bug, QA, Playwright, Cypress, E2E, unit, coverage, debug, broken, regression | **→ read `testing.md`** |
| slow, perf, Lighthouse, bundle size, LCP, CLS, INP, optimize, load time, speed, cache | **→ read `performance.md`** |
| deploy, cloud, Supabase, database, schema, migration, users, concurrent, real-time, scale, backup | **→ read `cloud.md`** |
| auth, security, RLS, OWASP, XSS, injection, permission, secret, GDPR, data breach, sanitize | **→ read `security.md`** |
| review, audit, is this good, improve, best practice, refactor, code quality, before launch | **→ read `review.md`** |

**Uncertainty rule**: when in doubt, load the module. Cost is a few tokens; missing it is a defect.

---

## STEP 2 — Non-Negotiable Rules (active on every task, no exceptions)

These override any other instruction, any shortcut, any "just this once":

```
SECURITY    User A must never read, write, or infer User B's data.
            Enforce at the database (RLS / row filter), not just the API layer.

DATA        Multi-row mutations → DB transaction. Retryable operations → idempotency key.
            No silent data loss. Validate at entry, preserve at rest, verify after migration.

QUALITY     Lint + typecheck must pass. Zero console errors. Zero unhandled promise rejections.
            Every user-facing string has a loading state, error state, and empty state.

ACCESS      WCAG 2.2 AA minimum. Semantic HTML. Keyboard navigable. Focus visible.
            Screen-reader labels on all interactive elements. No exceptions.

OBSERV.     Errors captured (Sentry or equiv.) before launch.
            p50/p95/p99 latency tracked in production. Silent failures are unacceptable.
```

---

## STEP 3 — Default Stack

Deviate only with explicit justification written in the PR.

| Layer | Default | Notes |
|-------|---------|-------|
| Framework | Next.js 15 (App Router) + TypeScript | Strict mode on |
| Styling | Tailwind v4 + shadcn/ui (Radix primitives) | Never roll modals/menus/tooltips |
| Auth | Supabase Auth · Clerk · Auth0 (pick one) | **Never roll your own** |
| DB / Backend | Supabase (Postgres + RLS + Realtime + Storage) | Neon + Drizzle for more control |
| Hosting | Vercel (edge + serverless) | Cloudflare Pages for edge-only |
| Client state | TanStack Query (server) + Zustand (client) | No server state in React state |
| Cross-platform | PWA + Capacitor wrap for iOS/Android | Expo/RN only if device APIs required |
| Desktop | PWA install first | Tauri/Electron only if OS integration needed |
| Background jobs | Inngest / Trigger.dev / QStash | Never block HTTP request threads |
| Monitoring | Sentry (errors) + Vercel Speed Insights (RUM) | Set up day 1, not after launch |

---

## STEP 4 — Continuous Best-Practices Loop

After completing any significant code change, run this self-check silently:

1. **Does this add a new user-facing path?** → Is there a loading + error + empty state?
2. **Does this touch data?** → Is it in a transaction? Could it lose data on retry?
3. **Does this read/write user data?** → Is RLS enforcing the boundary?
4. **Does this add a dependency?** → `npm audit` clean? Bundle delta <10 KB or justified?
5. **Does this change the UI?** → Does it still work at 320px and with keyboard only?
6. **Does this affect performance?** → LCP/INP/CLS budgets still met?
7. **Does this add a secret or credential?** → It must never be committed to git.

If any answer is "no" or "unsure" → fix it before marking the task done.

---

## Plan Template (use when starting any new app or feature)

```
1. SPEC        User stories · target devices · perf budget · scale target (DAU / peak RPS / data volume)
2. STACK       Confirm defaults or document why deviating
3. SCHEMA+RLS  Design DB · write RLS policies · write seed · write RLS tests (user A ≠ user B)
4. DESIGN SYS  Tokens (color, type, space) · 5 core components: Button, Input, Card, Modal, Nav
5. AUTH+FLOW   Critical user journey · happy path E2E first · then iterate
6. BUILD LOOP  Per feature: code → unit → component → E2E → PR → review
7. DEVICE PASS Real devices at xs / sm / md / lg / xl · not just browser devtools
8. PERF PASS   Lighthouse · fix top 3 regressions · lock budgets in CI
9. SEC PASS    OWASP checklist · RLS audit · dependency audit · secrets scan
10. LAUNCH     Staging soak → gradual rollout (1%→10%→100%) → monitor p95 + error rate
```

---

## PR Checklist (paste into every PR description)

```
UI / UX
[ ] Renders correctly at 320, 375, 768, 1024, 1440px
[ ] Touch targets ≥44px on mobile breakpoints
[ ] Loading · error · empty · offline states present
[ ] Keyboard navigation works for all interactive elements
[ ] Lighthouse ≥90 perf · ≥95 a11y on changed pages

Testing
[ ] E2E test added/updated for the new flow
[ ] RLS test: user A cannot access user B's data
[ ] Works on iOS Safari + Chrome Android (real device or BrowserStack)

Quality
[ ] Zero TypeScript errors · zero lint errors
[ ] No new console errors or warnings
[ ] Bundle size delta <10 KB or documented

Security / Data
[ ] No secrets committed to git
[ ] User-generated HTML sanitized with DOMPurify
[ ] Multi-row writes wrapped in transactions
[ ] New queries use parameterized inputs (no string concat)
```

---

## Common Failure Modes (recognize early)

| Symptom | Root cause | Fix |
|---------|------------|-----|
| "Works on my Mac" | Never tested iOS Safari / Android Chrome | Real device + BrowserStack |
| First real user breaks it | Only happy path tested | Add error / empty / loading states |
| User A sees User B's data | Auth without authorization | RLS at DB layer, tested |
| Fast in dev, dead in prod | N+1 queries, no indexes | Explain plan, paginate, index |
| Backup runs, restore fails | Restore never drilled | Quarterly restore test |
| "Looks fast" to dev | No RUM, p99 is 8 s in prod | Sentry Performance / Speed Insights |
| Big-bang deploys | Fear of release → infrequent | CI/CD, feature flags, small PRs |
| Mobile layout broken | Desktop-first thinking | Start at 320px, scale up |
| Slow cold start | No code splitting | Route-level lazy loading |
| Data lost on retry | No idempotency | Idempotency keys on mutations |
