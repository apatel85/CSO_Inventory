# Testing & QA Specialist Module

Activate when: test, spec, bug, QA, Playwright, Cypress, E2E, unit, integration,
coverage, debug, broken, regression, before launch, verify work.

---

## Testing Philosophy

- Tests document behaviour, not just catch bugs. Write tests that would have caught the last prod issue.
- Test the user journey, not the implementation. If you refactor and tests break, the tests were wrong.
- The test suite must run in CI on every PR. If it's flaky, fix or delete it — flaky tests are noise.
- Coverage % is a vanity metric. 80% on business logic beats 98% on getters/setters.

---

## Testing Pyramid

| Layer | Tool | What | Target |
|-------|------|------|--------|
| **Unit** | Vitest | Pure functions, hooks, business logic | >80% on logic files |
| **Component** | Testing Library + Vitest | Render, interact, assert DOM | Every interactive component |
| **Integration** | Vitest + MSW | API calls + state flows end-to-end in-process | Every critical service |
| **E2E** | Playwright | Full browser, real network, critical journeys | All auth + core flows |
| **Visual regression** | Playwright snapshots / Chromatic | CSS drift, layout breaks | Key pages + components |
| **Accessibility** | axe-playwright | Automated a11y violations | Every page route |
| **Performance** | Lighthouse CI | Core Web Vitals budgets | Every PR |
| **Manual / exploratory** | Real device + BrowserStack | Edge cases, feel, UX | Every release |

---

## Unit Test Patterns

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
export default defineConfig({
  test: {
    environment: 'jsdom',
    setupFiles: ['./tests/setup.ts'],
    coverage: { provider: 'v8', thresholds: { lines: 80, functions: 80 } }
  }
})

// tests/setup.ts
import '@testing-library/jest-dom'
import { server } from './mocks/server'
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }))
afterEach(() => server.resetHandlers())
afterAll(() => server.close())
```

```ts
// Unit test pattern — pure business logic
describe('calculateDiscount', () => {
  it('applies 10% for orders over $100', () => {
    expect(calculateDiscount(150, 'SAVE10')).toBe(15)
  })
  it('returns 0 for invalid code', () => {
    expect(calculateDiscount(150, 'INVALID')).toBe(0)
  })
  it('never returns negative discount', () => {
    expect(calculateDiscount(5, 'SAVE10')).toBeGreaterThanOrEqual(0)
  })
})
```

---

## Component Test Patterns

```tsx
// Component test — test behaviour, not internals
import { render, screen, userEvent } from '@testing-library/react'

describe('LoginForm', () => {
  it('shows error when submitting empty form', async () => {
    render(<LoginForm onSuccess={vi.fn()} />)
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }))
    expect(screen.getByRole('alert')).toHaveTextContent(/email is required/i)
  })

  it('calls onSuccess with token on valid credentials', async () => {
    const onSuccess = vi.fn()
    server.use(rest.post('/api/auth', (req, res, ctx) =>
      res(ctx.json({ token: 'abc123' }))))
    render(<LoginForm onSuccess={onSuccess} />)
    await userEvent.type(screen.getByLabelText(/email/i), 'user@example.com')
    await userEvent.type(screen.getByLabelText(/password/i), 'secret')
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }))
    await waitFor(() => expect(onSuccess).toHaveBeenCalledWith('abc123'))
  })

  it('disables submit while loading', async () => {
    server.use(rest.post('/api/auth', (req, res, ctx) =>
      res(ctx.delay(500), ctx.json({ token: 'x' }))))
    render(<LoginForm onSuccess={vi.fn()} />)
    // ... fill form
    await userEvent.click(screen.getByRole('button', { name: /sign in/i }))
    expect(screen.getByRole('button', { name: /sign in/i })).toBeDisabled()
  })
})
```

---

## E2E Test Patterns (Playwright)

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test'
export default defineConfig({
  testDir: './e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  reporter: [['html'], ['github']],
  use: { baseURL: 'http://localhost:3000', trace: 'on-first-retry' },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox',  use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit',   use: { ...devices['Desktop Safari'] } },
    { name: 'mobile-ios',     use: { ...devices['iPhone 15'] } },
    { name: 'mobile-android', use: { ...devices['Pixel 7'] } },
  ],
  webServer: { command: 'npm run dev', url: 'http://localhost:3000', reuseExistingServer: !process.env.CI },
})
```

```ts
// e2e/auth.spec.ts — authentication critical journey
test.describe('Authentication', () => {
  test('user can sign up, verify email, and log in', async ({ page }) => {
    await page.goto('/signup')
    await page.getByLabel('Email').fill('test@example.com')
    await page.getByLabel('Password').fill('SecurePass123!')
    await page.getByRole('button', { name: 'Create account' }).click()
    await expect(page.getByText('Check your email')).toBeVisible()
    // ... verify email link, login, assert dashboard
  })

  test('expired session redirects to login', async ({ page, context }) => {
    // Set up authenticated state
    await context.addCookies([{ name: 'session', value: 'expired-token', domain: 'localhost' }])
    await page.goto('/dashboard')
    await expect(page).toHaveURL('/login')
    await expect(page.getByText(/session expired/i)).toBeVisible()
  })
})

// e2e/rls.spec.ts — authorization boundary test
test('user A cannot access user B data', async ({ browser }) => {
  const ctxA = await browser.newContext()
  const ctxB = await browser.newContext()
  // ... login as userA, create resource, get resource ID
  // ... login as userB, attempt GET /api/resource/{userA_resource_id}
  const res = await ctxB.request.get(`/api/resources/${userAResourceId}`)
  expect(res.status()).toBe(403)  // or 404 — never 200
})
```

