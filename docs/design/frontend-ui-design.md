# MedMind — Frontend & UI Design

**Date:** June 24, 2026  
**Status:** Approved — Ready for Implementation  
**Framework:** Next.js 14 (App Router)  
**Styling:** Vanilla CSS with CSS custom properties  
**Icons:** Lucide React  
**Charts:** Recharts (symptom timeline only)  

---

## 1. Overview

MedMind's frontend has **6 pages** and a shared layout with a collapsible sidebar. The design prioritizes a premium, dark-mode-first aesthetic that feels trustworthy and calm — appropriate for a health application.

### Pages

| Route | Page | Priority |
|---|---|---|
| `/auth` | Login / Register | Required |
| `/chat` | Chat Interface (main page) | Required |
| `/memories` | Memory Viewer | Required |
| `/family` | Family Health Tree | Nice-to-have (cut visual if behind schedule) |
| `/timeline` | Symptom Timeline | Nice-to-have |
| `/health-summary` | Doctor-Ready Summary Export | Required |

---

## 2. Design System

### Color Palette

Dark mode by default. Medical apps used at night (which health anxiety tends to trigger) should not blast users with a white screen.

```css
:root {
  /* Primary — Trust, reliability */
  --color-primary: #2563EB;
  --color-primary-hover: #3B82F6;
  --color-primary-dark: #1E40AF;

  /* Accent — Health, positive */
  --color-accent: #10B981;
  --color-accent-hover: #34D399;

  /* Semantic */
  --color-warning: #F59E0B;
  --color-danger: #EF4444;
  --color-danger-hover: #F87171;

  /* Surfaces */
  --color-bg: #0F172A;
  --color-surface: #1E293B;
  --color-surface-hover: #334155;
  --color-border: #334155;

  /* Text */
  --color-text: #F8FAFC;
  --color-text-secondary: #94A3B8;
  --color-text-muted: #64748B;

  /* Memory Tier Colors */
  --tier-critical: #EF4444;
  --tier-important: #F59E0B;
  --tier-contextual: #3B82F6;
  --tier-ephemeral: #64748B;

  /* Spacing */
  --space-xs: 4px;
  --space-sm: 8px;
  --space-md: 16px;
  --space-lg: 24px;
  --space-xl: 32px;
  --space-2xl: 48px;

  /* Border Radius */
  --radius-sm: 6px;
  --radius-md: 10px;
  --radius-lg: 16px;
  --radius-full: 9999px;

  /* Shadows */
  --shadow-sm: 0 1px 2px rgba(0, 0, 0, 0.3);
  --shadow-md: 0 4px 12px rgba(0, 0, 0, 0.4);
  --shadow-lg: 0 8px 24px rgba(0, 0, 0, 0.5);

  /* Glassmorphism */
  --glass-bg: rgba(30, 41, 59, 0.7);
  --glass-border: rgba(148, 163, 184, 0.1);
  --glass-blur: blur(12px);

  /* Typography */
  --font-sans: 'Inter', -apple-system, BlinkMacSystemFont, sans-serif;
  --font-mono: 'JetBrains Mono', 'Fira Code', monospace;
}
```

### Typography

Load **Inter** from Google Fonts. It's clean, modern, and highly readable at all sizes.

```css
/* Font Sizes */
--text-xs: 0.75rem;    /* 12px — timestamps, badges */
--text-sm: 0.875rem;   /* 14px — secondary text, labels */
--text-base: 1rem;     /* 16px — body text, messages */
--text-lg: 1.125rem;   /* 18px — section headers */
--text-xl: 1.25rem;    /* 20px — page titles */
--text-2xl: 1.5rem;    /* 24px — hero text */
```

### Glassmorphism Cards

Used for memory cards, family member cards, and alert cards.

