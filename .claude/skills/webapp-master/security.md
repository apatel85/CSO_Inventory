# Security & Data Quality Module

Activate when: auth, security, RLS, OWASP, XSS, injection, permission, secret,
GDPR, data breach, sanitize, vulnerability, pentest, audit, compliance.

---

## Security First Principle

**Security is not a feature added at the end. It is enforced at every layer.**
Never rely on the client to enforce security. Assume all input is malicious.
Assume the network is compromised. Enforce at the DB level, not just the API.

---

## OWASP Top 10 — Checklist for Every App

### A01 — Broken Access Control
```
[x] Row-Level Security (RLS) enabled and FORCE-enabled on every data table
[x] Every API route checks authentication (not just some)
[x] IDOR prevented: route /api/items/:id verifies ownership in DB query, not just in UI
[x] Admin routes use role check middleware — not client-side conditional rendering
[x] Direct object references use UUIDs (not sequential IDs that can be enumerated)
[x] File access: signed URLs expire, never serve user files from public bucket
[x] E2E test: user A cannot read/write/delete user B's resources (tested)
```

### A02 — Cryptographic Failures
```
[x] HTTPS everywhere, HSTS header set
[x] Passwords hashed with bcrypt/Argon2 (managed auth does this — never store plaintext)
[x] JWT secrets ≥256 bits, rotated on breach
[x] Sensitive data (PII, payment) encrypted at rest (Supabase encrypts by default)
[x] Cookies: Secure; HttpOnly; SameSite=Lax (or Strict)
[x] No sensitive data in URL query strings (appears in logs, referrer headers)
[x] Backup files encrypted at rest and in transit
```

### A03 — Injection
```
[x] All DB queries use parameterized statements / ORM — never string concat
[x] User-provided filenames sanitized before use in file system paths
[x] Command execution: never shell out with user input
[x] GraphQL: query depth/complexity limits to prevent injection-style DoS
[x] Search inputs: use full-text search with parameterized queries, not LIKE '%$input%'
```

```ts
// ❌ NEVER — string interpolation in SQL
const items = await db.query(`SELECT * FROM items WHERE title = '${userInput}'`)

// ✓ Parameterized (Drizzle / Supabase / pg)
const items = await db.select().from(items).where(eq(items.title, userInput))
const { data } = await supabase.from('items').select().eq('title', userInput)
```

### A04 — Insecure Design
```
[x] Threat model documented for sensitive features (payment, data export, admin)
[x] Rate limiting on auth endpoints (login, signup, password reset)
[x] Account lockout after N failed logins (≥5)
[x] Password reset tokens: one-time use, expire in 1 hour
[x] MFA available for user accounts (mandatory for admin accounts)
[x] Principle of least privilege: API keys scoped to minimum permissions
```

### A05 — Security Misconfiguration
```
[x] No debug endpoints or stack traces in production responses
[x] All default credentials changed / removed
[x] Unused features disabled (e.g., Supabase dashboard auth for non-admin users)
[x] CORS: allowlist specific origins — never wildcard (*) in production
[x] HTTP security headers set (see below)
[x] Error messages: generic to users, detailed to logs — never expose stack traces
[x] Environment variables: .env not committed to git, .gitignore up to date
```

### A06 — Vulnerable and Outdated Components
```
[x] npm audit in CI — blocks on high/critical vulnerabilities
[x] Dependabot or Renovate configured for automated dependency PRs
[x] Remove unused dependencies (depcheck, knip)
[x] Lock file committed (package-lock.json / pnpm-lock.yaml)
[x] Node.js LTS version pinned in .nvmrc and package.json engines field
```

### A07 — Authentication Failures
```
[x] Auth managed by Supabase Auth / Clerk / Auth0 — never rolled manually
[x] Session tokens rotated on login and privilege change
[x] Logout invalidates session server-side (not just removes client cookie)
[x] "Remember me" uses separate long-lived token, not session token extension
[x] OAuth state parameter validated to prevent CSRF on OAuth callback
[x] No username enumeration on login (same message for wrong user / wrong password)
```

