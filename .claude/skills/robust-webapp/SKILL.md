---
name: robust-webapp
description: Master reference for building production-grade web apps that deliver excellent UX, work reliably across all devices and platforms (iOS/Android/Mac/Windows, 4"-15"), scale on cloud, and maintain data integrity for multiple concurrent users. Use when starting a new web app, reviewing one for robustness, or planning a release.
---

# Robust Web App Builder

Single skill that covers UX, cross-device, testing, performance, and cloud/multi-user concerns. Use the **Plan template** to start a new app, the **PR checklist** to ship, and the deep-dive references for specialized work.

## Default stack (override only with reason)

- **Framework**: Next.js 15 (App Router) + TypeScript
- **Styling**: Tailwind v4 + shadcn/ui (Radix primitives)
- **Backend**: Supabase (Postgres + Auth + Realtime + Storage) — or Neon + Clerk if more control needed
- **Hosting**: Vercel (edge + serverless) — or Cloudflare Pages
- **State**: TanStack Query (server) + Zustand (client) — never put server state in React state
- **Native wrap**: Capacitor for iOS/Android. Expo + React Native only when device APIs require it.
- **Desktop**: PWA install first; Tauri/Electron only if OS integration needed.

This stack means **one codebase → web + iOS + Android + Mac/Windows desktop**.

---

## 1. UX / UI

### Principles
- Mobile-first: design at 375px, scale up
- 1 primary action per screen, max 5 nav items
- Touch targets ≥44×44px (iOS) / ≥48×48dp (Material)
- <100ms = instant; show skeleton at >300ms; show progress at >1s
- Always show loading, empty, error, offline states
- Optimistic updates for user actions <500ms latency
- WCAG 2.2 AA: semantic HTML, alt text, focus rings, keyboard nav, contrast ≥4.5:1, prefers-reduced-motion respected

### Design system
- 1 type scale (modular ratio 1.125 or 1.25)
- Color in OKLCH (perceptually uniform); tokens for primary, neutral, success/warn/error
- 4-pt or 8-pt spacing grid
- Reuse Radix/shadcn for modals, menus, tooltips — never roll your own
- Dark mode via CSS variables (`--bg`, `--fg`), not duplicated classes
- One motion language: 150ms ease-out for UI, 300ms for transitions

### Anti-patterns
- Generic gradient hero + 3 feature cards = AI slop. Pick a clear aesthetic stance.
- Modal-on-modal-on-modal; nested dropdowns
- Carousels for primary content
- Disabled buttons with no explanation

→ Deep dive: `@frontend-design`, `@mobile-design`, `@design-taste-frontend`, `@high-end-visual-design`

---

## 2. Cross-platform / cross-device

### Breakpoints