```css
.glass-card {
  background: var(--glass-bg);
  backdrop-filter: var(--glass-blur);
  border: 1px solid var(--glass-border);
  border-radius: var(--radius-lg);
  padding: var(--space-lg);
  box-shadow: var(--shadow-md);
  transition: transform 0.2s ease, box-shadow 0.2s ease;
}

.glass-card:hover {
  transform: translateY(-2px);
  box-shadow: var(--shadow-lg);
}
```

### Micro-Animations

```css
/* Fade in for new messages */
@keyframes fadeInUp {
  from { opacity: 0; transform: translateY(10px); }
  to { opacity: 1; transform: translateY(0); }
}

/* Pulse for streaming indicator */
@keyframes pulse {
  0%, 80%, 100% { opacity: 0.4; transform: scale(0.8); }
  40% { opacity: 1; transform: scale(1); }
}

/* Slide in for sidebar */
@keyframes slideInLeft {
  from { transform: translateX(-100%); }
  to { transform: translateX(0); }
}
```

---

## 3. Shared Layout

All authenticated pages share a layout with a collapsible sidebar.

```
┌──────────────────────────────────────────────────────────┐
│  Sidebar (280px, collapsible)  │  Main Content Area      │
│                                │                         │
│  ┌────────────────────────┐    │                         │
│  │ 🧠 MedMind             │    │  (page-specific content)│
│  │    Your AI Health       │    │                         │
│  │    Companion            │    │                         │
│  └────────────────────────┘    │                         │
│                                │                         │
│  [+ New Chat]                  │                         │
│                                │                         │
│  Session History               │                         │
│  (grouped by date)             │                         │
│                                │                         │
│  ────────────────              │                         │
│                                │                         │
│  Navigation                    │                         │
│  💬 Chat                       │                         │
│  🧠 Memories                   │                         │
│  👨‍👩‍👧 Family Tree               │                         │
│  📈 Timeline                   │                         │
│  📋 Health Summary             │                         │
│                                │                         │
│  ────────────────              │                         │
│  ⚙️ Settings                   │                         │
│  🚪 Logout                     │                         │
└──────────────────────────────────────────────────────────┘
```

**Responsive behavior:**
- **Desktop (>1024px):** Sidebar always visible, 280px wide
- **Tablet (768-1024px):** Sidebar collapses to icon-only (60px)
- **Mobile (<768px):** Sidebar hidden, hamburger menu to toggle overlay

---

## 4. Page Designs

### 4.1 Auth Page (`/auth`)

Split-screen layout. No sidebar.

```
┌──────────────────────┬──────────────────────┐
│                      │                      │
│   🧠 MedMind         │   ┌──────────────┐   │
│                      │   │    Email      │   │
│   Your AI health     │   ├──────────────┤   │
│   companion that     │   │   Password   │   │
│   remembers your     │   ├──────────────┤   │
│   story.             │   │  [Login]     │   │
│                      │   └──────────────┘   │
│   ● Remembers your   │                      │
│     medical history  │   Don't have an       │
│   ● Tracks symptoms  │   account? Register   │
│     over time        │                      │
│   ● Prepares you     │                      │
│     for doctor visits│                      │
│                      │                      │
└──────────────────────┴──────────────────────┘
```

**Components:**
- `AuthPage` — container with split layout
- `AuthForm` — handles both login and register with a toggle
- Left panel has subtle gradient animation background

**Mobile:** Stacks vertically — branding on top, form below.

---

### 4.2 Chat Interface (`/chat`) — Main Page

Where users spend 90% of their time.