### A08 — Software and Data Integrity Failures
```
[x] Subresource Integrity (SRI) on any external CDN scripts
[x] Signed git commits on production branches (optional but recommended)
[x] File upload: validate MIME type server-side (magic bytes), not just extension
[x] Verify webhook signatures (Stripe, GitHub, etc.) before processing
[x] CI/CD pipeline: no untrusted actions without pinned commit SHA
```

### A09 — Security Logging and Monitoring Failures
```
[x] Auth events logged: login, logout, failed login, password reset
[x] Admin actions logged: who did what to which resource, when
[x] Rate limit violations logged and alerted
[x] Logs shipped to external system (not just local disk)
[x] Sensitive data (passwords, tokens, PII) never logged
[x] Alert on >5 failed logins per user per hour
[x] Alert on >100 failed logins from same IP per hour
```

### A10 — Server-Side Request Forgery (SSRF)
```
[x] Never fetch user-supplied URLs server-side without allowlist
[x] Outbound HTTP allowlisted to known external services
[x] Internal metadata endpoints (AWS 169.254.x.x) blocked in egress rules
```

---

## HTTP Security Headers

```ts
// next.config.ts — security headers
const securityHeaders = [
  { key: 'X-Content-Type-Options',    value: 'nosniff' },
  { key: 'X-Frame-Options',           value: 'DENY' },
  { key: 'X-XSS-Protection',          value: '1; mode=block' },
  { key: 'Referrer-Policy',           value: 'strict-origin-when-cross-origin' },
  { key: 'Permissions-Policy',
    value: 'camera=(), microphone=(), geolocation=(), interest-cohort=()' },
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },
  {
    key: 'Content-Security-Policy',
    value: [
      "default-src 'self'",
      "script-src 'self' 'nonce-{NONCE}'",       // use nonce for inline scripts
      "style-src 'self' 'unsafe-inline'",         // Tailwind requires unsafe-inline (or nonce)
      "img-src 'self' data: blob: https:",
      "font-src 'self'",
      "connect-src 'self' https://your-project.supabase.co wss://your-project.supabase.co",
      "frame-src 'none'",
      "object-src 'none'",
      "base-uri 'self'",
    ].join('; ')
  },
]
```

---

## XSS Prevention

```ts
// ❌ NEVER render raw user HTML directly
<div dangerouslySetInnerHTML={{ __html: userContent }} />

// ✓ Sanitize first with DOMPurify (client) or sanitize-html (server)
import DOMPurify from 'dompurify'
const clean = DOMPurify.sanitize(userContent, {
  ALLOWED_TAGS: ['b', 'i', 'em', 'strong', 'a', 'p', 'ul', 'ol', 'li'],
  ALLOWED_ATTR: ['href', 'target', 'rel'],
  FORCE_BODY: true,
})
<div dangerouslySetInnerHTML={{ __html: clean }} />

// For server-side rendering use sanitize-html:
import sanitizeHtml from 'sanitize-html'
const clean = sanitizeHtml(userContent, {
  allowedTags: sanitizeHtml.defaults.allowedTags.filter(t => t !== 'script'),
  allowedAttributes: { 'a': ['href', 'rel', 'target'] },
})

// ✓ For plain text — React escapes automatically (JSX)
<p>{userPlainText}</p>  // safe, React escapes HTML entities
```

---

## Secrets Management

```
Rules (absolute):
1. Secrets never in source code — ever
2. Secrets never in git history — use git-filter-repo if leaked
3. Secrets never in client-side code (NEXT_PUBLIC_ prefix only for non-secrets)
4. Secrets never in logs
5. Rotate immediately on any suspected exposure

Where secrets live:
  Development: .env.local (in .gitignore)
  CI/CD:       GitHub Actions Secrets / Vercel Environment Variables
  Production:  Vercel Env Vars / Doppler / AWS Secrets Manager / 1Password Connect

Secret types:
  NEXT_PUBLIC_SUPABASE_URL      — safe to be public (URL only)
  NEXT_PUBLIC_SUPABASE_ANON_KEY — safe to be public (anon key, RLS protects)
  SUPABASE_SERVICE_ROLE_KEY     — NEVER expose to client — bypasses RLS
  DATABASE_URL                  — server-side only
  JWT_SECRET                    — server-side only, ≥32 chars random
  STRIPE_SECRET_KEY             — server-side only
  SENTRY_DSN                    — can be public (NEXT_PUBLIC_)
```

