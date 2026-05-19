# Universal App Blueprint
**Source:** Patterns proven in pl-dashboard-v8.3.0  
**Purpose:** A reusable design system and infrastructure library. Works for any app, any data domain.

---

## WHAT THIS DOCUMENT IS

This blueprint captures **how an app looks, behaves, and is wired together** — not what data it stores. The design system, navigation, auth, auto-save, backup, search, help, and security patterns here work for any app: task managers, CRMs, inventory tools, booking systems, logs, or anything else.

**What is domain-specific (you define this yourself):**
- Your data fields and record shape
- Your table columns
- Your KPI metrics and how they're computed
- Your charts and what they visualize (optional — many apps don't need them)

**What this blueprint gives you (proven, ready to use):**
- Design tokens: colors, spacing, typography, component styles
- App shell: header, sidebar, mobile bottom tabs, modal overlay system
- Navigation: view switching, active state, URL shortcuts
- Auth gate: login, session cache, sign-out
- Auto-save indicator: visual feedback on every write
- Three-layer backup: browser snapshots, local file, cloud sync
- Search bar: debounced, full-text, resets pagination
- Help system: slide-in FAQ drawer, keyboard shortcut reference
- Security layer: XSS prevention, CSV injection prevention, CSP
- Toast notifications + 6-second undo
- Settings panel
- CSV/file import with field mapping
- Data export (CSV, JSON)
- PWA shell for offline + mobile install
- Keyboard shortcuts
- Theme toggle (dark / light)

---

## COMPONENT SELECTION MATRIX

| ID | Component | What It Does | Required? |
|----|-----------|--------------|-----------|
| **SHELL** | | | |
| S1 | Design System | Colors, typography, spacing, all base component styles | Always |
| S2 | App Shell | Header, sidebar, mobile tabs, view-switching engine | Always |
| S3 | Modal System | Overlay dialogs, confirm, double-confirm, sheet | Always |
| S4 | Toast + Undo | Notification pop-ups with 6-second undo window | Always |
| **INFRASTRUCTURE** | | | |
| I1 | Data Layer | IndexedDB engine — generic CRUD, query, paginate | Always |
| I2 | Settings System | App config stored in localStorage | Always |
| I3 | Auto-Save Indicator | "Saving…" / "All saved" dot in header | Recommended |
| I4 | Security Layer | XSS guard, CSV injection guard, Content Security Policy | Always |
| **FEATURES** | | | |
| F1 | Auth Gate | Login screen, session cache (24h), sign-out | If access-controlled |
| F2 | Three-Layer Backup | Snapshots + local file + Google Sheets | Recommended |
| F3 | Search Bar | Debounced full-text search across any fields | Recommended |
| F4 | Help System | Slide-in FAQ drawer, keyboard shortcut list | Recommended |
| F5 | Navigation | Sidebar (desktop) + bottom tabs (mobile) + URL shortcuts | Always |
| F6 | Quick-Entry Form | Fast single-record add, autocomplete field, Enter to save | If app has data entry |
| F7 | Data Table | Paginated, sortable, filterable table with bulk-delete+undo | If app has lists |
| F8 | CSV / File Import | Upload spreadsheet, map columns, bulk insert | If bulk import needed |
| F9 | Tag Management | Create / rename / delete labels that categorize records | If records have tags |
| F10 | Data Export | Download records as CSV or JSON | If users need their data |
| F11 | Keyboard Shortcuts | Key bindings for navigation and common actions | Recommended |
| F12 | PWA / Offline | Works without internet, installable on phone | Recommended |
| **OPTIONAL DATA DISPLAY** | | | |
| D1 | KPI Summary Cards | Metric boxes (you define what metrics to show) | Optional |
| D2 | Charts | Charts (you define datasets and types) | Optional |
| D3 | Summary Table | Grouped/aggregated view of records | Optional |

---

## PART 1 — DESIGN SYSTEM (S1)

### Color Tokens

All colors live as CSS custom properties on `:root`. Components only reference variables — never hard-code hex values in components.

```css
/* ============================================================
   DARK THEME (default)
   Change the values here to match your brand.
   Do not change the variable names — components reference them.
   ============================================================ */
:root,
:root[data-theme="dark"] {

  /* Backgrounds */
  --bg:            #0f1117;   /* Page background */
  --surface:       #181c27;   /* Card, sidebar, modal background */
  --surface2:      #1f2435;   /* Hover state, input background, alt rows */

  /* Borders */
  --border:        #2a3045;   /* Default border */
  --border-strong: #3a4260;   /* Emphasized border, active element outline */

  /* Brand / Accent — CHANGE THIS to your app's primary color */
  --primary:       #0ecfbe;   /* Primary buttons, active nav, links */
  --primary-dim:   rgba(14, 207, 190, 0.12); /* Subtle tint for active bg */

  /* Semantic colors — keep meanings, change values for your brand */
  --success:       #27ae60;   /* Confirmations, positive states */
  --warning:       #f5a623;   /* Caution, medium-risk states */
  --danger:        #e74c3c;   /* Errors, destructive actions */
  --info:          #3b82f6;   /* Informational, neutral alerts */

  /* Text */
  --text:          #e8eaf0;   /* Primary body text */
  --muted:         #6b7394;   /* Labels, secondary text, placeholders */

  /* Effects */
  --shadow:        0 4px 24px rgba(0, 0, 0, 0.45);
  --shadow-sm:     0 2px 8px  rgba(0, 0, 0, 0.30);
  --radius:        10px;      /* Default border-radius for cards */
  --radius-sm:     6px;       /* Smaller radius for buttons, inputs */
  --focus:         #6ea8ff;   /* Keyboard focus ring color */
}

/* ============================================================
   LIGHT THEME
   ============================================================ */
:root[data-theme="light"] {
  --bg:            #f5f6fa;
  --surface:       #ffffff;
  --surface2:      #f0f2f8;
  --border:        #d4d8e8;
  --border-strong: #b0b8d0;
  --primary:       #0aab9e;
  --primary-dim:   rgba(10, 171, 158, 0.10);
  --success:       #16a34a;
  --warning:       #d97706;
  --danger:        #dc2626;
  --info:          #2563eb;
  --text:          #1a1f35;
  --muted:         #6b7394;
  --shadow:        0 2px 12px rgba(0, 0, 0, 0.12);
  --shadow-sm:     0 1px 4px  rgba(0, 0, 0, 0.08);
  --radius:        10px;
  --radius-sm:     6px;
  --focus:         #2563eb;
}
```

---

### Typography

```css
/* Load fonts — or remove and use system fonts only (works offline) */
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@400;500;600;700&display=swap');

body {
  font-family: 'DM Sans', system-ui, -apple-system, sans-serif;
  font-size:   15px;
  line-height: 1.55;
  color:       var(--text);
  background:  var(--bg);
}

/* Heading scale */
h1 { font-size: 24px; font-weight: 700; line-height: 1.2; }
h2 { font-size: 20px; font-weight: 700; line-height: 1.3; }
h3 { font-size: 16px; font-weight: 600; line-height: 1.4; }
h4 { font-size: 14px; font-weight: 600; line-height: 1.4; }

/* Page section title (used inside views) */
.page-title   { font-size: 22px; font-weight: 700; margin-bottom: 4px; }
.page-subtitle { font-size: 13px; color: var(--muted); margin-bottom: 24px; }

/* Label above a form field or section */
.label     { font-size: 13px; color: var(--muted); font-weight: 500; margin-bottom: 5px; display: block; }
.label-sm  { font-size: 11px; color: var(--muted); font-weight: 600;
             text-transform: uppercase; letter-spacing: .06em; }
```

---

### Spacing Scale

Use multiples of 4px. Don't use arbitrary values.

```
4px   — tight (icon gap, badge padding)
8px   — small (between form label and input)
12px  — medium-small (inside compact card)
16px  — medium (standard gap between items)
20px  — medium-large (card padding on mobile)
24px  — large (between sections)
28px  — standard desktop main padding
32px  — desktop main content padding
40px  — section breathing room, empty states
```

---

### Base Component Styles

```css
/* ── RESET ─────────────────────────────────────────────── */
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
button   { cursor: pointer; font-family: inherit; }
input, select, textarea { font-family: inherit; font-size: 15px; }
a        { color: var(--primary); text-decoration: none; }
a:hover  { text-decoration: underline; }
img, svg { display: block; }

/* ── CARDS ──────────────────────────────────────────────── */
.card {
  background:    var(--surface);
  border:        1px solid var(--border);
  border-radius: var(--radius);
  padding:       20px;
}
.card-sm { padding: 14px 16px; }
.card-lg { padding: 28px 32px; }

/* ── BUTTONS ────────────────────────────────────────────── */
.btn {
  display:         inline-flex;
  align-items:     center;
  gap:             6px;
  padding:         8px 18px;
  border-radius:   var(--radius-sm);
  border:          none;
  font-size:       14px;
  font-weight:     500;
  line-height:     1.4;
  transition:      opacity .15s, background .15s;
  white-space:     nowrap;
}
.btn:hover        { opacity: .85; }
.btn:focus-visible { outline: 2px solid var(--focus); outline-offset: 2px; }
.btn:disabled     { opacity: .4; cursor: not-allowed; }

.btn-primary { background: var(--primary);  color: #000; }
.btn-danger  { background: var(--danger);   color: #fff; }
.btn-outline { background: transparent;     color: var(--text);  border: 1px solid var(--border); }
.btn-ghost   { background: transparent;     color: var(--muted); }
.btn-sm      { padding: 5px 12px; font-size: 13px; }
.btn-icon    { padding: 6px; border-radius: 6px; }  /* icon-only button */

/* ── FORM CONTROLS ──────────────────────────────────────── */
.input {
  width:         100%;
  padding:       9px 12px;
  background:    var(--surface2);
  border:        1px solid var(--border);
  border-radius: var(--radius-sm);
  color:         var(--text);
  font-size:     15px;      /* 16px min on iOS prevents zoom; 15px is safe with viewport meta */
  transition:    border-color .15s;
}
.input:focus        { outline: 2px solid var(--focus); border-color: transparent; }
.input::placeholder { color: var(--muted); }

select.input { appearance: none; background-image: url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='8' viewBox='0 0 12 8'%3E%3Cpath d='M1 1l5 5 5-5' stroke='%236b7394' stroke-width='1.5' fill='none'/%3E%3C/svg%3E"); background-repeat: no-repeat; background-position: right 12px center; padding-right: 32px; }

.form-row    { margin-bottom: 16px; }
.form-error  { font-size: 13px; color: var(--danger); margin-top: 5px; }

/* ── BADGES / CHIPS ─────────────────────────────────────── */
.badge {
  display:       inline-block;
  padding:       2px 10px;
  border-radius: 20px;
  font-size:     12px;
  font-weight:   500;
  white-space:   nowrap;
}
.badge-primary { background: var(--primary-dim); color: var(--primary); }
.badge-success { background: rgba(39,174,96,.15);  color: var(--success); }
.badge-danger  { background: rgba(231,76,60,.15);  color: var(--danger);  }
.badge-warning { background: rgba(245,166,35,.15); color: var(--warning); }
.badge-neutral { background: var(--surface2);      color: var(--muted);   }

/* Filter chips (for active/inactive toggle filters) */
.chip          { padding: 4px 14px; border-radius: 20px; border: 1px solid var(--border);
                 background: transparent; color: var(--muted); cursor: pointer; font-size: 13px; }
.chip.active   { border-color: var(--primary); color: var(--primary); background: var(--primary-dim); }

/* ── TABLES ─────────────────────────────────────────────── */
.table-wrap { overflow-x: auto; border-radius: var(--radius); border: 1px solid var(--border); }
table       { width: 100%; border-collapse: collapse; font-size: 14px; }
thead       { background: var(--surface2); }
th          { text-align: left; padding: 10px 14px; color: var(--muted);
              font-size: 11px; font-weight: 600; text-transform: uppercase;
              letter-spacing: .05em; border-bottom: 1px solid var(--border);
              white-space: nowrap; }
td          { padding: 12px 14px; border-bottom: 1px solid var(--border); vertical-align: middle; }
tr:last-child td  { border-bottom: none; }
tbody tr:hover td { background: var(--surface2); }
th.sortable       { cursor: pointer; user-select: none; }
th.sortable:hover { color: var(--text); }

/* ── DIVIDERS ───────────────────────────────────────────── */
.divider { border: none; border-top: 1px solid var(--border); margin: 20px 0; }

/* ── ICONS (emoji-based — no external icon library needed) ─ */
/*
  Use system emoji for all icons. They render everywhere with no dependencies.
  Recommended icon set used in the reference implementation:

  Navigation:    🏠 📊 📋 📅 ➕ 📤 🏷 💾 ⚙️ ☰ ✕
  Actions:       🗑 ✏️ 📋 💾 📥 📤 ↩ ✓ ⚠️
  Status:        ✅ ❌ ⏳ 🔄 🔒 🔓
  Feedback:      ☀️ 🌙 ❓ 🔍 🔔 ℹ️
  Data:          📈 📉 💰 📊

  To use a custom SVG icon instead, wrap in:
  <span class="icon" aria-hidden="true"><!-- svg here --></span>
*/
.icon-btn {
  padding:        6px 8px;
  border-radius:  6px;
  border:         none;
  background:     transparent;
  color:          var(--muted);
  cursor:         pointer;
  font-size:      16px;
  line-height:    1;
  transition:     background .15s, color .15s;
  min-width:      32px;
  min-height:     32px;   /* accessibility: 32px minimum tap target */
  display:        inline-flex;
  align-items:    center;
  justify-content:center;
}
.icon-btn:hover { background: var(--surface2); color: var(--text); }

/* ── EMPTY STATE ────────────────────────────────────────── */
.empty-state {
  text-align:  center;
  padding:     60px 20px;
  color:       var(--muted);
}
.empty-state h3  { font-size: 18px; margin-bottom: 8px; color: var(--text); }
.empty-state p   { font-size: 14px; margin-bottom: 20px; }

/* ── SCROLLBAR ──────────────────────────────────────────── */
::-webkit-scrollbar       { width: 5px; height: 5px; }
::-webkit-scrollbar-track { background: transparent; }
::-webkit-scrollbar-thumb { background: var(--border); border-radius: 3px; }
```

---

## PART 2 — APP SHELL (S2)

### Layout Structure

```
┌────────────────────────────── HEADER (58px, sticky) ───────────────────────────────┐
│  [☰ Hamburger mobile]  App Name                  [Save Status]  [? Help]  [☀ Theme] │
└────────────────────────────────────────────────────────────────────────────────────┘
┌──────────────┬─────────────────────────────────────────────────────────────────────┐
│              │                                                                     │
│  SIDEBAR     │  MAIN CONTENT AREA                                                  │
│  (220px)     │  (scrollable)                                                       │
│              │                                                                     │
│  Nav Section │  Page Header                                                        │
│  ─ Item 1    │    Title + Subtitle                                                 │
│  ─ Item 2    │    Action buttons / filter bar                                      │
│  ─ Item 3    │                                                                     │
│              │  View Content                                                       │
│  Nav Section │    Cards / Table / Form / etc.                                      │
│  ─ Item 4    │                                                                     │
│  ─ Item 5    │                                                                     │
│              │                                                                     │
│  ─ ─ ─ ─ ─  │                                                                     │
│  Danger Zone │                                                                     │
└──────────────┴─────────────────────────────────────────────────────────────────────┘
                                              (mobile only below)
┌────────────────────────────── BOTTOM TAB BAR (60px, fixed) ────────────────────────┐
│      [Tab 1]          [Tab 2]          [+ FAB]          [Tab 3]        [☰ More]   │
└────────────────────────────────────────────────────────────────────────────────────┘
┌─── TOAST STACK (fixed, bottom-center) ─────────────────────────────────────────────┐
│    [✓ Saved]   [⚠ Warning]   [✕ Error]                                            │
└────────────────────────────────────────────────────────────────────────────────────┘
```

### HTML Shell

```html
<!DOCTYPE html>
<html lang="en" data-theme="dark">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1, viewport-fit=cover">
  <!-- S4: Content Security Policy (adjust allowed domains as needed) -->
  <meta http-equiv="Content-Security-Policy"
    content="default-src 'self';
             script-src  'self' 'unsafe-inline' https://accounts.google.com;
             style-src   'self' 'unsafe-inline' https://fonts.googleapis.com;
             font-src    'self' https://fonts.gstatic.com;
             connect-src 'self' https://*.supabase.co https://sheets.googleapis.com https://www.googleapis.com;">
  <title>APP NAME</title>
  <link rel="manifest" href="manifest.json">
  <style>/* Paste Design System CSS here */</style>
</head>
<body>

<!-- ═══════════════════ HEADER ═══════════════════ -->
<header id="topbar">
  <div class="topbar-left">
    <!-- Mobile sidebar toggle (hidden on desktop) -->
    <button id="sidebar-toggle" class="icon-btn mobile-only" onclick="toggleSidebar()" aria-label="Menu">☰</button>
    <!-- App logo / name -->
    <div class="app-brand">
      <span class="app-name">APP NAME</span>
    </div>
  </div>
  <div class="topbar-right">
    <!-- I3: Auto-save indicator -->
    <span id="save-status" class="save-status" aria-live="polite"></span>
    <!-- F3: Search (optional — some apps put this in each view instead) -->
    <button class="icon-btn mobile-only" onclick="toggleMobileSearch()" aria-label="Search">🔍</button>
    <!-- F4: Help -->
    <button class="icon-btn" onclick="openHelp()" aria-label="Help">❓</button>
    <!-- S1: Theme toggle -->
    <button id="theme-toggle" class="icon-btn" onclick="toggleTheme()" aria-label="Toggle theme">☀️</button>
    <!-- F1: User avatar / sign out (if auth enabled) -->
    <div id="user-menu" style="display:none">
      <button class="icon-btn" onclick="openUserMenu()" id="user-avatar" aria-label="Account">👤</button>
    </div>
  </div>
</header>

<!-- ═══════════════════ APP BODY ═══════════════════ -->
<div id="app">

  <!-- F5: Sidebar navigation (desktop) -->
  <nav id="sidebar" aria-label="Main navigation">
    <!-- Optional: app/user info at top of sidebar -->
    <div id="sidebar-info" style="display:none; padding:14px 16px 8px; border-bottom:1px solid var(--border)">
      <div style="font-size:13px;font-weight:600" id="sidebar-app-name"></div>
      <div style="font-size:11px;color:var(--muted)" id="sidebar-sub"></div>
    </div>
    <!-- Nav groups injected by buildNav() — see F5 -->
  </nav>

  <!-- Mobile sidebar overlay -->
  <div id="sidebar-overlay" style="display:none;position:fixed;inset:0;background:rgba(0,0,0,.5);z-index:149" onclick="closeSidebar()"></div>

  <!-- Main content -->
  <main id="main" role="main">
    <!-- Views injected here — each is a div.view, only one visible at a time -->
    <!-- Example: <div id="home-view"   class="view"> ... </div> -->
    <!-- Example: <div id="list-view"   class="view"> ... </div> -->
    <!-- Example: <div id="add-view"    class="view"> ... </div> -->
  </main>

</div><!-- /#app -->

<!-- F5: Mobile bottom tab bar -->
<nav id="bottombar" aria-label="Tab bar"></nav>

<!-- S3: Modal container (modals are appended here by JS) -->
<div id="modal-root"></div>

<!-- F4: Help drawer -->
<div id="help-drawer" style="display:none">
  <div id="help-backdrop"  onclick="closeHelp()"></div>
  <div id="help-panel">
    <div class="help-header">
      <h2>Help</h2>
      <button class="icon-btn" onclick="closeHelp()">✕</button>
    </div>
    <div id="help-content"><!-- Injected by openHelp() --></div>
  </div>
</div>

<!-- S4: Toast notification container -->
<div id="toast" role="status" aria-live="polite" aria-atomic="false"></div>

<!-- F1: Auth gate (shown before login, hidden after) -->
<div id="auth-gate" style="display:none"><!-- See F1 --></div>

<script>/* All JavaScript below */</script>
</body>
</html>
```

### Header CSS

```css
#topbar {
  height:          58px;
  background:      var(--surface);
  border-bottom:   1px solid var(--border);
  display:         flex;
  align-items:     center;
  justify-content: space-between;
  padding:         0 20px;
  position:        sticky;
  top:             0;
  z-index:         100;
  gap:             12px;
}
.topbar-left  { display: flex; align-items: center; gap: 10px; }
.topbar-right { display: flex; align-items: center; gap: 4px; }

.app-brand    { display: flex; align-items: center; gap: 10px; }
.app-name     { font-size: 16px; font-weight: 700; color: var(--text); }

/* I3: Auto-save dot + text */
.save-status  { font-size: 12px; color: var(--muted); display: flex; align-items: center; gap: 5px; }
.save-dot     { width: 7px; height: 7px; border-radius: 50%; display: inline-block; }
.save-dot.saving  { background: var(--warning); animation: pulse 1s infinite; }
.save-dot.saved   { background: var(--success); }
.save-dot.error   { background: var(--danger); }
@keyframes pulse  { 0%,100% { opacity:1; } 50% { opacity:.4; } }
```

### App Layout CSS

```css
#app {
  display:   flex;
  height:    calc(100vh - 58px);
  overflow:  hidden;
}

/* ── Sidebar ── */
#sidebar {
  width:        220px;
  flex-shrink:  0;
  background:   var(--surface);
  border-right: 1px solid var(--border);
  overflow-y:   auto;
  padding:      12px 0;
  transition:   transform .25s;
}

.nav-section  { margin-bottom: 4px; }
.nav-label    { font-size: 10px; font-weight: 700; text-transform: uppercase;
                letter-spacing: .10em; color: var(--muted); padding: 14px 16px 5px; }
.nav-item     { display: flex; align-items: center; gap: 10px; padding: 9px 16px;
                margin: 1px 8px; border-radius: var(--radius-sm); color: var(--muted);
                cursor: pointer; font-size: 14px; transition: all .12s; user-select: none; }
.nav-item:hover  { background: var(--surface2); color: var(--text); }
.nav-item.active { background: var(--primary-dim); color: var(--primary); font-weight: 600; }
.nav-item .nav-icon { width: 18px; text-align: center; flex-shrink: 0; }
.nav-item .nav-badge { margin-left: auto; background: var(--primary); color: #000;
                       font-size: 10px; font-weight: 700; border-radius: 10px; padding: 1px 6px; }
.nav-danger { color: var(--danger) !important; }
.nav-danger:hover { background: rgba(231,76,60,.10) !important; }

/* ── Main Content ── */
#main {
  flex:       1;
  overflow-y: auto;
  padding:    28px 32px;
}

.view { display: none; }
.view.active { display: block; }

/* Standard view page header */
.view-header         { margin-bottom: 24px; }
.view-header-row     { display: flex; align-items: flex-start; justify-content: space-between; flex-wrap: wrap; gap: 12px; }
.view-actions        { display: flex; gap: 8px; flex-wrap: wrap; }

/* ── Bottom tab bar (mobile only) ── */
#bottombar {
  display:     none;
  position:    fixed;
  bottom:      0;
  left:        0;
  right:       0;
  background:  var(--surface);
  border-top:  1px solid var(--border);
  z-index:     100;
  /* safe-area-inset for notch phones */
  padding-bottom: env(safe-area-inset-bottom, 0);
}
#bottombar button {
  flex:            1;
  padding:         10px 0 6px;
  background:      none;
  border:          none;
  color:           var(--muted);
  font-size:       10px;
  display:         flex;
  flex-direction:  column;
  align-items:     center;
  gap:             3px;
  transition:      color .12s;
}
#bottombar button .tab-icon  { font-size: 20px; line-height: 1; }
#bottombar button.active      { color: var(--primary); }
/* FAB center button */
#bottombar button.tab-fab    { color: var(--primary); }
#bottombar button.tab-fab .tab-icon {
  background:    var(--primary);
  color:         #000;
  border-radius: 50%;
  width:         40px;
  height:        40px;
  display:       flex;
  align-items:   center;
  justify-content: center;
  font-size:     22px;
  font-weight:   700;
  box-shadow:    0 2px 10px rgba(14,207,190,.4);
}

/* ── Responsive breakpoints ── */
@media (max-width: 640px) {
  #sidebar   { display: none; position: fixed; top: 58px; bottom: 0; left: 0;
               z-index: 150; transform: translateX(-100%); }
  #sidebar.open { display: block; transform: translateX(0); }
  #main      { padding: 16px; padding-bottom: calc(72px + env(safe-area-inset-bottom, 0)); }
  #bottombar { display: flex; }
  #app       { height: calc(100vh - 58px); }
  .mobile-only { display: flex !important; }
  .desktop-only { display: none !important; }
}
@media (min-width: 641px) {
  .mobile-only  { display: none !important; }
  .desktop-only { display: flex !important; }
}
@media (max-width: 900px) {
  #main { padding: 20px 20px; }
}
```

---

## PART 3 — MODAL SYSTEM (S3)

### Standard Overlay Modal

```javascript
// Generic modal builder — use for any dialog in your app
function showModal({ id, title, body, actions, width = '480px' }) {
  const existing = document.getElementById(id);
  if (existing) existing.remove();

  const modal = document.createElement('div');
  modal.id    = id;
  modal.style.cssText = `position:fixed;inset:0;background:rgba(0,0,0,.65);
    z-index:500;display:flex;align-items:center;justify-content:center;padding:16px`;

  modal.innerHTML = `
    <div style="background:var(--surface);border:1px solid var(--border);border-radius:var(--radius);
      width:min(${width},100%);max-height:90vh;overflow-y:auto;padding:28px;box-shadow:var(--shadow)">
      <div style="display:flex;align-items:flex-start;justify-content:space-between;margin-bottom:20px">
        <h2 style="font-size:18px;font-weight:700">${escapeHtml(title)}</h2>
        <button class="icon-btn" onclick="document.getElementById('${id}').remove()">✕</button>
      </div>
      <div id="${id}-body">${body}</div>
      ${actions ? `<div style="display:flex;gap:8px;margin-top:24px;justify-content:flex-end">${actions}</div>` : ''}
    </div>`;

  // Click outside to close
  modal.addEventListener('click', e => { if (e.target === modal) modal.remove(); });
  document.getElementById('modal-root').appendChild(modal);
  return modal;
}

// Single confirm dialog
function showConfirm(title, message, onConfirm) {
  showModal({
    id:      'confirm-modal',
    title,
    body:    `<p style="color:var(--muted)">${escapeHtml(message)}</p>`,
    actions: `
      <button class="btn btn-outline" onclick="document.getElementById('confirm-modal').remove()">Cancel</button>
      <button class="btn btn-danger" onclick="document.getElementById('confirm-modal').remove();(${onConfirm})()">Confirm</button>`
  });
}

// Double-confirm — for destructive irreversible actions only (e.g. "Clear All Data")
function showDoubleConfirm(title, message, onConfirmed) {
  showConfirm(title, message, () => {
    showConfirm(
      'Are you absolutely sure?',
      'This cannot be undone. Click Confirm to permanently delete all data.',
      onConfirmed
    );
  });
}

// Close all open modals (e.g. on Escape key)
function closeAllModals() {
  document.querySelectorAll('[id$="-modal"]').forEach(m => m.remove());
  closeHelp();
}
```

### Modal CSS

```css
/* Bottom sheet on mobile (alternative to centered modal) */
.bottom-sheet {
  position:   fixed;
  bottom:     0; left: 0; right: 0;
  background: var(--surface);
  border-top: 1px solid var(--border);
  border-radius: var(--radius) var(--radius) 0 0;
  padding:    24px;
  z-index:    500;
  max-height: 80vh;
  overflow-y: auto;
  box-shadow: var(--shadow);
  animation:  slideUp .2s ease;
}
@keyframes slideUp { from { transform: translateY(100%); } to { transform: translateY(0); } }
```

---

## PART 4 — TOAST + UNDO SYSTEM (S4)

### Toast Component

```javascript
// Use textContent (not innerHTML) for the message — toasts can include user-sourced text
function toast(msg, type = 'info', opts = {}) {
  const container = document.getElementById('toast');
  const duration  = opts.duration || 3200;

  const colors = {
    success: 'background:var(--success);color:#fff',
    error:   'background:var(--danger);color:#fff',
    warning: 'background:var(--warning);color:#000',
    info:    'background:var(--surface2);color:var(--text);border:1px solid var(--border)',
  };

  const el = document.createElement('div');
  el.style.cssText = `padding:11px 18px;border-radius:8px;font-size:14px;cursor:pointer;
    box-shadow:var(--shadow);display:flex;align-items:center;gap:10px;max-width:380px;
    ${colors[type] || colors.info}`;

  el.appendChild(document.createTextNode(msg)); // safe: no innerHTML

  if (opts.action) {
    const btn = document.createElement('button');
    btn.textContent = opts.action.label;
    btn.style.cssText = 'background:rgba(255,255,255,.22);border:none;color:inherit;padding:2px 10px;border-radius:4px;cursor:pointer;font-weight:600;flex-shrink:0';
    btn.onclick = e => { e.stopPropagation(); opts.action.onClick(); el.remove(); };
    el.appendChild(btn);
  }

  el.addEventListener('click', () => el.remove());
  container.appendChild(el);

  const timer = setTimeout(() => {
    el.style.transition = 'opacity .3s';
    el.style.opacity    = '0';
    setTimeout(() => el.remove(), 300);
  }, duration);

  el.dataset.timer = timer;
}

// Undo: 6-second window to restore a deleted set of records
// Call AFTER dbDelete/dbBulkDelete, pass the full record objects
function offerUndo(label, deletedRecords, onRestored) {
  const saved = deletedRecords.map(r => ({ ...r })); // clone before any mutations

  toast(label, 'info', {
    duration: 6000,
    action: {
      label:   'Undo',
      onClick: async () => {
        await dbBulkPut(saved);
        scheduleSnapshot();
        if (onRestored) onRestored();
        toast('Restored', 'success');
      }
    }
  });
}
```

### Toast CSS

```css
#toast {
  position:        fixed;
  bottom:          24px;
  left:            50%;
  transform:       translateX(-50%);
  z-index:         9999;
  display:         flex;
  flex-direction:  column;
  gap:             8px;
  align-items:     center;
  pointer-events:  none; /* container transparent to clicks */
}
#toast > * { pointer-events: all; }

/* On mobile: sit above bottom tab bar */
@media (max-width: 640px) {
  #toast { bottom: calc(68px + env(safe-area-inset-bottom, 0)); }
}
```

---

## PART 5 — INFRASTRUCTURE LAYER

### I1: Generic Data Layer (IndexedDB)

```javascript
// ── CONFIGURE THIS BLOCK FOR YOUR APP ────────────────────────────────────
const DB_NAME    = 'YOUR_APP_NAME_DB';   // e.g. 'TaskManagerDB'
const DB_VERSION = 1;
const STORE_NAME = 'records';            // e.g. 'tasks', 'contacts', 'items'

// Add an index for every field you filter or sort by in queries
// Format: { name: 'indexName', keyPath: 'fieldName' }
const DB_INDEXES = [
  // Examples — uncomment what you need:
  // { name: 'date',     keyPath: 'date'     },
  // { name: 'status',   keyPath: 'status'   },
  // { name: 'category', keyPath: 'category' },
];
// ─────────────────────────────────────────────────────────────────────────

let db = null;

async function initDB() {
  return new Promise((resolve, reject) => {
    const req = indexedDB.open(DB_NAME, DB_VERSION);
    req.onupgradeneeded = e => {
      const d = e.target.result;
      if (!d.objectStoreNames.contains(STORE_NAME)) {
        const store = d.createObjectStore(STORE_NAME, { keyPath: 'id' });
        DB_INDEXES.forEach(i => store.createIndex(i.name, i.keyPath, { unique: false }));
      }
    };
    req.onsuccess = e => { db = e.target.result; resolve(db); };
    req.onerror   = () => {
      // NEVER fail silently — always show a visible error to the user
      document.body.innerHTML = `
        <div style="min-height:100vh;display:flex;align-items:center;justify-content:center;
          font-family:sans-serif;background:#0f1117;color:#e8eaf0;text-align:center;padding:40px">
          <div>
            <div style="font-size:48px;margin-bottom:16px">⚠️</div>
            <h2>Storage Unavailable</h2>
            <p style="color:#6b7394;margin-top:8px">
              This app requires browser storage.<br>
              Try Chrome or Edge, and disable private/incognito mode.
            </p>
          </div>
        </div>`;
      reject(req.error);
    };
  });
}

// ── CRUD ─────────────────────────────────────────────────────────────────

function dbPut(record) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    tx.objectStore(STORE_NAME).put(record).onsuccess = () => resolve();
    tx.onerror = e => reject(e.target.error);
  });
}

function dbGet(id) {
  return new Promise((resolve, reject) => {
    const tx  = db.transaction(STORE_NAME, 'readonly');
    const req = tx.objectStore(STORE_NAME).get(id);
    req.onsuccess = () => resolve(req.result);
    tx.onerror    = e => reject(e.target.error);
  });
}

function dbDelete(id) {
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    tx.objectStore(STORE_NAME).delete(id).onsuccess = () => resolve();
    tx.onerror = e => reject(e.target.error);
  });
}

function dbGetAll() {
  return new Promise((resolve, reject) => {
    const tx  = db.transaction(STORE_NAME, 'readonly');
    const req = tx.objectStore(STORE_NAME).getAll();
    req.onsuccess = () => resolve(req.result || []);
    tx.onerror    = e => reject(e.target.error);
  });
}

function dbBulkPut(records, onProgress) {
  return new Promise((resolve, reject) => {
    if (!records.length) { resolve(); return; }
    const tx    = db.transaction(STORE_NAME, 'readwrite');
    const store = tx.objectStore(STORE_NAME);
    let done    = 0;
    records.forEach(r => {
      const req = store.put(r);
      req.onsuccess = () => { done++; if (onProgress) onProgress(done, records.length); if (done === records.length) resolve(); };
    });
    tx.onerror = e => reject(e.target.error);
  });
}

function dbClear() {
  return new Promise((resolve, reject) => {
    const tx = db.transaction(STORE_NAME, 'readwrite');
    tx.objectStore(STORE_NAME).clear().onsuccess = () => resolve();
    tx.onerror = e => reject(e.target.error);
  });
}

function dbCount() {
  return new Promise((resolve, reject) => {
    const tx  = db.transaction(STORE_NAME, 'readonly');
    const req = tx.objectStore(STORE_NAME).count();
    req.onsuccess = () => resolve(req.result);
    tx.onerror    = e => reject(e.target.error);
  });
}

// Unique ID — always use this, never sequential counters
function newId() { return Date.now() + Math.random(); }

// Advanced in-memory query (filter, sort, paginate after dbGetAll)
async function dbQuery({
  search   = '',
  filters  = {},       // e.g. { status: 'active', category: 'Work' }
  sort     = 'id',
  dir      = 'desc',
  page     = 1,
  pageSize = 25
} = {}) {
  let rows = await dbGetAll();

  // Exact-match filters
  Object.entries(filters).forEach(([key, val]) => {
    if (val !== '' && val != null) rows = rows.filter(r => String(r[key]) === String(val));
  });

  // Full-text search across SEARCH_FIELDS (configure this for your app)
  if (search.trim()) {
    const q = search.trim().toLowerCase();
    rows = rows.filter(r =>
      SEARCH_FIELDS.some(f => String(r[f] ?? '').toLowerCase().includes(q))
    );
  }

  // Sort (handles strings and numbers)
  rows.sort((a, b) => {
    let av = a[sort] ?? '', bv = b[sort] ?? '';
    if (!isNaN(+av) && !isNaN(+bv)) { av = +av; bv = +bv; }
    if (av < bv) return dir === 'asc' ? -1 : 1;
    if (av > bv) return dir === 'asc' ?  1 : -1;
    return 0;
  });

  const total = rows.length;
  const start = (page - 1) * pageSize;
  return { rows: rows.slice(start, start + pageSize), total };
}

// ── CONFIGURE: fields included in full-text search ────────────────────────
const SEARCH_FIELDS = ['name', 'description', 'category'];
// ─────────────────────────────────────────────────────────────────────────
```

---

### I2: Settings System

```javascript
// ── CONFIGURE ─────────────────────────────────────────────────────────────
const SETTINGS_KEY = 'YOUR_APP_settings_v1';

// Define your app's scalar settings here (all values are strings or numbers)
let settings = {
  appName:     '',      // Displayed in sidebar info section
  // Add your own:
  // defaultView: 'list',
  // itemsPerPage: 25,
  // timezone:  'UTC',
};
// ─────────────────────────────────────────────────────────────────────────

function loadSettings() {
  try {
    const raw = localStorage.getItem(SETTINGS_KEY);
    if (raw) settings = { ...settings, ...JSON.parse(raw) };
  } catch { /* use defaults */ }
}

function saveSettings() {
  localStorage.setItem(SETTINGS_KEY, JSON.stringify(settings));
  updateSidebarInfo(); // reflect in UI immediately
}

function updateSidebarInfo() {
  const nameEl = document.getElementById('sidebar-app-name');
  const subEl  = document.getElementById('sidebar-sub');
  const infoEl = document.getElementById('sidebar-info');
  if (!nameEl) return;
  if (settings.appName) {
    nameEl.textContent = settings.appName;
    if (subEl && settings.subLabel) subEl.textContent = settings.subLabel;
    if (infoEl) infoEl.style.display = 'block';
  } else {
    if (infoEl) infoEl.style.display = 'none';
  }
}
```

---

### I3: Auto-Save Indicator

```javascript
// Shows a pulsing dot + "Saving..." when data is being written,
// then "All saved" when complete, then fades out after 3s.
function setSaveStatus(state) {
  const el = document.getElementById('save-status');
  if (!el) return;

  const states = {
    saving: { dot: 'saving', text: 'Saving…' },
    saved:  { dot: 'saved',  text: 'All saved' },
    error:  { dot: 'error',  text: 'Save error' },
  };

  const s = states[state];
  if (!s) { el.innerHTML = ''; return; }

  el.innerHTML = `<span class="save-dot ${s.dot}"></span>${s.dot === 'saving' ? s.text : ''}`;

  if (state === 'saved') {
    clearTimeout(el._timer);
    el._timer = setTimeout(() => { el.innerHTML = ''; }, 3000);
  }
}

// Wrap every dbPut / dbBulkPut call with these two lines:
// setSaveStatus('saving');
// await dbPut(record);
// setSaveStatus('saved');

// Or use this wrapper so you never forget:
async function dbPutWithStatus(record) {
  setSaveStatus('saving');
  try {
    await dbPut(record);
    setSaveStatus('saved');
    scheduleSnapshot();
  } catch (e) {
    setSaveStatus('error');
    toast('Could not save: ' + e.message, 'error');
    throw e;
  }
}
```

---

### I4: Security Layer

```javascript
// ── RULE 1: XSS Prevention ───────────────────────────────────────────────
// Wrap EVERY user-sourced value before inserting into innerHTML.
// When to use:  element.innerHTML = '...' + escapeHtml(userValue) + '...'
// When NOT needed: element.textContent = value  (DOM text node, always safe)
//                  input.value = value           (DOM property, always safe)
function escapeHtml(s) {
  return String(s == null ? '' : s).replace(/[&<>"']/g, c => ({
    '&':'&amp;', '<':'&lt;', '>':'&gt;', '"':'&quot;', "'":'&#39;'
  }[c]));
}

// ── RULE 2: CSV / Excel Injection Prevention ──────────────────────────────
// Wrap EVERY cell value before writing to a CSV or Excel file.
// Why: Excel executes cells starting with =, +, -, @, TAB, CR as formulas.
function csvSanitize(v) {
  const s = String(v == null ? '' : v);
  return /^[=+\-@\t\r]/.test(s) ? "'" + s : s;
}

// ── RULE 3: onclick with string parameters ────────────────────────────────
// For numeric/ID values in onclick:  onclick="fn(${JSON.stringify(r.id)})"
// For string values in onclick:      onclick="fn('${escapeHtml(val).replace(/'/g,"&#39;")}')"
// NEVER:                             onclick="fn(${val})"   ← injection risk

// ── RULE 4: Content Security Policy (in <head>) ───────────────────────────
// Already included in the HTML shell above.
// Do NOT add 'unsafe-eval' — never needed with this architecture.

// ── SECURITY CHECKLIST — verify before every release ─────────────────────
/*
  [ ] Every innerHTML assignment: all user values wrapped in escapeHtml()
  [ ] toast() always uses document.createTextNode (never innerHTML) for message
  [ ] Every CSV/Excel export: all cell values wrapped in csvSanitize()
  [ ] onclick attributes with string params: escapeHtml().replace(/'/g,"&#39;")
  [ ] onclick attributes with ID params: JSON.stringify(id)
  [ ] <meta CSP> tag present in <head>
  [ ] IndexedDB onerror shows user-visible message (not just console.error)
*/
```

---

## PART 6 — FEATURE MODULES

---

### F1: Auth Gate

**What it is:** A full-screen overlay that blocks the app until the user signs in. Stores the session for 24 hours so they don't log in every time. Provides a sign-out action in the header or settings.

```javascript
// ── CONFIGURE ─────────────────────────────────────────────────────────────
const AUTH_KEY    = 'YOUR_APP_auth_v1';
const SESSION_TTL = 24 * 60 * 60 * 1000; // 24 hours in ms
// ─────────────────────────────────────────────────────────────────────────

let authUser = null;

// Called during app init — checks cached session first
function checkAuthSession() {
  try {
    const raw  = localStorage.getItem(AUTH_KEY);
    if (!raw) { showAuthGate(); return; }
    const sess = JSON.parse(raw);
    if (Date.now() - sess.ts > SESSION_TTL) { showAuthGate(); return; }
    authUser = sess;
    bootApp();  // session valid — go directly to app
  } catch { showAuthGate(); }
}

function cacheSession(user) {
  authUser = { ...user, ts: Date.now() };
  localStorage.setItem(AUTH_KEY, JSON.stringify(authUser));
}

function showAuthGate() {
  document.getElementById('app').style.visibility      = 'hidden';
  document.getElementById('auth-gate').style.display   = 'flex';
  document.getElementById('bottombar').style.display   = 'none';
}

function bootApp() {
  document.getElementById('auth-gate').style.display   = 'none';
  document.getElementById('app').style.visibility      = 'visible';
  document.getElementById('bottombar').style.display   = '';
  showView(DEFAULT_VIEW);
  updateUserMenuUI();
}

function signOut() {
  localStorage.removeItem(AUTH_KEY);
  authUser = null;
  showAuthGate();
}

// Updates header user avatar / sign-out visibility
function updateUserMenuUI() {
  const menu = document.getElementById('user-menu');
  if (menu) menu.style.display = authUser ? 'block' : 'none';
}
```

**Auth gate HTML:**
```html
<div id="auth-gate" style="display:none;position:fixed;inset:0;background:var(--bg);
  z-index:1000;align-items:center;justify-content:center;flex-direction:column;gap:20px">

  <div style="text-align:center;margin-bottom:12px">
    <div style="font-size:48px;margin-bottom:10px">🔒</div>
    <h1 style="font-size:24px">Welcome to APP NAME</h1>
    <p style="color:var(--muted);margin-top:6px">Sign in to continue</p>
  </div>

  <!-- Option A: Google OAuth -->
  <button class="btn btn-primary" onclick="signInWithGoogle()" style="width:280px;justify-content:center;gap:10px">
    <span>G</span> Continue with Google
  </button>

  <!-- Option B: License key form -->
  <div style="width:280px">
    <div class="form-row">
      <label class="label">Email</label>
      <input id="auth-email" type="email" class="input" placeholder="you@example.com">
    </div>
    <div class="form-row">
      <label class="label">License Key</label>
      <input id="auth-key" type="text" class="input" placeholder="XXXX-XXXX-XXXX">
    </div>
    <div id="auth-error" class="form-error"></div>
    <button class="btn btn-primary" onclick="signInWithKey()" style="width:100%;justify-content:center;margin-top:8px">
      Sign In
    </button>
  </div>
</div>
```

**Sign-out in settings or header user menu:**
```html
<!-- User menu dropdown (appears in header when logged in) -->
<div id="user-dropdown" style="display:none;position:absolute;top:54px;right:12px;
  background:var(--surface);border:1px solid var(--border);border-radius:var(--radius);
  padding:12px;min-width:200px;box-shadow:var(--shadow);z-index:200">
  <div style="font-size:13px;color:var(--muted);padding-bottom:10px;border-bottom:1px solid var(--border)" id="user-email-display"></div>
  <button class="nav-item nav-danger" style="width:100%;margin:8px 0 0" onclick="signOut()">
    <span class="nav-icon">🚪</span> Sign Out
  </button>
</div>
```

---

### F2: Three-Layer Backup

**What it is:** Three overlapping safety nets so data is never lost. Layer 1 is automatic and requires no user action.

```
Layer 1 — Browser Snapshots (automatic, always on)
  Saves a snapshot to localStorage after every data change (3-second debounce).
  Keeps the last 5 snapshots. Restore any snapshot from the Backup view.
  No user setup required.

Layer 2 — Local File (Chrome/Edge/Brave only)
  User links a file on their computer. App auto-saves to it every 30 minutes.
  Also fires when the tab is hidden or closed (before the user leaves).
  Re-link required after page reload (browser security limitation).

Layer 3 — Cloud Sync (requires Google OAuth + Sheets API setup)
  Signs in with Google. Creates or links a Google Sheet.
  Pushes all records on every change. Pulls to sync across devices.
  Optional — skip entirely for local-only apps.
```

**Protection Score Banner:**
```javascript
// Shows in header or backup view — 0/3 to 3/3 layers active
function getProtectionScore() {
  let score = 0;
  if (loadSnapshots().length > 0) score++;  // Layer 1: has snapshots
  if (_autoFileHandle)             score++;  // Layer 2: file linked
  if (authUser && gsheetsSheetId) score++;  // Layer 3: sheet linked
  return score; // 0, 1, 2, or 3
}

function renderProtectionBanner() {
  const score  = getProtectionScore();
  const banner = document.getElementById('protection-banner');
  if (!banner) return;
  const colors = ['var(--danger)', 'var(--warning)', 'var(--warning)', 'var(--success)'];
  const labels = ['No backup active', '1 of 3 layers active', '2 of 3 layers active', 'Fully protected'];
  banner.style.color     = colors[score];
  banner.textContent     = `🛡 ${labels[score]}`;
}
```

**Layer 1 — Browser Snapshots:**
```javascript
const SNAPSHOTS_KEY = 'YOUR_APP_snapshots_v1';
const MAX_SNAPSHOTS = 5;
let   _snapTimer    = null;

// Call this after EVERY data mutation
function scheduleSnapshot() {
  clearTimeout(_snapTimer);
  _snapTimer = setTimeout(takeSnapshot, 3000); // 3s debounce
}

async function takeSnapshot() {
  const all   = await dbGetAll();
  let   snaps = loadSnapshots();
  snaps.unshift({ ts: Date.now(), count: all.length, data: all });
  snaps = snaps.slice(0, MAX_SNAPSHOTS);
  try {
    localStorage.setItem(SNAPSHOTS_KEY, JSON.stringify(snaps));
  } catch (e) {
    if (e.name === 'QuotaExceededError') {
      try { localStorage.setItem(SNAPSHOTS_KEY, JSON.stringify(snaps.slice(0, 2))); } catch {}
      toast('Storage nearly full — oldest snapshots trimmed', 'warning');
    }
  }
}

function loadSnapshots() {
  try { return JSON.parse(localStorage.getItem(SNAPSHOTS_KEY) || '[]'); }
  catch { return []; }
}

async function restoreSnapshot(ts) {
  const snap    = loadSnapshots().find(s => s.ts === ts);
  if (!snap) { toast('Snapshot not found', 'error'); return; }
  const current = await dbCount();
  showConfirm(
    'Restore Snapshot',
    `Replace your current ${current} records with ${snap.count} records from this snapshot?`,
    async () => {
      await dbClear();
      await dbBulkPut(snap.data);
      toast(`Restored ${snap.count} records`, 'success');
      showView(DEFAULT_VIEW);
    }
  );
}

function renderSnapshotList() {
  const snaps = loadSnapshots();
  const el    = document.getElementById('snapshot-list');
  if (!el) return;
  el.innerHTML = snaps.length
    ? snaps.map(s => `
        <div style="display:flex;align-items:center;gap:12px;padding:10px 0;border-bottom:1px solid var(--border)">
          <div style="flex:1">
            <div style="font-size:13px">${new Date(s.ts).toLocaleString()}</div>
            <div style="font-size:12px;color:var(--muted)">${s.count} records</div>
          </div>
          <button class="btn btn-sm btn-outline" onclick="restoreSnapshot(${JSON.stringify(s.ts)})">Restore</button>
          <button class="btn btn-sm btn-ghost"   onclick="downloadSnapshot(${JSON.stringify(s.ts)})">↓</button>
        </div>`).join('')
    : `<p style="color:var(--muted);font-size:13px">No snapshots yet — add a record to create one.</p>`;
}
```

**Layer 2 — Local File:**
```javascript
let   _autoFileHandle  = null;
let   _autoBackupTimer = null;
const L2_META_KEY       = 'YOUR_APP_l2_v1';

function isFileSystemSupported() { return typeof window.showSaveFilePicker === 'function'; }

async function linkBackupFile() {
  if (!isFileSystemSupported()) {
    toast('File backup requires Chrome, Edge, or Brave', 'warning'); return;
  }
  try {
    _autoFileHandle = await window.showSaveFilePicker({
      suggestedName: 'YOUR_APP_backup.json',
      types: [{ description: 'JSON backup', accept: { 'application/json': ['.json'] } }]
    });
    clearInterval(_autoBackupTimer);
    _autoBackupTimer = setInterval(runFileBackup, 30 * 60 * 1000);
    await runFileBackup();
    localStorage.setItem(L2_META_KEY, JSON.stringify({ fileName: _autoFileHandle.name }));
    toast(`Backup linked to ${_autoFileHandle.name}`, 'success');
    renderProtectionBanner();
  } catch (e) {
    if (e.name !== 'AbortError') toast('Could not link file', 'error'); // AbortError = user cancelled
  }
}

async function runFileBackup() {
  if (!_autoFileHandle) return;
  try {
    const all  = await dbGetAll();
    const json = JSON.stringify({ settings, records: all, backedUpAt: Date.now() }, null, 2);
    const w    = await _autoFileHandle.createWritable();
    await w.write(json); await w.close();
    setSaveStatus('saved');
  } catch (e) {
    toast('File backup failed: ' + e.message, 'error');
  }
}

// Fire immediately when user hides tab (before they close it)
document.addEventListener('visibilitychange', () => {
  if (document.hidden && _autoFileHandle) runFileBackup();
});
```

---

### F3: Search Bar

**What it is:** A single text input that filters any table. Debounced so it doesn't fire on every keystroke.

```javascript
let _searchTimer = null;

function onSearch(value, renderFn) {
  clearTimeout(_searchTimer);
  _searchTimer = setTimeout(() => {
    tableState.search = value;
    tableState.page   = 1; // always reset to page 1 on new search
    renderFn();
  }, 250);
}
```

**HTML (place in view header or topbar):**
```html
<div class="search-wrap">
  <span class="search-icon">🔍</span>
  <input id="search-input" type="search" class="input search-input"
         placeholder="Search…" oninput="onSearch(this.value, renderTable)"
         autocomplete="off" aria-label="Search records">
  <button class="search-clear" onclick="clearSearch()" id="search-clear-btn" style="display:none">✕</button>
</div>
```

**CSS:**
```css
.search-wrap  { position: relative; display: flex; align-items: center; max-width: 320px; flex: 1; }
.search-icon  { position: absolute; left: 10px; color: var(--muted); pointer-events: none; font-size: 14px; }
.search-input { padding-left: 32px; padding-right: 28px; }
.search-clear { position: absolute; right: 8px; background: none; border: none;
                color: var(--muted); cursor: pointer; font-size: 14px; padding: 2px; }
.search-clear:hover { color: var(--text); }
```

---

### F4: Help System

**What it is:** A slide-in panel from the right side showing FAQs and keyboard shortcuts. Press `?` or click the `❓` button to open it.

```javascript
// ── CONFIGURE: define your FAQ content ───────────────────────────────────
const HELP_FAQS = [
  {
    question: 'How do I add a record?',
    answer:   'Click the ➕ button in the sidebar or press the N key.'
  },
  {
    question: 'Will my data be lost if I close the browser?',
    answer:   'No. All data is saved in your browser automatically. Enable the File Backup in the Backup section for extra protection.'
  },
  {
    question: 'How do I import data from a spreadsheet?',
    answer:   'Go to Import, upload your CSV file, then map the columns to the right fields.'
  },
  // Add your own FAQs here
];

const HELP_SHORTCUTS = [
  { key: 'N',       action: 'New record' },
  { key: '/',       action: 'Focus search' },
  { key: '?',       action: 'Open this help panel' },
  { key: 'Escape',  action: 'Close any open dialog' },
  { key: 'G then H', action: 'Go to Home' },
  // Add your shortcuts here
];
// ─────────────────────────────────────────────────────────────────────────

function openHelp() {
  const drawer  = document.getElementById('help-drawer');
  const content = document.getElementById('help-content');

  content.innerHTML = `
    <div style="padding:0 20px 20px">
      <h3 style="font-size:15px;margin-bottom:14px">Frequently Asked Questions</h3>
      <div id="faq-list">
        ${HELP_FAQS.map((f, i) => `
          <div class="faq-item" style="border-bottom:1px solid var(--border)">
            <button class="faq-q" onclick="toggleFAQ(${i})" style="width:100%;text-align:left;
              padding:12px 0;background:none;border:none;color:var(--text);font-size:14px;
              display:flex;justify-content:space-between;align-items:center;cursor:pointer">
              ${escapeHtml(f.question)}
              <span id="faq-arrow-${i}" style="color:var(--muted)">›</span>
            </button>
            <div id="faq-body-${i}" style="display:none;padding:0 0 12px;font-size:13px;color:var(--muted);line-height:1.6">
              ${escapeHtml(f.answer)}
            </div>
          </div>`).join('')}
      </div>

      <h3 style="font-size:15px;margin:24px 0 12px">Keyboard Shortcuts</h3>
      <div style="display:flex;flex-direction:column;gap:6px">
        ${HELP_SHORTCUTS.map(s => `
          <div style="display:flex;justify-content:space-between;font-size:13px;padding:4px 0">
            <span style="color:var(--muted)">${escapeHtml(s.action)}</span>
            <kbd style="background:var(--surface2);border:1px solid var(--border);
              border-radius:4px;padding:1px 7px;font-family:monospace;font-size:12px">${escapeHtml(s.key)}</kbd>
          </div>`).join('')}
      </div>
    </div>`;

  drawer.style.display = 'block';
}

function closeHelp() {
  document.getElementById('help-drawer').style.display = 'none';
}

function toggleFAQ(i) {
  const body  = document.getElementById(`faq-body-${i}`);
  const arrow = document.getElementById(`faq-arrow-${i}`);
  const open  = body.style.display === 'none';
  body.style.display  = open ? 'block' : 'none';
  arrow.textContent   = open ? '∨' : '›';
}
```

**Help Drawer CSS:**
```css
#help-drawer   { position: fixed; inset: 0; z-index: 400; display: none; }
#help-backdrop { position: absolute; inset: 0; background: rgba(0,0,0,.5); }
#help-panel    { position: absolute; right: 0; top: 0; bottom: 0;
                 width: min(400px, 95vw); background: var(--surface);
                 border-left: 1px solid var(--border); overflow-y: auto;
                 animation: slideInRight .2s ease; }
.help-header   { display: flex; align-items: center; justify-content: space-between;
                 padding: 18px 20px; border-bottom: 1px solid var(--border);
                 position: sticky; top: 0; background: var(--surface); z-index: 1; }
@keyframes slideInRight { from { transform: translateX(100%); } to { transform: translateX(0); } }
```

---

### F5: Navigation System

```javascript
// ── CONFIGURE: define your app's views ───────────────────────────────────
const DEFAULT_VIEW = 'YOUR_DEFAULT_VIEW';  // first view shown after login

// Map each view ID to the function that renders it
const VIEW_RENDERERS = {
  // 'view-id': renderFunction
  // Example:
  // 'home-view':     renderHome,
  // 'list-view':     () => renderTable({ page: 1 }),
  // 'add-view':      initAddForm,
  // 'backup-view':   renderBackupView,
  // 'settings-view': loadSettingsForm,
};

// Desktop sidebar nav structure
// Each group has a label + array of items
const NAV_GROUPS = [
  {
    label: 'SECTION NAME',   // e.g. 'Overview', 'Data', 'Settings'
    items: [
      // { view: 'view-id', icon: '🏠', label: 'Home' },
    ]
  },
  // Add more groups
];

// Mobile bottom tab bar (max 5 items; use 'fab:true' for the center + button)
const MOBILE_TABS = [
  // { view: 'view-id', icon: '🏠', label: 'Home' },
  // { view: 'add-view', icon: '+', label: '', fab: true },
];
// ─────────────────────────────────────────────────────────────────────────

function buildNav() {
  // Build sidebar
  const sidebar = document.getElementById('sidebar');
  const navHtml = NAV_GROUPS.map(group => `
    <div class="nav-section">
      <div class="nav-label">${escapeHtml(group.label)}</div>
      ${group.items.map(item => `
        <div class="nav-item" data-view="${item.view}" onclick="showView('${item.view}')" role="button" tabindex="0">
          <span class="nav-icon">${item.icon}</span>
          <span>${escapeHtml(item.label)}</span>
          ${item.badge ? `<span class="nav-badge" id="nav-badge-${item.view}"></span>` : ''}
        </div>`).join('')}
    </div>`).join('');

  // Find existing nav HTML and append (preserve sidebar-info section)
  const infoEl = document.getElementById('sidebar-info');
  sidebar.innerHTML = (infoEl ? infoEl.outerHTML : '') + navHtml;

  // Build bottom tab bar
  const bar = document.getElementById('bottombar');
  bar.innerHTML = MOBILE_TABS.map(tab => `
    <button data-view="${tab.view || ''}" onclick="${tab.view ? `showView('${tab.view}')` : tab.onclick || ''}"
            class="${tab.fab ? 'tab-fab' : ''}" ${tab.view ? '' : ''}>
      <span class="tab-icon">${tab.icon}</span>
      ${tab.label ? `<span>${escapeHtml(tab.label)}</span>` : ''}
    </button>`).join('');
}

function showView(viewId) {
  // Hide all views
  document.querySelectorAll('.view').forEach(v => {
    v.style.display = 'none';
    v.classList.remove('active');
  });

  // Show target
  const target = document.getElementById(viewId);
  if (!target) { console.warn('showView: no element with id', viewId); return; }
  target.style.display = 'block';
  target.classList.add('active');

  // Update sidebar active state
  document.querySelectorAll('.nav-item[data-view]').forEach(el =>
    el.classList.toggle('active', el.dataset.view === viewId));

  // Update mobile tab bar active state
  document.querySelectorAll('#bottombar button[data-view]').forEach(el =>
    el.classList.toggle('active', el.dataset.view === viewId));

  // Scroll main content to top
  const main = document.getElementById('main');
  if (main) main.scrollTop = 0;

  // Run the view's render function
  const renderer = VIEW_RENDERERS[viewId];
  if (renderer) renderer();
}

// Mobile sidebar
function toggleSidebar() {
  const sidebar  = document.getElementById('sidebar');
  const overlay  = document.getElementById('sidebar-overlay');
  const isOpen   = sidebar.classList.contains('open');
  sidebar.classList.toggle('open', !isOpen);
  overlay.style.display = isOpen ? 'none' : 'block';
}

function closeSidebar() {
  document.getElementById('sidebar').classList.remove('open');
  document.getElementById('sidebar-overlay').style.display = 'none';
}
```

---

### F6: Quick-Entry Form

The form UI pattern for adding a single record fast. Domain-agnostic skeleton:

```javascript
// ── CONFIGURE: map HTML input IDs to your record fields ──────────────────
const FORM_CONFIG = {
  fields: [
    // { id: 'input-element-id', key: 'record-field-name', required: true, transform: fn }
    // Examples:
    // { id: 'f-name',   key: 'name',   required: true,  transform: v => v.trim()            },
    // { id: 'f-date',   key: 'date',   required: true,  transform: v => v                   },
    // { id: 'f-tag',    key: 'category', required: false, transform: v => v.trim() || 'None' },
    // { id: 'f-amount', key: 'amount', required: true,  transform: v => Math.abs(parseFloat(v)) },
    // { id: 'f-notes',  key: 'notes',  required: false, transform: v => v.trim()            },
  ],
  validators: [
    // fn receives the transformed record object, returns error string or null
    // Example: r => (!r.name) ? 'Name is required' : null,
    // Example: r => (!r.amount || r.amount <= 0) ? 'Amount must be positive' : null,
  ],
  onSuccess: null, // function to call after successful save (e.g. renderTable)
  keepAfterSubmit: ['f-date', 'f-tag'], // these inputs keep their value (rapid-entry UX)
  focusAfterSubmit: 'f-name',           // which input to focus after submit
};
// ─────────────────────────────────────────────────────────────────────────

function initAddForm() {
  // Default date to today
  const dateInput = document.querySelector('#add-view [type="date"]');
  if (dateInput && !dateInput.value) {
    dateInput.value = new Date().toISOString().slice(0, 10);
  }
  // Clear error
  const errEl = document.getElementById('form-error');
  if (errEl) errEl.textContent = '';
  // Update autocomplete list
  updateTagDatalist();
  renderRecentEntries();
}

async function submitForm(e) {
  if (e) e.preventDefault();

  const errEl = document.getElementById('form-error');
  if (errEl) errEl.textContent = '';

  // Read + transform all fields
  const raw = {};
  FORM_CONFIG.fields.forEach(f => {
    const el = document.getElementById(f.id);
    raw[f.key] = el ? f.transform(el.value) : (f.required ? '' : undefined);
  });

  // Validate
  for (const validator of FORM_CONFIG.validators) {
    const err = validator(raw);
    if (err) { if (errEl) errEl.textContent = err; return; }
  }

  // Build full record
  const record = { id: newId(), ...raw, created_at: new Date().toISOString() };

  await dbPutWithStatus(record);

  // Reset fields (keep some for rapid entry)
  FORM_CONFIG.fields.forEach(f => {
    if (!FORM_CONFIG.keepAfterSubmit.includes(f.id)) {
      const el = document.getElementById(f.id);
      if (el) el.value = '';
    }
  });

  // Focus the first input for rapid re-entry
  document.getElementById(FORM_CONFIG.focusAfterSubmit)?.focus();

  toast('Record added', 'success');
  renderRecentEntries();
  if (FORM_CONFIG.onSuccess) FORM_CONFIG.onSuccess();
}
```

---

### F7: Paginated Data Table

**Shared pagination renderer** (use for every table in the app — one function, zero duplication):

```javascript
function renderPagination(containerId, currentPage, total, pageSize, onPageChange) {
  const totalPages = Math.ceil(total / pageSize);
  const el = document.getElementById(containerId);
  if (!el || totalPages <= 1) { if (el) el.innerHTML = ''; return; }

  const start = (currentPage - 1) * pageSize + 1;
  const end   = Math.min(currentPage * pageSize, total);

  // Page number array with gaps for large page counts
  const nums = [];
  for (let i = 1; i <= totalPages; i++) {
    if (i === 1 || i === totalPages || (i >= currentPage - 2 && i <= currentPage + 2)) nums.push(i);
  }
  const withGaps = [];
  nums.forEach((p, i) => { if (i > 0 && p - nums[i-1] > 1) withGaps.push('…'); withGaps.push(p); });

  el.innerHTML = `
    <span class="pg-info">Showing ${start}–${end} of ${total}</span>
    <button onclick="(${onPageChange})(1)"                ${currentPage===1?'disabled':''}>«</button>
    <button onclick="(${onPageChange})(${currentPage-1})"  ${currentPage===1?'disabled':''}>‹</button>
    ${withGaps.map(p => p === '…'
      ? `<span style="padding:0 4px;color:var(--muted)">…</span>`
      : `<button class="${p===currentPage?'active':''}" onclick="(${onPageChange})(${p})">${p}</button>`
    ).join('')}
    <button onclick="(${onPageChange})(${currentPage+1})"  ${currentPage===totalPages?'disabled':''}>›</button>
    <button onclick="(${onPageChange})(${totalPages})"      ${currentPage===totalPages?'disabled':''}>»</button>`;
}
```

**Pagination CSS:**
```css
.pagination             { display: flex; align-items: center; gap: 4px; padding: 14px 0; flex-wrap: wrap; }
.pagination button      { padding: 5px 10px; border-radius: 5px; border: 1px solid var(--border);
                          background: var(--surface2); color: var(--text); font-size: 13px; }
.pagination button.active  { background: var(--primary); color: #000; border-color: var(--primary); }
.pagination button:disabled { opacity: .4; cursor: not-allowed; }
.pg-info                { font-size: 13px; color: var(--muted); margin-right: 8px; }
```

---

### F8: CSV Import

See the full implementation in Part 2 — Component C07 of this document. The RFC 4180 parser, Web Worker, and field mapping modal are domain-agnostic. Only the `IMPORT_FIELDS` config array changes per app.

---

### F9: Tag / Label Management

See Part 2 — Component C08. Only the `DEFAULT_TAGS` array and the field name used to store the tag (`r.category` in the P&L example — change to `r.tag`, `r.label`, `r.type`, etc.) change per app.

---

### F10: Data Export

```javascript
// Export records as CSV — configure CSV_COLUMNS for your fields
async function exportCSV() {
  const records = await dbGetAll();
  if (!records.length) { toast('Nothing to export', 'info'); return; }

  // ── CONFIGURE ────────────────────────────────────────────────────────────
  const CSV_COLUMNS = [
    // { header: 'Column Header', value: record => record.fieldName }
    // Example:
    // { header: 'Name',        value: r => r.name        || '' },
    // { header: 'Date',        value: r => r.date        || '' },
    // { header: 'Category',    value: r => r.category    || '' },
    // { header: 'Description', value: r => r.description || '' },
  ];
  // ─────────────────────────────────────────────────────────────────────────

  const lines = [
    CSV_COLUMNS.map(c => `"${c.header}"`).join(','),
    ...records.map(r =>
      CSV_COLUMNS.map(c => `"${String(csvSanitize(c.value(r))).replace(/"/g, '""')}"`).join(',')
    )
  ];
  downloadFile(lines.join('\n'), `export_${Date.now()}.csv`, 'text/csv;charset=utf-8;');
  toast('CSV downloaded', 'success');
}

