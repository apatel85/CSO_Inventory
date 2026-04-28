# Continuous Review & Best Practices Module

Activate when: review, audit, "is this good", improve, best practice, refactor,
code quality, before launch, PR review, technical debt, assess existing code.

---

## Review Philosophy

A good review finds problems the author couldn't see — not opportunities to rewrite.
Flag what matters: security, correctness, maintainability. Not style preference.
Be specific: "line 42 — this will throw if `user` is null" beats "handle errors."

---

## Full Review Protocol

Run this for any significant change or "is this production-ready?" question.

### Pass 1 — Security & Data Integrity (blocking)
```
[ ] No SQL string concatenation with user input (injection risk)
[ ] No user HTML rendered without DOMPurify sanitization
[ ] No secrets, API keys, or tokens in code (check git diff too)
[ ] All API routes verify authentication before processing
[ ] Authorization checked at DB level (RLS), not just UI/API
[ ] Multi-row mutations in transactions
[ ] User A access to User B data: impossible (tested)
[ ] File uploads: MIME type validated server-side
[ ] Error messages: generic to user, detailed to log — no stack traces to client
```

### Pass 2 — Correctness (blocking)
```
[ ] TypeScript: zero type errors (tsc --noEmit)
[ ] All async calls awaited or handled
[ ] All Promises have error handling (try/catch or .catch())
[ ] Edge cases handled: null/undefined, empty array, 0, negative numbers
[ ] Race conditions: concurrent mutations use locking or idempotency key
[ ] Memory leaks: useEffect cleanup functions where subscriptions opened
[ ] No infinite re-render loops (useEffect deps complete and stable)
```

### Pass 3 — User Experience (blocking)
```
[ ] Loading state visible for every async operation
[ ] Error state shown with actionable message on every failure
[ ] Empty state shown when list/query has no results
[ ] Form: validation errors shown inline, submit disabled during loading
[ ] Optimistic updates rollback correctly on failure
[ ] Navigation works with browser back/forward
[ ] Deep links resolve correctly (no blank page on direct URL)
```

### Pass 4 — Accessibility (blocking)
```
[ ] All interactive elements reachable by keyboard
[ ] Focus management correct (modal traps focus, returns on close)
[ ] Dynamic content changes announced via aria-live
[ ] All images have appropriate alt text
[ ] Color is not the only way status is conveyed
[ ] Contrast ≥4.5:1 for body text, ≥3:1 for large/UI elements
```

### Pass 5 — Performance (advisory → blocking if budget exceeded)
```
[ ] Bundle size delta <10 KB gzip (or documented reason)
[ ] No new N+1 queries (check DB query count in dev tools)
[ ] New DB columns indexed on filter/sort paths
[ ] Images use Next/Image with correct `sizes` attribute
[ ] Lighthouse ≥90 perf on affected pages
[ ] No unnecessary re-renders (React DevTools Profiler)
```

### Pass 6 — Code Quality (advisory)
```
[ ] No dead code (unused imports, variables, functions)
[ ] No magic numbers — use named constants
[ ] No deep nesting — extract named functions
[ ] Component / function has one clear responsibility
[ ] Tests cover the happy path + primary error case
[ ] No TODO comments without a linked issue
```

---

## Automated Review Commands

```bash
# Run before every PR review
npm run typecheck              # tsc --noEmit — zero errors required
npm run lint                   # ESLint — zero errors required
npm run test -- --run          # Vitest — all pass
npx playwright test            # E2E — all pass
npx next build                 # Build succeeds, check bundle size output
npx lighthouse http://localhost:3000 --output json | jq '.categories | {perf: .performance.score, a11y: .accessibility.score}'

# Security scan
npm audit --audit-level=high   # zero high/critical
npx trufflesecurity/trufflehog git file://. --only-verified  # no secrets found

# Bundle analysis
ANALYZE=true npx next build    # check for unexpected large chunks

# Accessibility
# In Playwright test: import { checkA11y } from 'axe-playwright'
# await checkA11y(page)  — zero violations
```

---

## Code Smell Catalog

Patterns that require fixing, not just commenting:

### Security smells
```ts
// ❌ String concat in query
db.query(`SELECT * FROM users WHERE email = '${email}'`)
// ✓ Parameterized
db.select().from(users).where(eq(users.email, email))

// ❌ Direct HTML from user
<div dangerouslySetInnerHTML={{ __html: comment }} />
// ✓ Sanitized
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(comment) }} />

// ❌ Service role key in client component
const supabase = createClient(url, process.env.SUPABASE_SERVICE_ROLE_KEY)
// ✓ Service role only in server-side, anon key in client
```

