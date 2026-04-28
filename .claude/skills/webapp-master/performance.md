# Performance Engineering Module

Activate when: slow, perf, Lighthouse, bundle size, LCP, CLS, INP, TTFB,
optimize, load time, speed, cache, Core Web Vitals, lagging, jank.

---

## Performance Budgets (enforce in CI — failing these blocks merge)

| Metric | Target | Hard fail |
|--------|--------|-----------|
| LCP (Largest Contentful Paint) | <2.5 s | >4 s |
| INP (Interaction to Next Paint) | <200 ms | >500 ms |
| CLS (Cumulative Layout Shift) | <0.1 | >0.25 |
| TTFB (Time to First Byte) | <800 ms | >1.8 s |
| FCP (First Contentful Paint) | <1.8 s | >3 s |
| JS initial bundle (gzip) | <170 KB | >300 KB |
| Total page weight | <1.5 MB | >3 MB |
| Image weight (above fold) | <500 KB | >1 MB |
| Lighthouse Performance score | ≥90 | <80 |

---

## Measurement First — Never Optimize Blind

```bash
# Local measurement
npx lighthouse http://localhost:3000 --view
npx @lhci/cli autorun                            # CI-ready Lighthouse
npx webpack-bundle-analyzer stats.json           # Next.js: next build --analyze
npx @next/bundle-analyzer                        # Next.js built-in

# Real-user measurement (set up day 1)
# Vercel Speed Insights — zero-config for Vercel deploys
# Sentry Performance — p50/p95/p99 per route
# Web Vitals API (self-hosted):
```

```ts
// lib/vitals.ts — report real-user Core Web Vitals to analytics
import { onCLS, onINP, onLCP, onFCP, onTTFB } from 'web-vitals'

function sendToAnalytics({ name, value, rating }: Metric) {
  // Send to your analytics endpoint
  fetch('/api/vitals', { method: 'POST', body: JSON.stringify({ name, value, rating }) })
}

onCLS(sendToAnalytics)
onINP(sendToAnalytics)
onLCP(sendToAnalytics)
onFCP(sendToAnalytics)
onTTFB(sendToAnalytics)
```

---

## Network Optimization

```
HTTP/2 or HTTP/3 (Vercel/Cloudflare enable by default)
Brotli compression (10–15% smaller than gzip — enable at CDN level)
CDN for all static assets (Vercel, Cloudflare, CloudFront)
Edge caching for SSR HTML (Cache-Control: s-maxage=60, stale-while-revalidate=300)
Preconnect/DNS prefetch for critical third parties:
```

```html
<!-- In <head> — only for origins you'll definitely use -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="dns-prefetch" href="https://api.example.com">
<link rel="preload" as="font" href="/fonts/inter.woff2" crossorigin>
<link rel="preload" as="image" href="/hero.avif">   <!-- LCP image only -->
```

```
Cache-Control strategy:
  Hashed assets (JS/CSS with content hash): Cache-Control: public, max-age=31536000, immutable
  HTML (index.html, /about):               Cache-Control: public, max-age=0, must-revalidate
  API responses (read-heavy):              Cache-Control: private, max-age=60, stale-while-revalidate=300
  User-specific data:                      Cache-Control: private, no-store (never CDN-cache)
```

---

## JavaScript Optimization

### Code splitting (Next.js)
```ts
// Route-level: automatic in Next.js App Router
// Component-level: dynamic import for heavy components
import dynamic from 'next/dynamic'

const HeavyChart   = dynamic(() => import('./Chart'), { loading: () => <ChartSkeleton /> })
const VideoPlayer  = dynamic(() => import('./VideoPlayer'), { ssr: false })
const AdminPanel   = dynamic(() => import('./AdminPanel'))  // only loads if user visits route
```

### Bundle analysis and trimming
```bash
# Find what's large
npx next build && npx @next/bundle-analyzer

# Common offenders and fixes:
moment.js (67 KB)       → replace with date-fns (tree-shakeable) or Temporal API
lodash (70 KB)          → import { debounce } from 'lodash-es'  or use native
faker (>1 MB in test)   → keep out of production bundle (devDependency)
all of an icon library  → import { IconName } from '@heroicons/react/24/solid'  (not index)

# Check for duplicates
npx duplicate-package-checker-webpack-plugin
```

### Runtime performance
```ts
// Expensive computations — memoize with useMemo
const sortedItems = useMemo(() =>
  items.toSorted((a, b) => b.createdAt - a.createdAt),
  [items])

// Stable callbacks — useCallback to prevent child rerenders
const handleSubmit = useCallback(async (data: FormData) => {
  await createItem(data)
}, [createItem])

// Large lists — virtualize, never render all rows
import { useVirtualizer } from '@tanstack/react-virtual'

// Debounce search inputs
const debouncedSearch = useDebouncedCallback((value: string) => {
  setQuery(value)
}, 300)

// Expensive images — blur placeholder prevents CLS
<Image src="..." blurDataURL="..." placeholder="blur" />
```

---

## Render Optimization (Next.js App Router)

```ts
// Server components (default) — no JS sent to client, fast TTFB
// Use for: data fetching, layout, non-interactive content
async function ProductList() {
  const products = await db.products.findMany()  // runs on server
  return <ul>{products.map(p => <ProductCard key={p.id} product={p} />)}</ul>
}

// Client components — only when needed (interactivity, browser APIs)
'use client'
function SearchInput() {
  const [query, setQuery] = useState('')
  return <input value={query} onChange={e => setQuery(e.target.value)} />
}

// Streaming with Suspense — fast initial HTML, fill in incrementally
export default function Page() {
  return (
    <>
      <StaticHeader />           {/* renders immediately */}
      <Suspense fallback={<DashboardSkeleton />}>
        <Dashboard />            {/* streams in when data ready */}
      </Suspense>
    </>
  )
}

// Parallel data fetching (not waterfall)
const [user, posts, comments] = await Promise.all([
  getUser(id), getPosts(id), getComments(id)
])
```