async function exportJSON() {
  const records = await dbGetAll();
  const payload = JSON.stringify({ settings, records, exportedAt: new Date().toISOString() }, null, 2);
  downloadFile(payload, `backup_${Date.now()}.json`, 'application/json');
  toast('JSON backup downloaded', 'success');
}

async function importJSONBackup(file) {
  try {
    const data = JSON.parse(await file.text());
    if (!Array.isArray(data.records)) { toast('Invalid backup file', 'error'); return; }
    showConfirm(
      'Import Backup',
      `Add ${data.records.length} records? Existing records with the same ID will be skipped.`,
      async () => {
        const existing  = new Set((await dbGetAll()).map(r => r.id));
        const toImport  = data.records.filter(r => !existing.has(r.id));
        await dbBulkPut(toImport);
        scheduleSnapshot();
        toast(`Imported ${toImport.length} new records`, 'success');
        showView(DEFAULT_VIEW);
      }
    );
  } catch { toast('Could not read backup file', 'error'); }
}

function downloadFile(content, filename, type) {
  const blob = new Blob([content], { type });
  const url  = URL.createObjectURL(blob);
  const a    = Object.assign(document.createElement('a'), { href: url, download: filename });
  document.body.appendChild(a);
  a.click();
  document.body.removeChild(a);
  URL.revokeObjectURL(url);
}
```

---

### F11: Keyboard Shortcuts

```javascript
// ── CONFIGURE ─────────────────────────────────────────────────────────────
const SHORTCUTS = {
  // key (lowercase) → function
  'n': () => showView(ADD_VIEW_ID),
  '/': () => document.getElementById('search-input')?.focus(),
  '?': openHelp,
  // Add your own single-key shortcuts
};

