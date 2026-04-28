# Cross-Device / Cross-Platform Specialist Module

Activate when: mobile, tablet, iPhone, Android, responsive, breakpoint, 4", 7", 15",
PWA, native, cross-platform, Capacitor, iOS, Windows, Mac, safe area, viewport work.

---

## The Core Law

**Mobile is NOT a small desktop.** Design constraints first, aesthetics second.
Start every layout at 320px. Scale up. Never the reverse.

---

## Mandatory Pre-Work Questions

If any of these are not explicitly stated, ask before proceeding:

| Question | Why it matters |
|----------|----------------|
| Target platforms: web only / iOS / Android / desktop? | Changes navigation patterns, gestures, APIs |
| Minimum device width to support? | 320px (iPhone SE) or 375px minimum |
| Must it work offline? | Determines service worker + sync strategy |
| Native device APIs needed (camera, push, GPS)? | Determines PWA vs Capacitor vs Expo/RN |
| Any known accessibility requirements beyond WCAG AA? | Motor, visual, cognitive |

---

## Breakpoint System

```css
/* Tailwind v4 custom breakpoints — add to tailwind.config */
screens: {
  'xs':  '320px',   /* Small phones: iPhone SE, budget Android (4"–4.7") */
  'sm':  '480px',   /* Phones: 5"–7" */
  'md':  '768px',   /* Tablets: 7"–10" (iPad mini, Nexus 7) */
  'lg':  '1024px',  /* Tablets/small laptops: 11"–13" (iPad Pro, MacBook Air) */
  'xl':  '1440px',  /* Laptops/desktop: 13"–15" MacBook, 1440p monitors */
  '2xl': '1920px',  /* Large desktop: 27" iMac, 4K */
}
```

### Layout rules per breakpoint

| Breakpoint | Columns | Nav pattern | Primary action |
|------------|---------|-------------|----------------|
| xs (320–479) | 1 col | Bottom bar (5 items max) | Full-width sticky CTA |
| sm (480–767) | 1–2 col | Bottom bar or hamburger | Full-width or large pill |
| md (768–1023) | 2–3 col | Top bar or sidebar | Right-aligned or inline |
| lg (1024–1439) | 3–4 col | Sidebar or top nav | Inline |
| xl+ (1440+) | 4–12 col | Sidebar, expanded | Inline |

---

## Fluid Typography and Spacing

```css
/* Use clamp() — fluid between breakpoints, no abrupt jumps */
font-size: clamp(1rem, 2.5vw, 1.25rem);     /* base body text */
font-size: clamp(1.5rem, 4vw, 2.5rem);      /* h1 */
padding:   clamp(1rem, 4vw, 2rem);           /* section padding */

/* Container queries — component-level, not page-level */
@container card (min-width: 400px) {
  .card-body { display: flex; gap: 1rem; }
}
```

---

## Platform-Specific Quirks (all must be handled)

### iOS Safari
```css
/* 100vh bug — iOS Safari interprets 100vh as full height including address bar */
height: 100dvh;          /* dvh = dynamic viewport height — correct value */

/* Momentum scroll */
overflow-y: scroll;
-webkit-overflow-scrolling: touch;

/* Safe areas — notch, home bar, Dynamic Island */
padding-top:    env(safe-area-inset-top);
padding-bottom: env(safe-area-inset-bottom);
padding-left:   env(safe-area-inset-left);
padding-right:  env(safe-area-inset-right);

/* Tap delay elimination */
touch-action: manipulation;    /* on interactive elements */

/* Input zoom prevention (iOS zooms when font-size < 16px) */
input, select, textarea { font-size: max(16px, 1rem); }
```

### Android Chrome
```css
/* Address bar resize causes viewport jump */
/* Same fix: use dvh, svh (small), lvh (large) */
min-height: 100svh;    /* safe minimum — excludes address bar */

/* Bottom nav safe area for gesture navigation bar */
padding-bottom: env(safe-area-inset-bottom, 16px);
```

### Mac / Windows desktop
```
- Keyboard shortcuts: Cmd (Mac) vs Ctrl (Windows) — use navigator.platform or UA hints
- Right-click context menus: respect them, don't suppress
- Trackpad: inertia scroll works natively, don't override
- Cursor: pointer for clickable, text for selectable, grab for draggable
- Hover states: only apply with @media (hover: hover) — not on touch devices
```

### Touch vs pointer
```css
/* Hover effects only on pointer devices */
@media (hover: hover) and (pointer: fine) {
  .button:hover { background: var(--color-primary-600); }
}

/* Larger targets on touch devices */
@media (pointer: coarse) {
  .button { min-height: 48px; min-width: 48px; }
}
```

---

## Touch Target Rules

| Standard | Minimum size | Preferred |
|----------|-------------|-----------|
| iOS HIG | 44×44 pt | 44×44 pt |
| Material Design | 48×48 dp | 56×56 dp |
| WCAG 2.5.8 (AA) | 24×24 CSS px | 44×44 CSS px |
| WCAG 2.5.5 (AAA) | 44×44 CSS px | 44×44 CSS px |