---

## MSW (API Mocking for unit/integration)

```ts
// tests/mocks/handlers.ts
import { http, HttpResponse } from 'msw'

export const handlers = [
  http.get('/api/user', () => HttpResponse.json({ id: '1', name: 'Test User' })),
  http.post('/api/items', async ({ request }) => {
    const body = await request.json()
    return HttpResponse.json({ id: crypto.randomUUID(), ...body }, { status: 201 })
  }),
  // Error scenario
  http.get('/api/broken', () => HttpResponse.error()),
]

// Override in specific tests:
server.use(http.get('/api/user', () => new HttpResponse(null, { status: 401 })))
```

---

## Must-Test Scenarios (cover all before launch)

```
AUTH
[ ] Sign up (email + OAuth)
[ ] Login + logout
[ ] Session expiry → redirect to login, return to intended page after re-auth
[ ] Password reset (full flow including email link)
[ ] Account locked after N failed attempts
[ ] OAuth callback with invalid state token → rejected

AUTHORIZATION
[ ] User A GET/PUT/DELETE of user B's resource → 403 or 404 (never 200)
[ ] Admin actions blocked for regular users
[ ] Unauthenticated requests blocked on all protected routes
[ ] Token from expired session rejected

DATA & FORMS
[ ] Submit empty required fields → validation errors shown
[ ] Submit malformed data (XSS payload, SQL string, emoji, 10k chars) → rejected gracefully
[ ] Duplicate submit (double-click) → processed once, not twice
[ ] Network failure mid-submit → user informed, can retry, no duplicate
[ ] Browser back after submit → no duplicate submission
[ ] File upload: oversized file, wrong mime type, network drop mid-upload

STATES
[ ] Loading state visible during all async operations
[ ] Error state shown with actionable message on all failures
[ ] Empty state shown when no data
[ ] Offline: detect, show banner, queue mutations, sync on reconnect

SCALE
[ ] 0 items in a list
[ ] 1 item
[ ] 1000 items (pagination/virtualization works)
[ ] Search with 0 results vs many results

CONCURRENCY
[ ] Two browser tabs editing same record → last write wins or conflict UI shown
[ ] Rapid clicks on submit → idempotent (processed once)

ACCESSIBILITY
[ ] Tab through entire page in logical order
[ ] Screen reader announces dynamic content changes
[ ] All forms operable without mouse
[ ] All errors announced via aria-live

CROSS-DEVICE (manual)
[ ] iOS Safari + Android Chrome (real devices)
[ ] Works at 320px width without horizontal scroll
[ ] Keyboard doesn't cover active input on mobile
```

---

## CI Configuration (GitHub Actions)

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci
      - run: npm run typecheck       # tsc --noEmit
      - run: npm run lint            # eslint + prettier --check
      - run: npm run test            # vitest run --coverage
      - run: npx size-limit          # bundle size gate

  e2e:
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '22', cache: 'npm' }
      - run: npm ci && npx playwright install --with-deps
      - run: npm run build
      - run: npx playwright test
      - uses: actions/upload-artifact@v4
        if: failure()
        with: { name: playwright-report, path: playwright-report/ }

  lighthouse:
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build
      - uses: treosh/lighthouse-ci-action@v12
        with:
          urls: |
            http://localhost:3000
            http://localhost:3000/dashboard
          budgetPath: .lighthouserc.json
          uploadArtifacts: true
```

```json
// .lighthouserc.json
{
  "ci": {
    "assert": {
      "assertions": {
        "categories:performance":        ["error", { "minScore": 0.9 }],
        "categories:accessibility":      ["error", { "minScore": 0.95 }],
        "categories:best-practices":     ["error", { "minScore": 0.95 }],
        "categories:seo":                ["error", { "minScore": 0.9 }],
        "first-contentful-paint":        ["error", { "maxNumericValue": 2000 }],
        "largest-contentful-paint":      ["error", { "maxNumericValue": 2500 }],
        "cumulative-layout-shift":       ["error", { "maxNumericValue": 0.1 }],
        "total-blocking-time":           ["error", { "maxNumericValue": 300 }]
      }
    }
  }
}
```

---

## Debugging Protocol (when something is broken)

```
1. REPRODUCE — minimal reproducible case. Exact steps. Exact error message.
2. ISOLATE   — binary search: which component, which function, which data state?
3. EVIDENCE  — read the stack trace. Check network tab. Check console. Check DB query log.
4. HYPOTHESIZE — one root cause. If unsure, list 3 candidates, test the most likely first.
5. FIX       — surgical change. No refactoring while debugging.
6. VERIFY    — does the original reproduction case pass? No regression on related tests?
7. PREVENT   — add a test that would have caught this. Fix the failure mode, not just the symptom.
```

### Common root causes by symptom

| Symptom | Check first |
|---------|-------------|
| "Works locally, fails in prod" | Env vars missing, different Node version, SSR vs CSR |
| "Works once, breaks on refresh" | State not persisted (localStorage, cookie) |
| "Only fails for some users" | Auth state, locale/timezone, RLS policy, race condition |
| "Slow on mobile" | N+1 queries, no pagination, unoptimized images, no code splitting |
| "Layout breaks on iOS" | 100vh bug, safe area, tap delay, form zoom (font-size<16px) |
| "Form submits twice" | No submit button disable, no idempotency key |
| "Data loss on concurrent edit" | Missing optimistic lock or conflict detection |