// Two-key shortcuts — press 'g' then a letter
const G_SHORTCUTS = {
  // 'h': () => showView('home-view'),
  // 'l': () => showView('list-view'),
  // 's': () => showView('settings-view'),
  // 'b': () => showView('backup-view'),
  // Add your view shortcuts
};
// ─────────────────────────────────────────────────────────────────────────

let _gKeyActive = false;
let _gTimer     = null;

document.addEventListener('keydown', e => {
  // Skip when typing in any input, textarea, or select
  if (['INPUT','TEXTAREA','SELECT'].includes(e.target.tagName)) return;
  if (e.metaKey || e.ctrlKey || e.altKey) return;

  const key = e.key.toLowerCase();

  if (e.key === 'Escape') { closeAllModals(); return; }

  if (_gKeyActive) {
    _gKeyActive = false;
    clearTimeout(_gTimer);
    if (G_SHORTCUTS[key]) G_SHORTCUTS[key]();
    return;
  }

  if (key === 'g') {
    _gKeyActive = true;
    _gTimer = setTimeout(() => { _gKeyActive = false; }, 1500); // reset g-prefix after 1.5s
    return;
  }

  if (SHORTCUTS[key]) SHORTCUTS[key]();
});
```

---

### F12: PWA / Offline Support

**manifest.json:**
```json
{
  "name":             "YOUR APP NAME",
  "short_name":       "APP",
  "start_url":        "/",
  "display":          "standalone",
  "orientation":      "any",
  "theme_color":      "#0f1117",
  "background_color": "#0f1117",
  "description":      "ONE LINE DESCRIPTION OF YOUR APP",
  "categories":       ["productivity", "utilities"],
  "icons": [
    { "src": "icon-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any maskable" },
    { "src": "icon-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any maskable" }
  ],
  "shortcuts": [
    { "name": "Add Record", "url": "/?action=add",  "description": "Quickly add a new record" },
    { "name": "View List",  "url": "/?action=list", "description": "Open the full list" }
  ]
}
```

**service-worker.js:**
```javascript
const CACHE_NAME  = 'YOUR_APP_v1';
const CORE_ASSETS = ['/', '/index.html', '/manifest.json', '/icon-192.png', '/icon-512.png'];