```css
/* Expand tap target without changing visual size */
.small-icon-button {
  position: relative;
}
.small-icon-button::after {
  content: '';
  position: absolute;
  inset: -8px;       /* expand hitbox by 8px all sides */
  min-width: 44px;
  min-height: 44px;
}
```

---

## PWA Setup (required for any app targeting mobile)

### manifest.json
```json
{
  "name": "App Name",
  "short_name": "App",
  "description": "One line description",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "orientation": "portrait-primary",
  "icons": [
    { "src": "/icons/192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "/icons/512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ],
  "screenshots": [
    { "src": "/screenshots/mobile.png", "sizes": "390x844", "type": "image/png", "form_factor": "narrow" },
    { "src": "/screenshots/desktop.png", "sizes": "1280x720", "type": "image/png", "form_factor": "wide" }
  ]
}
```

### Service Worker (minimal viable offline)
```js
// public/sw.js — cache shell + network-first for API
const CACHE = 'v1';
const SHELL = ['/', '/offline.html', '/icons/192.png'];

self.addEventListener('install', e =>
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(SHELL))));

self.addEventListener('fetch', e => {
  if (e.request.mode === 'navigate') {
    e.respondWith(fetch(e.request).catch(() => caches.match('/offline.html')));
  }
});
```

---

## Capacitor — Native Wrap

Use when PWA cannot fulfil: push notifications on iOS <16.4, camera with full control,
background Bluetooth/NFC, App Store listing requirement.

```bash
# Setup
npm install @capacitor/core @capacitor/cli
npx cap init "App Name" com.company.app --web-dir=out
npm install @capacitor/ios @capacitor/android
npx cap add ios
npx cap add android

# Build + sync
npm run build && npx cap sync

# Common plugins
@capacitor/push-notifications   # APNs + FCM
@capacitor/camera               # full camera access
@capacitor/filesystem           # file access
@capacitor/haptics              # vibration feedback
@capacitor/splash-screen        # native splash
@capacitor/status-bar           # status bar color
```

### Capacitor testing requirements
- Test in iOS Simulator **and** real iPhone (Safari WebView behaviour differs)
- Test in Android Emulator **and** real Android device
- Test push notifications on real device (simulators don't receive push)
- Test "install from TestFlight / internal track" flow before release

---

## Device Test Matrix (test all before shipping)

| Device | Screen | Platform | Why |
|--------|--------|----------|-----|
| iPhone SE (3rd gen) | 375×667 | iOS Safari | Smallest common iPhone |
| iPhone 16 Pro Max | 430×932 | iOS Safari | Largest iPhone, Dynamic Island |
| Samsung Galaxy A15 | 412×915 | Chrome Android | Budget Android (largest segment) |
| Pixel 9 Pro | 412×892 | Chrome Android | Google reference device |
| iPad mini (6th gen) | 744×1133 | iPadOS Safari | 7" tablet |
| iPad Pro 12.9" | 1024×1366 | iPadOS Safari | 12" tablet |
| MacBook Pro 13" | 1280 logical | Safari Mac | Developer reference |
| 1920×1080 desktop | 1920 | Chrome Win | Corporate/enterprise |
| 4K 27" | 2560–3840 | Any | Power user, max width test |

### What to check on each device
```
[ ] Layout doesn't break or overflow
[ ] Text is readable without zooming
[ ] Touch targets are reachable with thumb (bottom of screen preferred zone)
[ ] Keyboard pushes layout correctly (doesn't cover active input)
[ ] Swipe/scroll works smoothly (no jank)
[ ] Safe area insets applied correctly (no content behind notch or home bar)
[ ] Images load at correct resolution (not blurry, not oversized)
[ ] Form inputs use native keyboard type (numeric, email, tel)
```

---

## Image and Media Responsive Rules

```html
<!-- Responsive image with AVIF + WebP + fallback -->
<picture>
  <source srcset="hero.avif" type="image/avif">
  <source srcset="hero.webp" type="image/webp">
  <img
    src="hero.jpg"
    alt="Descriptive alt text"
    width="1200"
    height="600"
    loading="lazy"          <!-- above-fold: loading="eager" -->
    decoding="async"
    sizes="(max-width: 768px) 100vw, 50vw"
    srcset="hero-400.webp 400w, hero-800.webp 800w, hero-1200.webp 1200w"
  >
</picture>

<!-- Video — never autoplay with sound -->
<video
  src="demo.mp4"
  autoplay muted loop playsinline
  poster="demo-poster.jpg"
  aria-label="Demo of X feature"
></video>
```

---

## Continuous Cross-Device Check (run after any layout change)

1. Does it look correct at 320px width without horizontal scroll?
2. Are all touch targets ≥44px on mobile breakpoints?
3. Is the `100dvh` / safe area fix applied to full-height layouts?
4. Does hover-only interaction have a touch equivalent?
5. Does the keyboard close gracefully on form submit on iOS?
6. Are images served at the right resolution (not 1200px image on a 375px screen)?
7. Does the install prompt / PWA manifest exist and work?
