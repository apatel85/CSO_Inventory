# Cloud Architecture & Multi-User Data Module

Activate when: deploy, cloud, database, schema, migration, real-time, scale,
concurrent users, backup, infrastructure, serverless, multi-tenant, data integrity.

---

## Architecture Stack (default — override only with documented reason)

```
Users
  ↓ HTTPS (Vercel Edge Network / Cloudflare)
Next.js App (Vercel Serverless + Edge Runtime)
  ↓ Connection Pooling (Supabase Pooler / PgBouncer)
Postgres (Supabase / Neon) — RLS enforced at DB
  ├─ Supabase Realtime (WebSocket → broadcast / presence / DB changes)
  ├─ Supabase Storage (files → R2/S3 behind CDN)
  └─ Supabase Auth (JWT issued to users)
Background Jobs: Inngest / Trigger.dev (async, retryable, idempotent)
Cache: Upstash Redis (session store, rate limit, short-lived cache)
Email: Resend / Postmark (transactional) + SendGrid (bulk)
CDN: Cloudflare R2 or AWS S3 + CloudFront for assets
Monitoring: Sentry (errors) + Vercel Analytics (RUM) + Uptime Robot
```

---

## Database Design Principles

### Schema conventions
```sql
-- Every table
CREATE TABLE items (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  created_at  TIMESTAMPTZ DEFAULT now() NOT NULL,
  updated_at  TIMESTAMPTZ DEFAULT now() NOT NULL,  -- trigger keeps in sync
  created_by  UUID REFERENCES auth.users(id) NOT NULL,
  org_id      UUID REFERENCES orgs(id) NOT NULL    -- multi-tenant isolation key
);

-- Auto-update trigger
CREATE OR REPLACE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$ BEGIN
  NEW.updated_at = now(); RETURN NEW;
END; $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_items_updated_at
  BEFORE UPDATE ON items
  FOR EACH ROW EXECUTE FUNCTION set_updated_at();

-- Indexes — always on FK columns + filter/sort columns
CREATE INDEX CONCURRENTLY idx_items_org_id     ON items(org_id);
CREATE INDEX CONCURRENTLY idx_items_created_by ON items(created_by);
CREATE INDEX CONCURRENTLY idx_items_created_at ON items(created_at DESC);
-- Composite for common queries:
CREATE INDEX CONCURRENTLY idx_items_org_created ON items(org_id, created_at DESC);
```

### Multi-tenant isolation patterns
```
Pattern A — Shared schema, org_id column (recommended for most apps)
  + Simple to manage, one DB, easy cross-tenant analytics
  + RLS handles isolation automatically
  - Must be careful never to leak org_id filter

Pattern B — Schema per tenant (Postgres schemas)
  + Hard isolation, easy per-tenant backup/migrate
  - Complex migrations (N schemas), hard analytics

Pattern C — DB per tenant
  + Maximum isolation
  - Expensive, very complex operations
  → Only for regulated data (HIPAA, FedRAMP)

Default: Pattern A with strict RLS.
```

---

## Row-Level Security (RLS) — The Core Multi-User Guarantee

```sql
-- Enable on every table that holds user/org data
ALTER TABLE items ENABLE ROW LEVEL SECURITY;
ALTER TABLE items FORCE ROW LEVEL SECURITY;  -- enforce even for table owner

-- Standard patterns

-- 1. User owns their own rows
CREATE POLICY "users_own_rows" ON items
  FOR ALL USING (auth.uid() = created_by);

-- 2. Org members access org rows (multi-tenant)
CREATE POLICY "org_members_access" ON items
  FOR ALL USING (
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid()
    )
  );

-- 3. Org admins can delete, members can only read/write
CREATE POLICY "org_read_write" ON items
  FOR SELECT USING (org_id IN (SELECT org_id FROM org_members WHERE user_id = auth.uid()));

CREATE POLICY "org_delete_admin_only" ON items
  FOR DELETE USING (
    org_id IN (
      SELECT org_id FROM org_members
      WHERE user_id = auth.uid() AND role = 'admin'
    )
  );

-- 4. Service role bypasses RLS — only use in server-side admin contexts, never in client
-- In Supabase: use service_role key server-side, anon/user key client-side
```