self.addEventListener('install',  e => {
  e.waitUntil(caches.open(CACHE_NAME).then(c => c.addAll(CORE_ASSETS)));
  self.skipWaiting();
});

self.addEventListener('activate', e => {
  e.waitUntil(
    caches.keys().then(keys =>
      Promise.all(keys.filter(k => k !== CACHE_NAME).map(k => caches.delete(k)))
    )
  );
  self.clients.claim();
});

self.addEventListener('fetch', e => {
  // Pass through all API calls — never cache third-party network requests
  if (/supabase\.co|googleapis\.com|accounts\.google\.com/.test(e.request.url)) return;

  e.respondWith(
    caches.match(e.request).then(cached =>
      cached || fetch(e.request).then(res => {
        if (res.ok) caches.open(CACHE_NAME).then(c => c.put(e.request, res.clone()));
        return res;
      })
    )
  );
});
```

**Registration + URL shortcut handling:**
```javascript
// Register service worker
if ('serviceWorker' in navigator) {
  window.addEventListener('load', () =>
    navigator.serviceWorker.register('/service-worker.js')
      .catch(err => console.warn('SW not registered:', err))
  );
}

// Handle PWA shortcut URLs (?action=add, ?action=list, etc.)
// Call this inside initApp() after bootApp()
function handleURLActions() {
  const action = new URLSearchParams(location.search).get('action');
  if (action === 'add')  showView(ADD_VIEW_ID);
  if (action === 'list') showView(LIST_VIEW_ID);
  // Add your own action handlers here
}
```

---

## PART 7 — OPTIONAL DATA DISPLAY (D1, D2, D3)

> These three components are **optional**. Many apps don't need charts or aggregated summaries.
> The design and data are entirely defined by you — this section provides the proven structural pattern, not P&L-specific metrics.

---

### D1: KPI Summary Cards

**What it is:** A grid of metric boxes. You define what each card measures and how to format it.

```javascript
// ── YOU DEFINE THIS — what metrics your app shows ─────────────────────────
// Each entry: { id, label, compute(records[]) → value, format(value) → string, color(value) → CSSvar }
const MY_KPI_METRICS = [
  // Example for a task app:
  // { id:'kpi-total',   label:'Total',     compute: r => r.length,                               format: v=>v,  color:()=>'var(--text)' },
  // { id:'kpi-open',    label:'Open',      compute: r => r.filter(t=>t.status==='open').length,   format: v=>v,  color: v=>v>0?'var(--warning)':'var(--muted)' },
  // { id:'kpi-done',    label:'Complete',  compute: r => r.filter(t=>t.status==='done').length,   format: v=>v,  color: v=>v>0?'var(--success)':'var(--muted)' },
];
// ─────────────────────────────────────────────────────────────────────────