---

## Image Optimization

```tsx
// Next.js Image component — handles srcset, AVIF/WebP, lazy load, aspect ratio
import Image from 'next/image'

// Above-fold LCP image
<Image
  src="/hero.jpg"
  alt="Hero image"
  width={1200}
  height={600}
  priority               // preloads — only for LCP image
  sizes="(max-width: 768px) 100vw, 50vw"
/>

// Below-fold
<Image
  src="/feature.jpg"
  alt="Feature"
  width={600}
  height={400}
  loading="lazy"        // default in Next/Image
  sizes="(max-width: 768px) 100vw, 33vw"
/>
```

```
Image rules:
- Never serve a 1200px image to a 375px screen
- Always specify width+height (prevents CLS)
- AVIF first, WebP fallback, JPEG last
- Hero image: serve ≤100 KB on mobile (AVIF compressed)
- User avatars: serve at 2× display size, round to nearest power of 2
- Icons/logos: SVG (no raster at all)
- Alt text: describes content, not filename
```

---

## Database / API Performance

```sql
-- Check slow queries (Supabase / Postgres)
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 20;

-- Index columns used in WHERE, ORDER BY, JOIN ON
CREATE INDEX CONCURRENTLY idx_posts_user_id ON posts(user_id);
CREATE INDEX CONCURRENTLY idx_posts_created_at ON posts(created_at DESC);
-- Composite index for common filter + sort
CREATE INDEX CONCURRENTLY idx_posts_user_created ON posts(user_id, created_at DESC);

-- Avoid SELECT * — select only needed columns
SELECT id, title, created_at FROM posts WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20;

-- Cursor pagination (more efficient than OFFSET for large tables)
SELECT * FROM posts WHERE created_at < $cursor ORDER BY created_at DESC LIMIT 20;

-- Avoid N+1 — use JOIN or batch
-- BAD: posts.map(p => getUser(p.userId))   → N queries
-- GOOD: JOIN or IN clause:
SELECT p.*, u.name, u.avatar FROM posts p JOIN users u ON u.id = p.user_id LIMIT 20;
```

```ts
// TanStack Query — client-side cache with stale-while-revalidate
const { data } = useQuery({
  queryKey: ['posts', userId],
  queryFn: () => fetchPosts(userId),
  staleTime: 60_000,       // consider fresh for 60s
  gcTime: 300_000,         // keep in memory 5 min after unused
})

// Prefetch on hover for instant navigation
const queryClient = useQueryClient()
<Link
  onMouseEnter={() => queryClient.prefetchQuery({ queryKey: ['post', id], queryFn: () => fetchPost(id) })}
>
```

---

## CSS Performance

```css
/* Prefer CSS transitions over JS animations */
.card { transition: transform var(--duration-normal) var(--ease-out); }
.card:hover { transform: translateY(-4px); }  /* compositor-only — no layout */

/* Use transform + opacity for animation (compositor thread, no repaint) */
@keyframes fade-in {
  from { opacity: 0; transform: translateY(8px); }
  to   { opacity: 1; transform: translateY(0); }
}

/* Avoid animating layout-triggering properties */
/* ❌ */ width, height, top, left, margin, padding
/* ✓ */ transform, opacity, filter, clip-path

/* Content-visibility for off-screen content (native lazy render) */
.below-fold-section {
  content-visibility: auto;
  contain-intrinsic-size: auto 500px;  /* estimated height to prevent CLS */
}

/* Critical CSS — inline in <head>, defer rest */
/* Next.js handles this automatically */
```

---

## Performance Anti-Patterns

```
JavaScript
✗  Import entire library (import _ from 'lodash') — always tree-shake
✗  useEffect for derived state — use useMemo
✗  New object/array literals in render without useMemo — causes child rerenders
✗  Synchronous localStorage in render — blocks paint
✗  document.querySelector in React — use refs

Images
✗  Large PNG/JPEG where SVG would work
✗  Missing width/height attributes (CLS)
✗  Lazy-loading the LCP image (delays LCP)
✗  Eager-loading below-fold images (wastes bandwidth)

Data
✗  SELECT * on large tables
✗  Fetching all records then filtering in JS — filter in SQL
✗  No pagination on lists
✗  Waterfall data fetching (await A, then await B) — use Promise.all
✗  Polling when WebSocket/SSE would do

CSS
✗  @import in CSS files (blocking — use <link> or import in JS)
✗  Animating width/height — use transform: scale() instead
✗  transition: all (hits every property including expensive ones)
✗  Huge unused CSS — use PurgeCSS / Tailwind's built-in purge
```

---

## Continuous Performance Check (run after any feature PR)

1. Does `next build` report a larger JS bundle? By how much? Justified?
2. Does the page still pass Lighthouse ≥90 perf?
3. Is the LCP element (hero image, H1, product title) above the fold and loading eagerly?
4. Are new images using Next/Image with proper `sizes`?
5. Are new data fetches parallel (Promise.all) not waterfall (sequential await)?
6. Are new lists paginated (not fetching all records)?
7. Are new DB queries indexed on their filter/sort columns?
