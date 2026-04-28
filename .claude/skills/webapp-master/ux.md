# UX / UI Specialist Module

Activate when: UI, component, design, layout, CSS, style, animation, accessibility,
wireframe, visual, typography, brand, user experience work.

---

## Pre-Work Assessment (run before any design decision)

Score each dimension 1–5:

| Dimension | Question |
|-----------|----------|
| Aesthetic intent | Is there a defined visual direction (not just "clean")? |
| Functional clarity | Is the primary action on this screen unmistakable? |
| Accessibility readiness | Can a keyboard-only or screen-reader user complete the flow? |
| Performance safety | Will this render fast on a mid-range Android at 3G? |
| Consistency fit | Does this extend the existing design system or fracture it? |

Score < 3 on any dimension → resolve before writing code.

---

## Design System — Build Once, Use Everywhere

### Tokens (define in CSS variables / Tailwind theme, never hardcode)

```css
/* Type scale — modular ratio 1.25 */
--text-xs:   0.64rem;
--text-sm:   0.8rem;
--text-base: 1rem;
--text-lg:   1.25rem;
--text-xl:   1.563rem;
--text-2xl:  1.953rem;
--text-3xl:  2.441rem;

/* Spacing — 4-pt grid */
--space-1: 0.25rem;   /* 4px  */
--space-2: 0.5rem;    /* 8px  */
--space-3: 0.75rem;   /* 12px */
--space-4: 1rem;      /* 16px */
--space-6: 1.5rem;    /* 24px */
--space-8: 2rem;      /* 32px */
--space-12: 3rem;     /* 48px */
--space-16: 4rem;     /* 64px */

/* Color — OKLCH for perceptual uniformity */
--color-primary-500: oklch(55% 0.18 250);
--color-primary-600: oklch(48% 0.18 250);
--color-neutral-50:  oklch(98% 0 0);
--color-neutral-900: oklch(15% 0 0);
--color-success:     oklch(55% 0.15 145);
--color-warning:     oklch(65% 0.18 70);
--color-error:       oklch(55% 0.22 25);

/* Motion */
--duration-fast:   100ms;
--duration-normal: 200ms;
--duration-slow:   350ms;
--ease-out:        cubic-bezier(0.16, 1, 0.3, 1);
```

### Core Component Rules

```
Button     — 1 primary per screen · min 44×44px tap target · loading state required
             disabled = greyed + aria-disabled + tooltip explaining why
Input      — label always visible (no placeholder-only) · error message below · character count if limited
             autofocus only on first field of a focused form
Modal      — focus trap · Escape closes · scroll lock body · aria-modal · return focus on close
             max 1 modal at a time · no modal from modal
Navigation — max 5 primary items · active state · keyboard arrows through items
             mobile: bottom bar · desktop: sidebar or top bar (never both)
Toast      — max 1 visible · auto-dismiss 4s · manual dismiss always · aria-live="polite"
             error toasts: aria-live="assertive" · never auto-dismiss errors
Table      — sortable columns announce sort direction · pagination · empty state
             keyboard: Tab between cells, Enter to activate row action
```

### Dark Mode

```css
/* Use CSS variables — never duplicate classes */
:root {
  --bg-primary:   var(--color-neutral-50);
  --fg-primary:   var(--color-neutral-900);
  --border:       oklch(85% 0 0);
}
@media (prefers-color-scheme: dark) {
  :root {
    --bg-primary:   oklch(12% 0 0);
    --fg-primary:   oklch(92% 0 0);
    --border:       oklch(30% 0 0);
  }
}
/* also support data-theme="dark" for manual toggle */
```

---

## UX Principles — Non-Negotiable

### Information hierarchy
- 1 primary action per screen. Secondary actions visually subordinate.
- Progressive disclosure: show advanced options only when relevant.
- Chunk information: max 5–7 items per group.

### States — every user-facing element needs all four
```
Loading  → skeleton (not spinner for layout-affecting content)
          spinner only for actions (submit button, search)
Error    → specific message · recovery action · contact support link if persistent
Empty    → explain why empty · primary call-to-action to fill it
Offline  → detect navigator.onLine + service worker · show banner · queue mutations
```