async function renderKPIs() {
  const records = await dbGetAll();
  MY_KPI_METRICS.forEach(m => {
    const el = document.getElementById(m.id);
    if (!el) return;
    const val = m.compute(records);
    el.textContent  = m.format(val);
    el.style.color  = m.color(val);
  });
}
```

**HTML template (repeat once per metric, or generate from MY_KPI_METRICS):**
```html
<div class="kpi-grid">
  <div class="card">
    <div class="label">YOUR LABEL</div>
    <div class="kpi-value" id="kpi-YOUR_ID">—</div>
  </div>
</div>
```

**CSS:**
```css
.kpi-grid  { display: grid; grid-template-columns: repeat(auto-fit, minmax(160px, 1fr)); gap: 16px; margin-bottom: 24px; }
.kpi-value { font-size: 28px; font-weight: 700; margin-top: 6px; }
```

---

### D2: Charts (Optional)

**What it is:** Chart.js-powered charts. You define the datasets. The structural boilerplate (destroy-before-recreate, theme colors, responsive container) is what you reuse — not the chart content.

**Required one-time setup:**
- Inline Chart.js 4.4.1 in your HTML (do not load from CDN — breaks offline)
- One canvas element per chart

**Structural boilerplate (copy verbatim, use for every chart):**
```javascript
const charts = {}; // track all Chart.js instances