```ts
// RLS test — required for every protected table
test('RLS: user A cannot read user B items', async () => {
  const { data: userAItems } = await supabaseAs(userB).from('items')
    .select('id').eq('created_by', userA.id)
  expect(userAItems).toHaveLength(0)  // RLS filtered them out
})

test('RLS: org member can read org items', async () => {
  const { data } = await supabaseAs(orgMember).from('items')
    .select('id').eq('org_id', orgId)
  expect(data!.length).toBeGreaterThan(0)
})

test('RLS: non-member cannot read org items', async () => {
  const { data } = await supabaseAs(outsider).from('items')
    .select('id').eq('org_id', orgId)
  expect(data).toHaveLength(0)
})
```

---

## Concurrency & Data Integrity

### Optimistic locking (prevent lost updates)
```sql
-- Add version column
ALTER TABLE items ADD COLUMN version INTEGER NOT NULL DEFAULT 1;

-- On every UPDATE, check version and increment
UPDATE items
SET title = $1, version = version + 1, updated_at = now()
WHERE id = $2 AND version = $3;  -- $3 = version client read

-- If rows_affected = 0 → someone else updated first → return 409 Conflict
```

```ts
// Server action / API route
async function updateItem(id: string, title: string, version: number) {
  const result = await db
    .update(items)
    .set({ title, version: sql`version + 1` })
    .where(and(eq(items.id, id), eq(items.version, version)))
  if (result.rowCount === 0) throw new ConflictError('Item was modified by another user')
}
```

### Transactions — wrap all multi-table mutations
```ts
// Drizzle ORM transaction
await db.transaction(async (tx) => {
  const order = await tx.insert(orders).values({ userId, total }).returning()
  await tx.insert(orderItems).values(items.map(i => ({ orderId: order[0].id, ...i })))
  await tx.update(inventory).set({ stock: sql`stock - ${qty}` }).where(eq(inventory.id, itemId))
  // All succeed or all roll back
})

// Supabase RPC (Postgres function) for complex atomic operations
const { data, error } = await supabase.rpc('create_order_with_items', {
  p_user_id: userId,
  p_items: items
})
```

### Idempotency keys (for retryable operations)
```ts
// Client generates key, server uses it to detect retries
const idempotencyKey = crypto.randomUUID()  // stored in localStorage until confirmed

// Server: check if already processed
const existing = await db.query.idempotencyKeys.findFirst({
  where: eq(idempotencyKeys.key, key)
})
if (existing) return existing.result  // return cached result, don't re-run

// Process + store result atomically
await db.transaction(async (tx) => {
  const result = await processPayment(tx, data)
  await tx.insert(idempotencyKeys).values({ key, result: JSON.stringify(result), expiresAt: addHours(new Date(), 24) })
  return result
})
```

---

## Real-Time & Live Collaboration

```ts
// Supabase Realtime — subscribe to DB changes
const channel = supabase
  .channel('items-changes')
  .on('postgres_changes', {
    event: '*',
    schema: 'public',
    table: 'items',
    filter: `org_id=eq.${orgId}`  // filter to org — RLS also applies
  }, (payload) => {
    queryClient.invalidateQueries({ queryKey: ['items', orgId] })
  })
  .subscribe()

// Cleanup
return () => supabase.removeChannel(channel)

// Presence (who is online)
channel.track({ userId, name, cursor })
channel.on('presence', { event: 'sync' }, () => {
  const state = channel.presenceState()
  setOnlineUsers(Object.values(state).flat())
})
```

```
For advanced collaboration (Google Docs-style):
→ Use Liveblocks (managed CRDT) or Yjs + y-supabase provider
→ Never build CRDT yourself
```

---

## Database Migrations

```bash
# Supabase migrations
supabase migration new add_items_version_column
# Edit supabase/migrations/<timestamp>_add_items_version_column.sql
supabase db push                    # local
supabase db push --linked           # remote (production)
```