### Feedback timing
```
<100ms    → feels instant — no indicator needed
100–300ms → debounce check (skeleton if content area changes)
300ms–1s  → show skeleton or spinner
>1s       → progress bar with estimated time if deterministic
>10s      → background task: let user leave, notify on completion
```

### Form UX
- Validate on blur, not on keystroke (except confirming format like email).
- Show errors inline, below the field, in red with icon.
- Never clear a form on navigation — save draft to localStorage.
- Mobile: set correct `inputmode`, `autocomplete`, `autocapitalize` attributes.
- Submit button: disable after submit, show spinner, re-enable on error.

### Animation rules
```
Purpose  → only animate if it communicates state change or spatial relationship
Duration → UI micro: 100ms · transitions: 200ms · page: 300ms · celebration: 500ms
Easing   → enter: ease-out · exit: ease-in · bounce: spring (framer-motion spring)
Respect  → always check prefers-reduced-motion:
           @media (prefers-reduced-motion: reduce) { * { animation: none !important; } }
```

---

## Accessibility Checklist (WCAG 2.2 AA)

```
Structure
[x] Single <h1> per page · logical heading order (h1→h2→h3, no skips)
[x] Landmark regions: <header>, <main>, <nav>, <footer>, <aside>
[x] Skip-to-main link as first focusable element
[x] Lists use <ul>/<ol>, not <div> with margin

Interactive elements
[x] All clickable elements are focusable (tabindex="0" or native element)
[x] Focus ring visible and ≥3:1 contrast against adjacent color
[x] Hover effects also work on focus
[x] Click targets ≥44×44px (WCAG 2.5.5 AAA target, 2.5.8 AA minimum 24px)
[x] No keyboard trap (except modal — trap is required there)

Images & icons
[x] Decorative images: alt=""
[x] Informative images: alt describes content
[x] Icon buttons: aria-label on button, icon itself aria-hidden="true"
[x] SVGs: role="img" aria-label="..." or aria-hidden="true"

Color & contrast
[x] Text ≥4.5:1 (normal) · ≥3:1 (large text ≥18pt or ≥14pt bold)
[x] UI components and focus rings ≥3:1 against adjacent colors
[x] Never use color alone to convey status — add icon or text

Dynamic content
[x] aria-live="polite" for non-urgent updates (counters, notifications)
[x] aria-live="assertive" for urgent interruptions only (errors)
[x] Route changes: move focus to <h1> or main content on SPA navigation
[x] Modal open: trap focus · close: return to trigger element

Forms
[x] Every input has <label> (not just placeholder)
[x] Error messages linked via aria-describedby
[x] Required fields: aria-required="true" · never use color alone
[x] Autocomplete attributes on personal data fields (name, email, tel)
```

---

## Anti-Patterns — Never Do These

```
Design
✗  Generic hero + 3 feature cards — AI slop default
✗  Carousels for primary content
✗  Modal from a modal
✗  Disabled button with no tooltip/explanation
✗  Placeholder-only labels (they disappear on focus)
✗  "Click here" / "Learn more" link text
✗  Infinite scroll without a way to reach the footer or page landmarks
✗  Hamburger menu on desktop for primary navigation

CSS
✗  z-index: 9999 (or more) — use a stacking context strategy
✗  !important outside of reset / utility overrides
✗  Fixed pixel font sizes (use rem)
✗  Overflow: hidden on body without scroll lock logic
✗  Hardcoded colors — always use tokens
✗  Transitions on all properties: transition: all — kills performance

Accessibility
✗  onClick on <div> (use <button>)
✗  aria-label duplicating visible text verbatim — combine instead
✗  tabindex > 0 — breaks natural focus order
✗  Removing outline without providing an alternative focus style
✗  Auto-playing media with sound
```

---

## Continuous UX Improvement Protocol

After every UI change, ask:
1. Can someone complete this task using only a keyboard?
2. Would a first-time user understand what to do on this screen without help?
3. What happens when the data takes 3 seconds to load?
4. What does this look like with 0 items? 1 item? 1000 items?
5. Does this still work if the user has reduced motion enabled?
6. Does this still pass contrast in both light and dark mode?

Fix any "no" before closing the task.