```
┌────────────────────────────────────────────────────────┐
│ Sidebar          │  Chat Area                          │
│                  │                                     │
│ [+ New Chat]     │  ┌─────────────────────────────┐   │
│                  │  │ 🧠 MedMind                    │   │
│ Today            │  │ Welcome back, Ade. Before we  │   │
│  ● Headache...   │  │ get to your question — you    │   │
│  ● Medication..  │  │ mentioned wanting to get your │   │
│                  │  │ cholesterol checked...         │   │
│ Yesterday        │  └─────────────────────────────┘   │
│  ● Family...     │                                     │
│                  │        ┌────────────────────────┐   │
│                  │        │ Yeah I haven't done    │   │
│                  │        │ that yet, but I wanted │   │
│                  │        │ to ask about...        │   │
│                  │        └────────────────────────┘   │
│                  │                                     │
│                  │  ┌─────────────────────────────┐   │
│                  │  │ 🧠 MedMind                    │   │
│                  │  │ No worries — I'll keep that   │   │
│                  │  │ on my list...                  │   │
│                  │  │              [🔄 Regenerate]   │   │
│                  │  └─────────────────────────────┘   │
│ ──────────       │                                     │
│ 📋 Summary       │  ┌──────────────────────────────┐  │
│ 🧠 Memories      │  │ 💬 Type your message...   [➤] │  │
│ 👨‍👩‍👧 Family       │  └──────────────────────────────┘  │
│ 📈 Timeline      │                                     │
└────────────────────────────────────────────────────────┘
```

**Components:**
- `ChatSidebar` — session list grouped by date + navigation links
- `SessionItem` — shows preview text, click to load session
- `MessageBubble` — different styling for user (right-aligned, blue) and AI (left-aligned, surface gray)
- `StreamingIndicator` — three animated dots while AI generates
- `RegenerateButton` — only shown on the last AI message
- `ChatInput` — text input with submit button, auto-grows with content

**Message bubble styling:**
```css
/* User messages */
.message-user {
  background: var(--color-primary);
  color: white;
  border-radius: var(--radius-lg) var(--radius-lg) var(--radius-sm) var(--radius-lg);
  margin-left: 20%;
  align-self: flex-end;
}

/* AI messages */
.message-ai {
  background: var(--color-surface);
  color: var(--color-text);
  border-radius: var(--radius-lg) var(--radius-lg) var(--radius-lg) var(--radius-sm);
  margin-right: 20%;
  align-self: flex-start;
}
```

**SSE streaming handling:**
```
1. User submits message → POST /api/chat
2. Add user message to UI immediately (optimistic)
3. Create empty AI message bubble with StreamingIndicator
4. Open EventSource connection
5. On each "token" event → append to AI message bubble
6. On "done" event → close connection, show RegenerateButton
```

---

### 4.3 Memory Viewer (`/memories`)

Full-page view of all stored memories, grouped by tier.