function themeColors() {
  const s = getComputedStyle(document.documentElement);
  return {
    text:    s.getPropertyValue('--text').trim(),
    muted:   s.getPropertyValue('--muted').trim(),
    border:  s.getPropertyValue('--border').trim(),
    primary: s.getPropertyValue('--primary').trim(),
    success: s.getPropertyValue('--success').trim(),
    danger:  s.getPropertyValue('--danger').trim(),
    warning: s.getPropertyValue('--warning').trim(),
  };
}

// ALWAYS call before new Chart() — never skip
function destroyChart(key) {
  if (charts[key]) { try { charts[key].destroy(); } catch(e){} delete charts[key]; }
}

// Base options shared by bar / line charts — adjust as needed
function baseChartOptions(tc, tooltipFormat) {
  return {
    responsive: true, maintainAspectRatio: false,
    plugins: {
      legend: { labels: { color: tc.text, font: { size: 12 } } },
      tooltip: { callbacks: { label: tooltipFormat || (ctx => ctx.parsed.y) } },
    },
    scales: {
      x: { ticks: { color: tc.muted }, grid: { color: tc.border + '44' } },
      y: { ticks: { color: tc.muted }, grid: { color: tc.border + '44' } },
    }
  };
}

// Re-render all charts on theme toggle
function retintCharts() {
  // Destroy and re-call each render function — simpler than patching existing instances
  Object.keys(charts).forEach(k => destroyChart(k));
  renderAllCharts(); // YOUR function that calls each individual renderXChart()
}
```

**Canvas HTML template:**
```html
<div class="card">
  <div class="card-header">YOUR CHART TITLE</div>
  <!-- chart-wrap must have an explicit height for Chart.js responsive sizing -->
  <div style="position:relative;height:260px">
    <canvas id="chart-YOUR_KEY"></canvas>
  </div>