### Data integrity smells
```ts
// ❌ Multiple writes without transaction
await db.update(orders).set({ status: 'paid' }).where(eq(orders.id, id))
await db.update(inventory).set({ reserved: sql`reserved - 1` }).where(eq(inventory.id, itemId))
// ✓ Transaction
await db.transaction(async tx => {
  await tx.update(orders).set({ status: 'paid' }).where(eq(orders.id, id))
  await tx.update(inventory).set({ reserved: sql`reserved - 1` }).where(eq(inventory.id, itemId))
})

// ❌ Mutation without idempotency check
async function chargeCard(paymentMethodId: string, amount: number) {
  return stripe.charges.create({ amount, source: paymentMethodId })
}
// ✓ With idempotency key
async function chargeCard(paymentMethodId: string, amount: number, idempotencyKey: string) {
  return stripe.charges.create({ amount, source: paymentMethodId }, { idempotencyKey })
}
```

### Performance smells
```ts
// ❌ N+1 query
const posts = await getPosts()
const postsWithAuthors = await Promise.all(posts.map(p => getUser(p.userId)))  // N queries!
// ✓ JOIN
const postsWithAuthors = await db.select({ post: posts, author: users })
  .from(posts).leftJoin(users, eq(posts.userId, users.id))

// ❌ Select all columns
const items = await supabase.from('items').select('*')
// ✓ Select needed columns
const items = await supabase.from('items').select('id, title, created_at')

// ❌ No pagination
const allItems = await db.select().from(items)  // will die at 100k rows
// ✓ Paginated
const page = await db.select().from(items)
  .where(lt(items.createdAt, cursor)).orderBy(desc(items.createdAt)).limit(20)
```

### React / Next.js smells
```ts
// ❌ Server data in client state
'use client'
const [user, setUser] = useState(null)
useEffect(() => { fetch('/api/me').then(r => r.json()).then(setUser) }, [])
// ✓ Server component or TanStack Query
const user = await getUser()  // server component

// ❌ useEffect for derived state
const [doubled, setDoubled] = useState(0)
useEffect(() => { setDoubled(count * 2) }, [count])
// ✓ useMemo
const doubled = useMemo(() => count * 2, [count])

// ❌ Missing cleanup
useEffect(() => {
  const sub = supabase.channel('x').subscribe()
  // missing return () => supabase.removeChannel(sub)
}, [])
// ✓ With cleanup
useEffect(() => {
  const sub = supabase.channel('x').subscribe()
  return () => supabase.removeChannel(sub)
}, [])
```

---

## Pre-Launch Checklist (run before any public release)

```
FUNCTIONAL
[ ] All E2E tests pass on Chromium + WebKit + Firefox + mobile
[ ] Manual exploratory test on real iPhone + real Android device
[ ] All auth flows work: signup, login, logout, password reset, OAuth
[ ] Critical user journey works end-to-end with production data
[ ] Error monitoring verified: trigger an error, confirm it appears in Sentry

SECURITY
[ ] npm audit: zero high/critical vulnerabilities
[ ] No secrets in codebase or git history
[ ] Security headers verified (securityheaders.com)
[ ] OWASP checklist completed (see security.md)
[ ] RLS tested: cross-user data access blocked
[ ] Rate limiting active on auth endpoints

PERFORMANCE
[ ] Lighthouse ≥90 perf, ≥95 a11y on all main pages
[ ] Core Web Vitals passing in CI (Lighthouse CI)
[ ] Real-user monitoring active (Speed Insights / Sentry)

OPERATIONS
[ ] Error tracking active (Sentry DSN configured for prod)
[ ] Uptime monitoring configured (external health check every 1 min)
[ ] Backup verified: take pg_dump + confirm it restores
[ ] Rollback plan: how to revert if deploy breaks production
[ ] On-call: who gets paged if prod goes down?
[ ] Support email / feedback mechanism accessible from app

LEGAL / COMPLIANCE
[ ] Privacy policy linked from footer (required for any data collection)
[ ] Terms of service linked from signup
[ ] Cookie consent if using non-essential analytics (EU)
[ ] GDPR: data export + deletion available if EU users
```

---

## Technical Debt Triage

When reviewing existing code for debt, classify each item:

| Class | Definition | Action |
|-------|------------|--------|
| **Critical** | Security vulnerability, data loss risk, production outage risk | Fix immediately, before next deploy |
| **High** | Performance degradation, accessibility failure, RLS gap | Fix in current sprint |
| **Medium** | Code smell, missing test, outdated dependency | Schedule in next 2 sprints |
| **Low** | Style inconsistency, minor cleanup, nice-to-have | Backlog, fix when in the area |

Never let Critical or High items sit. They compound.

---

## Continuous Improvement Loop

After every release, spend 30 minutes on:

1. **Sentry**: What were the top 5 errors this week? Fix the top 2.
2. **RUM**: What routes have p95 >1s? Profile and fix the worst one.
3. **User feedback**: Any reports of confusion, broken flows, or lost data?
4. **Dependencies**: Any new `npm audit` high/critical advisories?
5. **Test coverage**: Did the last production bug have a test? Add it.

Document findings in a brief retro note. Trends matter more than one-off spikes.