```
┌────────────────────────────────────────────────────────┐
│ Sidebar │  Memory Viewer                               │
│         │                                              │
│         │  What I Remember About You    [Filter ▼]     │
│         │                                              │
│         │  ┌─ CRITICAL (5) ─────────────────────────┐  │
│         │  │                                         │  │
│         │  │  🔴 Penicillin allergy — causes hives   │  │
│         │  │     Mar 15, 2026 · Accessed 7 times     │  │
│         │  │     [✏️ Edit] [🗑️ Delete]                │  │
│         │  │                                         │  │
│         │  │  💊 Lisinopril 20mg — blood pressure     │  │
│         │  │     Updated May 2026 (was 10mg)         │  │
│         │  │     [✏️ Edit] [🗑️ Delete] [📜 History]   │  │
│         │  │                                         │  │
│         │  │  👨‍👩‍👧 Mother — Type 2 Diabetes            │  │
│         │  │     Mar 10, 2026 · Accessed 12 times    │  │
│         │  │     [✏️ Edit] [🗑️ Delete]                │  │
│         │  │                                         │  │
│         │  └─────────────────────────────────────────┘  │
│         │                                              │
│         │  ┌─ IMPORTANT (12) ────────────────────────┐ │
│         │  │  ... (collapsed by default, click to     │ │
│         │  │       expand)                            │ │
│         │  └─────────────────────────────────────────┘  │
│         │                                              │
│         │  ┌─ CONTEXTUAL (8) ────────────────────────┐ │
│         │  │  ...                                     │ │
│         │  └─────────────────────────────────────────┘  │
│         │                                              │
│         │  ┌─ EPHEMERAL (3) ─────────────────────────┐ │
│         │  │  ...                                     │ │
│         │  └─────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

**Components:**
- `MemoryViewer` — page container with filter controls
- `TierSection` — collapsible section per tier with colored left border
- `MemoryCard` — glassmorphism card showing fact, metadata, and actions
- `EditMemoryModal` — inline modal for editing a memory's fact text
- `VersionHistoryModal` — shows superseded versions of a memory
- `FilterDropdown` — filter by tier, category, or subject

**Tier section left-border colors:**
- Critical: `var(--tier-critical)` (red)
- Important: `var(--tier-important)` (amber)
- Contextual: `var(--tier-contextual)` (blue)
- Ephemeral: `var(--tier-ephemeral)` (gray)

---

### 4.4 Family Health Tree (`/family`)

CSS-based tree layout with connecting lines. No external graph library.

```
┌────────────────────────────────────────────────────────┐
│ Sidebar │  Family Health Tree          [+ Add Member]  │
│         │                                              │
│         │          Paternal Side  │  Maternal Side      │
│         │                        │                     │
│         │    ┌──────────────┐    │  ┌──────────────┐   │
│         │    │  Grandfather │    │  │ Grandmother  │   │
│         │    │  Hypertension│    │  │  (healthy)   │   │
│         │    └──────┬───────┘    │  └──────┬───────┘   │
│         │           │           │          │           │
│         │    ┌──────┴───────┐   │  ┌───────┴──────┐   │
│         │    │    Father    │   │  │    Mother    │   │
│         │    │  Stroke (55) │───┼──│  Diabetes T2 │   │
│         │    │  Hypertension│   │  │              │   │
│         │    └──────┬───────┘   │  └──────┬───────┘   │
│         │           └───────────┼─────────┘           │
│         │                       │                      │
│         │              ┌────────┴────────┐             │
│         │              │      YOU        │             │
│         │              │  Hypertension   │             │
│         │              │  Pre-diabetes   │             │
│         │              └─────────────────┘             │
│         │                                              │
│         │  Risk Factors From Family:                    │
│         │  ⚠️ Diabetes risk (mother's side)             │
│         │  ⚠️ Cardiovascular risk (father's side)       │
└────────────────────────────────────────────────────────┘
```

**Components:**
- `FamilyTree` — CSS grid/flexbox tree with `::before`/`::after` connecting lines
- `FamilyMemberCard` — glassmorphism card showing member name, label, conditions
- `AddMemberModal` — form to add a new family member
- `RiskSummary` — bottom section summarizing inherited risk factors

**CSS tree connecting lines:**
```css
.tree-node::before {
  content: '';
  position: absolute;
  top: -20px;
  left: 50%;
  width: 2px;
  height: 20px;
  background: var(--color-border);
}

.tree-level {
  display: flex;
  justify-content: center;
  gap: var(--space-xl);
  position: relative;
}

.tree-level::before {
  content: '';
  position: absolute;
  top: 0;
  left: 25%;
  width: 50%;
  height: 2px;
  background: var(--color-border);
}
```

---

### 4.5 Symptom Timeline (`/timeline`)

Vertical timeline with month markers and alert badges.

```
┌────────────────────────────────────────────────────────┐
│ Sidebar │  Symptom Timeline            [6 months ▼]    │
│         │                                              │
│         │  ⚠️ ALERT: Progressive neurological symptoms  │
│         │  Headache → Vision → Dizziness over 3 months │
│         │  Severity: High — See a doctor               │
│         │                                              │
│         │  Jun 2026 ──────────────────────────────     │
│         │    │                                         │
│         │    ├── ● Positional dizziness                │
│         │    │   Confidence: High                      │
│         │    │   "I've been feeling dizzy when I       │
│         │    │    stand up quickly"                     │
│         │    │                                         │
│         │  May 2026 ──────────────────────────────     │
│         │    │                                         │
│         │    │   (no symptoms recorded)                │
│         │    │                                         │
│         │  Apr 2026 ──────────────────────────────     │
│         │    │                                         │
│         │    ├── ● Blurred vision (evenings)           │
│         │    │   Confidence: High                      │
│         │    │                                         │
│         │  Mar 2026 ──────────────────────────────     │
│         │    │                                         │
│         │    ├── ● Recurring headaches (3-4x/week)    │
│         │    │   Confidence: High                      │
│         │                                              │
└────────────────────────────────────────────────────────┘
```

**Components:**
- `TimelinePage` — container with time range selector
- `AlertBanner` — red/amber banner at top for pattern alerts
- `TimelineMonth` — month label with vertical line
- `TimelineEntry` — symptom card connected to the vertical line
- `TimeRangeSelector` — dropdown: 3 months, 6 months, 12 months, all

**Vertical line styling:**
```css
.timeline-line {
  position: absolute;
  left: 24px;
  top: 0;
  bottom: 0;
  width: 2px;
  background: linear-gradient(
    to bottom,
    var(--color-primary),
    var(--color-border)
  );
}

.timeline-dot {
  width: 12px;
  height: 12px;
  border-radius: 50%;
  background: var(--color-primary);
  border: 2px solid var(--color-bg);
  position: absolute;
  left: 19px;
}
```

---

### 4.6 Health Summary (`/health-summary`)

Clean, print-friendly page with copy and print buttons.

```
┌────────────────────────────────────────────────────────┐
│ Sidebar │  Health Summary                              │
│         │                              [📋 Copy] [🖨️]  │
│         │                                              │
│         │  ┌──────────────────────────────────────┐    │
│         │  │  HEALTH SUMMARY                      │    │
│         │  │  Generated June 24, 2026             │    │
│         │  │  Patient-reported information         │    │
│         │  │  (not a medical record)              │    │
│         │  │                                      │    │
│         │  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │    │
│         │  │                                      │    │
│         │  │  ALLERGIES                           │    │
│         │  │  • Penicillin (causes hives)         │    │
│         │  │  • Sulfa drugs                       │    │
│         │  │                                      │    │
│         │  │  CURRENT MEDICATIONS                 │    │
│         │  │  • Lisinopril 20mg (blood pressure)  │    │
│         │  │    Changed from 10mg → 20mg (May '26)│    │
│         │  │  • Metformin 500mg (blood sugar)     │    │
│         │  │                                      │    │
│         │  │  CHRONIC CONDITIONS                  │    │
│         │  │  • Hypertension (diagnosed 2024)     │    │
│         │  │  • Pre-diabetes (diagnosed 2025)     │    │
│         │  │                                      │    │
│         │  │  FAMILY HISTORY                      │    │
│         │  │  • Mother — Type 2 Diabetes          │    │
│         │  │  • Father — Stroke (55), Hypertension│    │
│         │  │                                      │    │
│         │  │  RECENT SYMPTOMS                     │    │
│         │  │  • Mar 2026: Headaches (3-4x/week)   │    │
│         │  │  • Apr 2026: Blurred vision          │    │
│         │  │  • Jun 2026: Positional dizziness    │    │
│         │  │                                      │    │
│         │  │  PENDING                             │    │
│         │  │  • Cholesterol test (Jun 10)         │    │
│         │  │                                      │    │
│         │  │  ━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━   │    │
│         │  │  ⚕️ MedMind — AI health information   │    │
│         │  │  tool. Not a medical record.         │    │
│         │  └──────────────────────────────────────┘    │
│         │                                              │
└────────────────────────────────────────────────────────┘
```

**Components:**
- `HealthSummaryPage` — container with action buttons
- `SummaryCard` — the actual summary content, styled for print
- `CopyButton` — copies `formatted_text` from API to clipboard
- `PrintButton` — triggers `window.print()` with print-specific CSS

**Print styles:**
```css
@media print {
  .sidebar, .action-buttons, nav {
    display: none !important;
  }
  .summary-card {
    background: white;
    color: black;
    box-shadow: none;
    border: 1px solid #ccc;
  }
}
```

---

## 5. Responsive Breakpoints

| Breakpoint | Sidebar | Layout | Chat Input |
|---|---|---|---|
| Desktop (>1024px) | Full (280px) | Side-by-side | Fixed bottom |
| Tablet (768-1024px) | Icons only (60px) | Side-by-side | Fixed bottom |
| Mobile (<768px) | Hidden (overlay on toggle) | Full width | Fixed bottom with safe area |

```css
@media (max-width: 1024px) {
  .sidebar { width: 60px; }
  .sidebar-label { display: none; }
}

@media (max-width: 768px) {
  .sidebar {
    position: fixed;
    z-index: 100;
    transform: translateX(-100%);
    transition: transform 0.3s ease;
  }
  .sidebar.open {
    transform: translateX(0);
  }
}
```

---

## 6. Key Interactions & Animations

| Interaction | Animation |
|---|---|
| New message appears | `fadeInUp` (0.3s ease) |
| AI streaming tokens | Text appears character-by-character |
| Streaming in progress | Three dots pulsing |
| Sidebar session hover | Background lightens, subtle scale |
| Memory card hover | `translateY(-2px)` + shadow increase |
| Page navigation | Fade transition (0.2s) |
| Modal open | Backdrop fade + modal slide up |
| Delete confirmation | Red-tinted modal with confirmation |
| Copy to clipboard | Button text changes to "Copied ✓" for 2s |
| Alert banner | Subtle left-border pulse animation |

---

## 7. Component Tree

```
App
├── AuthPage (no sidebar)
│   ├── BrandingPanel
│   └── AuthForm
│
└── AppLayout (with sidebar)
    ├── Sidebar
    │   ├── Logo
    │   ├── NewChatButton
    │   ├── SessionList
    │   │   └── SessionItem (×N)
    │   ├── NavLinks
    │   └── UserMenu (settings, logout)
    │
    ├── ChatPage
    │   ├── MessageList
    │   │   ├── MessageBubble (×N)
    │   │   ├── StreamingIndicator
    │   │   └── RegenerateButton
    │   └── ChatInput
    │
    ├── MemoryViewerPage
    │   ├── FilterControls
    │   ├── TierSection (×4)
    │   │   └── MemoryCard (×N)
    │   ├── EditMemoryModal
    │   └── VersionHistoryModal
    │
    ├── FamilyTreePage
    │   ├── FamilyTree
    │   │   └── FamilyMemberCard (×N)
    │   ├── RiskSummary
    │   └── AddMemberModal
    │
    ├── TimelinePage
    │   ├── AlertBanner
    │   ├── TimeRangeSelector
    │   └── TimelineMonth (×N)
    │       └── TimelineEntry (×N)
    │
    └── HealthSummaryPage
        ├── ActionButtons (copy, print)
        └── SummaryCard
```

---

## 8. Design Decisions

| Decision | Choice | Rationale |
|---|---|---|
| **Dark mode default** | Yes, no light mode toggle for MVP | Health apps are used at night. Dark mode is expected. Light mode is a post-hackathon feature. |
| **Family tree approach** | CSS tree with `::before`/`::after` lines | Zero dependencies, looks great in demo video, fast to build. Upgrade to D3/React Flow later if time permits. |
| **Chat streaming** | SSE (Server-Sent Events) | Simpler than WebSocket. One-direction streaming is all we need. |
| **Font** | Inter (Google Fonts) | Clean, modern, excellent readability. Industry standard. |
| **Icons** | Lucide React | Lightweight, consistent, tree-shakeable. |
| **Charts** | Recharts (timeline only) | Only used on one page. Lightweight, React-native, good defaults. |
| **No component library** | Vanilla CSS + custom components | Full control over the premium look. No fighting a library's opinions. Faster for a small app. |
| **Print support** | CSS `@media print` on health summary | Users need to print/copy the summary for their doctor. Simple CSS solution. |

---

## 9. Next Steps

1. Design Auth (final component)
2. Write the full implementation plan
3. Begin coding