</div>
```

---

### D3: Summary / Aggregated Table (Optional)

**What it is:** A read-only table that groups records by a field and shows aggregated totals. Not paginated — the number of groups is bounded.

```javascript
// ── YOU DEFINE THIS ───────────────────────────────────────────────────────
// GROUP_BY_FIELD: the field to group records by (e.g. 'status', 'category', 'month')
// AGGREGATE_FNS:  what to compute per group
// ─────────────────────────────────────────────────────────────────────────

async function renderSummaryTable(groupByField, aggregateFns, tbodyId) {
  const records = await dbGetAll();

  // Group records
  const groups = {};
  records.forEach(r => {
    const key = r[groupByField] || 'Unknown';
    if (!groups[key]) groups[key] = [];
    groups[key].push(r);
  });

  const tbody = document.getElementById(tbodyId);
  if (!tbody) return;

  const keys = Object.keys(groups).sort();
  if (!keys.length) {
    tbody.innerHTML = `<tr><td colspan="${aggregateFns.length + 1}" class="empty-state">No data yet</td></tr>`;
    return;
  }

  tbody.innerHTML = keys.map(key =>
    `<tr>
      <td>${escapeHtml(key)}</td>
      ${aggregateFns.map(fn => `<td>${fn.format(fn.compute(groups[key]))}</td>`).join('')}
    </tr>`
  ).join('');
}

// Example usage — task app grouping by status:
// renderSummaryTable('status', [
//   { compute: rows => rows.length,                              format: v => v },
//   { compute: rows => rows.filter(r=>r.priority==='high').length, format: v => v + ' high-priority' },
// ], 'summary-tbody');
```

---

## PART 8 — APP INITIALIZATION

```javascript
// Exact order matters — changing the order causes blank screens or missing data
async function initApp() {
  loadSettings();          // 1. Load settings first (other functions depend on settings values)
  loadTags();              // 2. Load tags/labels from localStorage
  applyTheme();            // 3. Apply theme BEFORE rendering anything (prevents flash)
  await initDB();          // 4. Open IndexedDB — must succeed before any data read/write
  buildNav();              // 5. Build sidebar + bottom tab bar from config
  checkAuthSession();      // 6. Auth: show gate or boot app (boot calls showView)
  //  showView(DEFAULT_VIEW);  // ← skip this line if using auth; checkAuthSession calls bootApp which calls showView
  handleURLActions();      // 7. Handle PWA shortcut URLs
}