```sql
-- Migration best practices
-- ✓ Always reversible (write DOWN migration)
-- ✓ Concurrent index creation (no table lock)
-- ✓ Add nullable column first, backfill, then add NOT NULL constraint
-- ✓ Test on a copy of production data before running on prod

-- Safe nullable → NOT NULL migration:
-- Step 1: Add nullable
ALTER TABLE items ADD COLUMN score INTEGER;
-- Step 2: Backfill
UPDATE items SET score = 0 WHERE score IS NULL;
-- Step 3: Add constraint (can be separate deployment)
ALTER TABLE items ALTER COLUMN score SET NOT NULL;
ALTER TABLE items ALTER COLUMN score SET DEFAULT 0;
```

---

## Backups & Recovery

```
Supabase:
  - Automatic daily backups (Pro plan)
  - Point-in-time recovery to 7 days (Pro) / 30 days (Team+)
  - Manual: pg_dump + store in S3 with versioning

Recovery drill (run quarterly):
  1. Take pg_dump of production
  2. Restore to staging DB
  3. Run full E2E test suite against restored DB
  4. Document recovery time (target: <1 hour for data, <4 hours for full restore)

Retention policy:
  - Production: 30 days PITR + monthly snapshots to cold storage
  - Staging: 7 days
  - Dev: on-demand only
```

---

## Observability Setup (configure before first deployment)

```ts
// sentry.config.ts — errors + performance
import * as Sentry from '@sentry/nextjs'
Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,           // 10% for performance tracing (adjust based on volume)
  profilesSampleRate: 0.1,
  integrations: [Sentry.replayIntegration({ maskAllText: true })],
  replaysSessionSampleRate: 0.05,  // 5% of sessions
  replaysOnErrorSampleRate: 1.0,   // 100% of sessions with errors
})
```

```ts
// Structured logging (Pino) — one request ID per trace
import pino from 'pino'
export const logger = pino({ level: process.env.LOG_LEVEL ?? 'info' })

// In every API route/server action:
const log = logger.child({ requestId: crypto.randomUUID(), userId: session.user.id })
log.info({ action: 'create_item', itemId }, 'Item created')
log.error({ err }, 'Failed to create item')
```

```
Alerting rules (set in Sentry / PagerDuty):
  - Error rate >1% on any route → alert immediately
  - p95 latency >2× baseline over 5 min → alert
  - DB connection pool saturation >80% → alert
  - Failed job queue >10 → alert
  - Uptime check failure → alert immediately
```

---

## Scaling Playbook

| Scale trigger | Action |
|--------------|--------|
| DB CPU >70% sustained | Add read replica; route SELECT to replica |
| DB connections maxing out | Use PgBouncer pooler (already in Supabase) |
| API p95 >500ms | Profile — likely N+1, missing index, or missing cache |
| Storage >500 GB | Enable tiered storage (Supabase), archive old files to S3 Glacier |
| >10k concurrent WebSocket connections | Use Supabase Realtime managed or Ably for large scale |
| Background job queue depth >1000 | Add more workers (Inngest auto-scales) |
| Static assets >10 GB/month CDN cost | Move to R2 (free egress) |

### Rate limiting (set from day 1)
```ts
// Upstash Ratelimit
import { Ratelimit } from '@upstash/ratelimit'
import { Redis } from '@upstash/redis'

const ratelimit = new Ratelimit({
  redis: Redis.fromEnv(),
  limiter: Ratelimit.slidingWindow(10, '10 s'),  // 10 req per 10s per identifier
})

// In API route or middleware:
const { success, limit, remaining } = await ratelimit.limit(req.ip ?? 'anonymous')
if (!success) return new Response('Too Many Requests', { status: 429,
  headers: { 'Retry-After': '10', 'X-RateLimit-Limit': limit.toString() }
})
```

---

## Continuous Cloud/Data Check (after any data-touching change)

1. Is every new table protected by RLS? Is `FORCE ROW LEVEL SECURITY` set?
2. Are new multi-row mutations wrapped in a transaction?
3. Are new DB columns indexed where they'll be queried/sorted?
4. Are file uploads going through signed URLs (not public buckets)?
5. Is the migration reversible? Tested on a data copy?
6. Are background jobs idempotent (safe to retry)?
7. Are new secrets in env variables, not hardcoded?
8. Is the new code path covered by Sentry error tracking?