| Class | Width | Targets |
|-------|-------|---------|
| xs | 320–479 | small phones (4–4.7") |
| sm | 480–767 | phones (5–7") |
| md | 768–1023 | tablets (7–10") |
| lg | 1024–1439 | tablets/laptops (11–15") |
| xl | 1440–1919 | desktop |
| 2xl | 1920+ | large desktop |

### Rules
- `clamp(min, vw, max)` for fluid type and spacing
- Container queries (`@container`) for component-level responsiveness
- Test on real devices: iPhone SE (4.7"), iPhone 16 Pro Max (6.9"), Pixel 8, iPad mini (8.3"), iPad Pro (12.9"), MacBook 13", 27" desktop
- Safe areas: `env(safe-area-inset-*)` for notched devices
- Use `100dvh` not `100vh` (iOS Safari bug)
- Pointer detection: `@media (hover: hover)` for hover effects, larger targets on touch
- Respect `prefers-color-scheme`, `prefers-reduced-motion`, `prefers-contrast`

### Platform-specific
- **iOS Safari**: tap delays gone with `touch-action`; momentum scroll with `-webkit-overflow-scrolling: touch`; date inputs render natively, can't fully style
- **Android Chrome**: address bar resize triggers viewport changes; use `dvh`
- **Mac/Windows**: keyboard shortcuts (Cmd vs Ctrl); right-click menus; trackpad scroll inertia
- **PWA install**: `manifest.json`, service worker, offline shell, install prompt UX

### Native via Capacitor
- Push notifications: Capacitor Push Notifications + APNs/FCM
- Camera/files: Capacitor plugins
- Test inside iOS Safari WebView and Android WebView (not just Chrome desktop)
- App Store/Play Store: prepare screenshots at required resolutions, privacy manifest, age rating

→ Deep dive: `@mobile-developer`, `@expo-ui-swift-ui`, `@android-jetpack-compose-expert`, `@ios-developer`

---

## 3. Testing

### Pyramid (with target coverage)
1. **Unit** (Vitest) — pure functions, hooks. Goal: >80% on logic
2. **Component** (Testing Library) — render + interact. Goal: every interactive component
3. **Integration** (Vitest + MSW) — API + state flows
4. **E2E** (Playwright) — critical journeys, run per PR
5. **Visual regression** (Playwright snapshots or Chromatic) — catch CSS drift
6. **Accessibility** (axe-playwright) — automated a11y on every page
7. **Performance** (Lighthouse CI) — budgets enforced
8. **Manual** — exploratory pass on real devices before each release

### What to test (must-have list)
- Happy path + edge cases: empty, error, loading, offline, slow network
- Auth: signup, login, logout, session expiry, password reset, OAuth
- Authorization: user A cannot read/edit user B's data (RLS test)
- Concurrency: two tabs editing same record → no data loss
- Network: disconnect mid-action, retry on reconnect, idempotent writes
- Scale: 1k / 10k / 100k records — pagination, virtualization, search
- Forms: validation, submit, error recovery, browser back, autofill
- File upload: large file, wrong type, network drop mid-upload
- i18n / RTL if applicable
- Browser back/forward, deep links, refresh mid-flow

### CI gates (block merge if any fail)
- Lint (ESLint) + format (Prettier) + typecheck (tsc --noEmit)
- Unit + integration tests pass
- Playwright E2E on Chromium + WebKit + Firefox
- Lighthouse CI: perf ≥90, a11y ≥95, best-practices ≥95, SEO ≥90
- Bundle size budget enforced (next-bundle-analyzer or size-limit)
- Visual regression: 0 unintentional diffs

→ Deep dive: `@e2e-testing`, `@find-bugs`, `@bug-hunter`, `@debugging-strategies`, `@javascript-testing-patterns`

---

## 4. Performance

### Budgets (Core Web Vitals — set in CI)

| Metric | Target | Hard fail |
|--------|--------|-----------|
| LCP | <2.5s | >4s |
| INP | <200ms | >500ms |
| CLS | <0.1 | >0.25 |
| TTFB | <800ms | >1.8s |
| JS initial bundle (gzip) | <170 KB | >300 KB |
| Image weight (above fold) | <500 KB | >1 MB |
| Total page weight | <1.5 MB | >3 MB |

### Tactics by layer

**Network**
- HTTP/2 or HTTP/3, Brotli compression
- Preconnect/preload critical origins and fonts
- CDN for all static assets, edge cache for SSR responses
- HTTP cache headers correct (immutable for hashed assets, SWR for HTML)

**Render**
- Server components / SSR / SSG for first paint
- Stream HTML (React Suspense) for fast TTFB
- Lazy-hydrate non-critical islands
- Avoid layout thrash: batch DOM reads/writes, use `transform` not `top/left` for animation

**JavaScript**
- Code-split at route boundary
- Dynamic `import()` for below-fold and modal content
- Tree-shake; avoid lodash whole-import; prefer native or per-function imports
- Remove unused dependencies (depcheck, knip)

**Images / fonts**
- Next/Image or `<picture>` with AVIF + WebP fallback
- Responsive `srcset` + `sizes`
- Lazy-load below fold
- `font-display: swap`, self-host or `next/font`, subset to needed glyphs

**Data**
- DB indexes on FKs and filtered/sorted columns
- Paginate (cursor-based for large sets), don't `SELECT *`
- Avoid N+1: batch queries, use joins or DataLoader
- Cache: TanStack Query client cache + server cache (Redis or edge KV)

**Measurement**
- Lighthouse + WebPageTest pre-launch
- Real-User Monitoring: Vercel Speed Insights, Sentry Performance, or similar
- Track p50 / p95 / p99 — p99 is what frustrated users feel

→ Deep dive: `@performance-engineer`, `@performance-optimizer`, `@react-component-performance`, `@postgresql-optimization`

---

## 5. Cloud + multi-user data

### Architecture (default)
- **Edge**: Vercel/Cloudflare for app, ISR/SSR
- **DB**: Supabase Postgres (or Neon) — pick region near most users
- **Files**: Cloudflare R2 / S3 with signed URLs
- **Realtime**: Supabase Realtime / Liveblocks for collab
- **Background jobs**: Inngest / Trigger.dev / QStash — never block requests

### Multi-user data integrity (non-negotiable)
- **Auth**: managed (Supabase Auth, Clerk, Auth0). Never roll your own.
- **Authorization**: Row-Level Security (Postgres RLS) — enforce at DB, not just API. Test it.
- **Concurrency**:
  - Optimistic locking: `WHERE updated_at = $expected` on UPDATE; reject if mismatch
  - Or CRDT (Yjs / Liveblocks) for true real-time collab
- **Transactions**: wrap multi-row writes in DB transactions
- **Idempotency keys**: required for any mutation that may retry (payments, webhooks, mobile flaky network)
- **Migrations**: versioned, reversible, tested on staging copy of prod data
- **Backups**: automated daily + point-in-time recovery + restore drill quarterly

### Scale
- Stateless app servers — sticky sessions only when WebSocket-bound
- DB: read replicas for read scale; partitioning for >100M rows
- Rate limit at edge: per-user + per-IP (Upstash Ratelimit, Cloudflare)
- Queues for spiky / slow work (email, image processing, exports)

### Observability (set up day 1, not after launch)
- Structured logs with request ID propagated end-to-end
- Metrics: p50 / p95 / p99 latency, error rate, RPS per route
- Error tracking: Sentry (frontend + backend, source maps uploaded)
- Uptime: Better Stack / Pingdom external check
- Alerts: error rate >1%, p95 latency >2x baseline, DB connection saturation

### Security baseline (review before launch)
- HTTPS only, HSTS preload
- CSP headers, no `unsafe-inline`, nonce for inline scripts if needed
- Sanitize user HTML with DOMPurify; parameterized queries everywhere
- Secrets in env / Vault; never commit. Use Doppler/1Password Connect for sharing.
- Auth: rate-limit login, lockout after N fails, MFA available
- OWASP Top 10 checklist
- `npm audit` / Snyk in CI
- GDPR / data export + deletion endpoints if EU users

→ Deep dive: `@database-architect`, `@nextjs-supabase-auth`, `@postgres-best-practices`, `@cloud-penetration-testing`

---

## Plan template (use when starting any new app)

1. **Spec** — user stories, target devices, perf budget, scale target (DAU, peak RPS, data volume)
2. **Stack** — confirm default stack or document why deviating
3. **Schema + RLS** — design DB, write RLS policies, write seed data, write RLS test
4. **Design tokens + 5 core components** — Button, Input, Card, Modal, Nav. Storybook them.
5. **Auth + first critical flow** — happy path E2E test before iterating
6. **Build loop per feature** — code → unit → component → E2E → PR
7. **Cross-device pass** — real devices on each breakpoint, not just devtools
8. **Perf pass** — Lighthouse, fix top 3 regressions, lock budgets in CI
9. **Security pass** — OWASP checklist, run `@security-review`, dependency audit
10. **Soak + launch** — staging soak with synthetic traffic, gradual rollout (1% → 10% → 100%), monitor

---

## PR checklist (paste into every PR description)

- [ ] Renders correctly at 320, 375, 768, 1024, 1440px
- [ ] Touch targets ≥44px on mobile breakpoints
- [ ] Keyboard navigation works for all interactive elements
- [ ] Loading, error, empty, offline states present
- [ ] Lighthouse ≥90 perf, ≥95 a11y on changed pages
- [ ] E2E test added/updated for the new flow
- [ ] RLS prevents cross-user data leak (tested)
- [ ] Works in Safari iOS and Chrome Android (real device check)
- [ ] No new console errors or warnings
- [ ] No new TypeScript errors
- [ ] Bundle size delta <10 KB or justified
- [ ] Telemetry / error tracking added for new code paths

---

## Common failure modes (recognize early)

- **"It works on my Mac"** — never tested on actual iOS Safari or Android Chrome
- **CSS on everything** — no design system, every component invents its own spacing
- **Tested only happy path** — no error/empty/loading states; first real user breaks it
- **Auth without authorization** — logged-in user A can `GET /api/user/B/data`
- **N+1 queries** — fast in dev with 10 rows, dies at 10k
- **No backups verified** — backup runs, restore never tested, real outage = data loss
- **No real-user monitoring** — perf "looks good" in Lighthouse, p99 is 8s in prod
- **Manual deploys** — fear of release → infrequent → bigger releases → more breakage