window.addEventListener('DOMContentLoaded', initApp);
```

---

## PART 9 — AUDIT CHECKLIST

Run this checklist before releasing any new app built from this blueprint.

### Foundation Checks
```
[ ] App loads in Chrome — no console errors
[ ] App loads in Firefox — no console errors
[ ] App loads in Safari — no console errors
[ ] Reload after adding a record → record persists (IndexedDB works)
[ ] Open in private/incognito → shows storage error UI (not a blank page)
[ ] escapeHtml('<script>alert(1)</script>') returns '&lt;script&gt;alert(1)&lt;/script&gt;'
[ ] csvSanitize('=SUM(A1)') returns "'=SUM(A1)"
[ ] newId() called 1000 times → 1000 unique values (check with new Set)
[ ] Settings save and reload after page refresh
[ ] Theme preference (dark/light) persists after page refresh
```

### Design System
```
[ ] Both dark and light themes render correctly (no invisible text)
[ ] All buttons have a visible focus ring when tabbed to (accessibility)
[ ] All icon buttons are at least 32×32px (minimum tap target)
[ ] Mobile layout (375px viewport): bottom tab bar visible, no content cut off
[ ] Tablet layout (768px): sidebar visible, content readable
[ ] Desktop layout (1440px): content doesn't stretch awkwardly wide
[ ] Toast messages appear above bottom tab bar on mobile
[ ] All modals close when clicking outside them
[ ] All modals close when pressing Escape
```

### Auto-Save Indicator (I3)
```
[ ] Add a record → "Saving…" dot appears in header
[ ] After save completes → dot turns green "All saved"
[ ] After 3 seconds → dot disappears
[ ] On save error → red dot + error toast shown
```

### Auth Gate (F1)
```
[ ] App blocked before login — no data accessible
[ ] Reload after login → stays logged in (session cache)
[ ] Session expires after 24h → redirected to login
[ ] Sign-out → immediately shows auth gate
[ ] Sign-out → cached session cleared (sign back in required)
```

### Three-Layer Backup (F2)
```
Layer 1:
[ ] Add record → snapshot auto-created after ~3 seconds
[ ] Maximum 5 snapshots stored (older ones discarded)
[ ] Restore snapshot → confirmation modal shown with record counts
[ ] Restore → database replaced with snapshot records
[ ] Fill localStorage to capacity → app doesn't crash (graceful trim)

Layer 2:
[ ] Firefox: shows "requires Chrome/Edge/Brave" — not a JS error
[ ] Chrome: link file dialog opens
[ ] Cancel file dialog → no error toast
[ ] Link file → file created immediately, backup timer starts
[ ] Hide tab → backup fires before tab goes background
[ ] Unlink → timer stopped, file no longer updated
[ ] Reload → file handle gone; UI shows "previously linked: [name]"

Layer 3 (if implemented):
[ ] Google OAuth flow completes without errors
[ ] Records push to Google Sheet after linking
[ ] Pull from sheet merges records without duplicating
```

### Search (F3)
```
[ ] Search shows only matching records
[ ] Search resets table to page 1
[ ] Clear search → all records return
[ ] Search "xyz" with no matches → empty state shown (not an error)
[ ] Search debounced: typing quickly doesn't fire for every character
```

### Help System (F4)
```
[ ] ❓ button opens help drawer
[ ] Press ? key opens help drawer
[ ] Click backdrop closes help drawer
[ ] All FAQ items expand and collapse
[ ] Keyboard shortcut list visible in help drawer
[ ] Help drawer slides in from right (not a jarring jump)
```

### Navigation (F5)
```
[ ] Clicking each nav item shows the correct view
[ ] Active nav item highlighted in sidebar
[ ] Mobile: active tab highlighted in bottom bar
[ ] Mobile: sidebar opens and closes with hamburger button
[ ] Mobile: tapping backdrop closes mobile sidebar
[ ] View's render function fires when navigating to it
[ ] Navigating to a view scrolls main content back to top
[ ] PWA shortcut URL (?action=add) navigates to correct view
```

### Quick-Entry Form (F6)
```
[ ] Date defaults to today
[ ] Submit with missing required field → error shown (not a crash)
[ ] Submit valid data → success toast, correct fields cleared
[ ] Submit valid data → record in "recently added" list
[ ] Rapid entry: add 10 records without any ID collisions
[ ] Autocomplete list populated with existing tag values
```

### Data Table (F7)
```
[ ] Correct records shown (matching any active filters)
[ ] Search filters records, resets to page 1
[ ] Pagination "Showing X–Y of Z" is accurate
[ ] Prev/Next/First/Last buttons navigate correctly
[ ] Prev button disabled on page 1; Next button disabled on last page
[ ] Sort ascending then descending works on each sortable column
[ ] Sort persists when navigating between pages
[ ] Bulk select: selecting rows shows bulk action bar
[ ] Bulk delete: records removed from DB (not just hidden)
[ ] Undo toast appears 6 seconds after bulk delete
[ ] Undo restores all deleted records
[ ] XSS test: add record with name "<script>alert(1)</script>" → renders as text
```

### CSV Import (F8)
```
[ ] Upload 10-row CSV → mapping modal opens with preview
[ ] Auto-guess populates column selectors for obvious headers
[ ] Cancel → nothing imported
[ ] Import → records appear in table
[ ] Import same file twice → no duplicates (dedup by ID works)
[ ] CSV field with comma inside quotes parses correctly
[ ] Cell starting with "=" exported with leading quote
[ ] File > 500KB → UI doesn't freeze (Web Worker used)
[ ] File with only a header row → "no valid rows" error
```

### Data Export (F10)
```
[ ] Export CSV → file downloads
[ ] Open CSV in Excel → data is correct
[ ] Cell with "=SUM(A1)" → exported as "'=SUM(A1)"
[ ] Export JSON → valid JSON, parseable
[ ] Import JSON backup → records added without duplicating existing IDs
```

### Keyboard Shortcuts (F11)
```
[ ] Each configured shortcut works
[ ] Shortcuts do NOT fire when user is typing in an input field
[ ] G + H (or your configured g-shortcuts) navigate to correct view
[ ] Escape closes open modals and the help drawer
```

### PWA (F12)
```
[ ] manifest.json loads without 404
[ ] App installs in Chrome (install icon appears in address bar)
[ ] Installed app opens in standalone mode (no browser chrome)
[ ] Disconnect internet → reload → app still loads from cache
[ ] Add record while offline → saves to IndexedDB (no server needed)
```

### Security Regression Tests
```
Run each — window.__xss must remain undefined after each test:

[ ] Add record with name:        <img src=x onerror="window.__xss=1">
    → Check table renders it as escaped text, no alert/xss fires

[ ] Add record with category:    <script>window.__xss=1</script>
    → Check tag list renders it as escaped text

[ ] Upload CSV with header:      <img src=x onerror="window.__xss=1">
    → Check mapping modal preview renders escaped

[ ] Export CSV with description: =SUM(A1:A10)
    → Open in Excel → cell shows text, no formula executes

[ ] Auth gate: email input:      '; DROP TABLE records; --
    → Stored and displayed safely (no SQL — just verify no crash)
```

### End-to-End Smoke Test
```
 1. Clear all browser data, open fresh
 2. Verify empty state / auth gate
 3. Sign in (or skip if no auth)
 4. Go to Settings → save app name → verify it appears in header
 5. Go to Add → add 3 records
 6. Go to list view → verify 3 records shown
 7. Search for one record → only that record shown
 8. Clear search → all 3 return
 9. Sort by a column → order changes
10. Select 2 records → bulk bar appears → delete → Undo → both restored
11. Go to Import → upload 5-row CSV → map → import → 5 records added
12. Export CSV → download, open in spreadsheet, data is correct
13. Toggle theme → switch works, no data lost
14. Go to Backup → snapshot exists → restore one → data replaced
15. Reload page → all data and settings still present
16. Mobile (375px viewport) → bottom bar visible → all tabs navigate correctly
17. Help drawer opens → FAQ works → shortcuts listed → closes on backdrop click

PASS: All 17 steps complete with no errors and no console warnings.
```

---

## PART 10 — COMMON BUGS TABLE

| Bug | Root Cause | Prevention |
|-----|-----------|-----------|
| XSS alert fires in a table cell | `innerHTML` used without `escapeHtml()` | Every user string in every `render()` function must call `escapeHtml()` |
| XSS alert fires in toast | `innerHTML` used for toast message | Use `document.createTextNode(msg)` — never innerHTML in toast |
| Excel opens and executes a formula from exported data | Missing `csvSanitize()` on export | Wrap every CSV/Excel cell value with `csvSanitize()` |
| Number displays as `0.30000000000000004` | Raw float arithmetic | `Math.round(x * 100) / 100` before any display |
| Date field off by one day | `new Date("2025-01-15")` = UTC midnight → local time = previous day | Split the string: `[y,m,d] = date.split('-').map(Number)` |
| Chart.js throws "Canvas already in use" | New chart created on canvas that still has one | Always call `destroyChart(key)` before `new Chart()` |
| Search stuck showing 0 results on page 2+ | Page not reset when search changes | Always set `page = 1` when search or any filter changes |
| Duplicate record IDs after bulk import | Sequential counter resets on reload | Always use `Date.now() + Math.random()` — never `++counter` |
| App shows blank white page on first load | IndexedDB error handled with only `console.error` | Show a user-visible error UI in `req.onerror` |
| localStorage crash with QuotaExceededError | Too many or too large snapshots | Catch the error, trim snapshots further, retry, warn user |
| Bulk undo restores wrong or partial data | Records array mutated between delete and undo timeout | Clone array immediately: `saved = records.map(r => ({...r}))` |
| Tag rename doesn't update some records | Wrong field name in propagation (`r.type` vs `r.category`) | Double-check the field name matches your schema before deploying |
| Chart shows wrong colors after theme toggle | `retintCharts()` not wired to `toggleTheme()` | Always call `retintCharts()` inside `toggleTheme()` |
| File backup fails silently after permission revoke | No error handling on `createWritable()` | Wrap in try/catch, show error toast on failure |
| User sees error toast when cancelling file picker | `AbortError` not filtered | `if (e.name !== 'AbortError') toast(...)` |
| Auth session never expires | Session TTL not checked | Always verify `Date.now() - sess.ts < SESSION_TTL` on load |
| PWA install banner shown when already installed | Standalone mode not detected | Check `matchMedia('(display-mode: standalone)').matches` first |
| Autocomplete list out of date after tag rename | `updateTagDatalist()` not called after tag change | Call `updateTagDatalist()` at the end of every tag add/rename/delete |
| onSearch doesn't reset to page 1 | `page` state not reset in search handler | Set `state.page = 1` inside `onSearch()` before calling render |
| Mobile modal covers full screen and can't scroll | Modal container not set to `overflow-y: auto` | Every modal inner div: `max-height: 90vh; overflow-y: auto` |
| CSP blocks inline scripts | CSP `script-src` missing `'unsafe-inline'` | Include `'unsafe-inline'` in CSP script-src (required for single-file apps) |