```bash
# Scan for accidentally committed secrets
npx trufflesecurity/trufflehog git file://. --only-verified
git log --all -p | grep -iE '(password|secret|api_key|private_key)' | head -20
```

---

## Data Quality & Integrity Rules

### Validation layers (all three required)
```
1. Client (UX)  — immediate feedback, never trust for security
2. API (server) — authoritative validation, return structured errors
3. DB (schema)  — NOT NULL, CHECK constraints, FK integrity, UNIQUE
```

```ts
// Zod schema — shared between client and server
import { z } from 'zod'

export const CreateItemSchema = z.object({
  title:       z.string().min(1, 'Required').max(200, 'Too long'),
  description: z.string().max(5000).optional(),
  price:       z.number().positive().multipleOf(0.01).max(999999),
  category:    z.enum(['a', 'b', 'c']),
  tags:        z.array(z.string().max(50)).max(10).default([]),
})

// In server action / API route:
const result = CreateItemSchema.safeParse(req.body)
if (!result.success) {
  return { error: 'Validation failed', issues: result.error.flatten() }
}
```

```sql
-- DB constraints as last line of defense
ALTER TABLE items ADD CONSTRAINT chk_title_length CHECK (length(title) BETWEEN 1 AND 200);
ALTER TABLE items ADD CONSTRAINT chk_price_positive CHECK (price > 0);
ALTER TABLE items ADD CONSTRAINT chk_status CHECK (status IN ('draft', 'active', 'archived'));
-- Unique constraints
ALTER TABLE items ADD CONSTRAINT uq_items_slug_org UNIQUE (slug, org_id);
```

### Data consistency checklist
```
[x] Foreign keys defined for all relationships
[x] Cascade behavior explicit (CASCADE DELETE vs RESTRICT vs SET NULL)
[x] Enum values validated at application + DB level
[x] Money stored as INTEGER (cents) — never FLOAT
[x] Dates stored as TIMESTAMPTZ (UTC) — never naive datetime
[x] Phone numbers stored in E.164 format (+14155552671)
[x] Emails normalized (lowercased) before insert
[x] Text fields have max length constraints at DB level
[x] Soft delete pattern: deleted_at TIMESTAMPTZ — filter WHERE deleted_at IS NULL
```

---

## GDPR / Privacy Compliance

```
Lawful basis documented for every data collection point.

User rights (must implement if serving EU users):
  [x] Right to access: GET /api/me/export → JSON of all user data
  [x] Right to deletion: DELETE /api/me → anonymise or delete all user rows
  [x] Right to portability: export in machine-readable format (JSON/CSV)
  [x] Consent: explicit opt-in for marketing/non-essential cookies

Data minimization:
  [x] Collect only what's needed for the stated purpose
  [x] Retention policy: auto-delete inactive accounts after N months (document policy)
  [x] PII not in logs (mask email, phone in log output)
  [x] Analytics: Plausible / PostHog self-hosted (or Vercel Analytics) over Google Analytics
  [x] Cookie banner: strictly necessary cookies need no consent; others require opt-in

Third-party processors:
  [x] DPA (Data Processing Agreement) signed with Supabase, Sentry, Vercel, etc.
  [x] Subprocessor list documented and reviewed
```

---

## Continuous Security Check (after any security-relevant change)

1. Does the new endpoint verify authentication AND authorization?
2. Does the new DB query use parameterized inputs (no string concat)?
3. Does new user-generated HTML go through DOMPurify?
4. Does the new feature add new secrets? Are they in env vars only?
5. Does the new file upload validate MIME type server-side?
6. Does the new dependency pass `npm audit`?
7. Is there a test verifying unauthorized users cannot access the new data?
8. Are new error messages generic to users, detailed to logs?
