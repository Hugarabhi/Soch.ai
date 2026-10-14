# 06 — UI Wireframes & Design System

**Visual DNA:** Apple-calm × Linear-precise × Vercel-grounded. Minimal,
typography-led, glassmorphism, micro-animated, dark & light.

---

## 1. Design tokens

### Color (HSL)
| Token | Light | Dark |
| --- | --- | --- |
| `--bg` | `210 40% 98%` | `222 42% 6%` |
| `--surface` / glass | `0 0% 100% / .7` | `222 30% 12% / .55` |
| `--border` | `220 13% 88%` | `220 12% 22%` |
| `--primary` (Soch indigo) | `243 84% 62%` | `243 84% 67%` |
| `--success` | `160 84% 39%` | `160 84% 52%` |
| `--warning` | `38 92% 50%` | `38 92% 55%` |
| `--danger` | `0 72% 51%` | `0 72% 60%` |
| `--text` / `--muted` | `222 47% 12%` / `222 16% 45%` | `210 40% 96%` / `218 11% 65%` |

### Type & spacing
- Font: **Inter** (UI) + **Geist Mono** (code/telemetry). Fluid scale: `12/14/16/18/24/32/48` with `clamp()`.
- Spacing: 4px base grid (4/8/12/16/24/32/48/64). Radius: `8/12/16/full`.
- Motion: 120–200 ms ease-out for micro-interactions; spring for entrances;
  `prefers-reduced-motion` → static.

### Glassmorphism spec
Backdrop `blur(20px) saturate(160%)`, surface `background: var(--surface)`,
1px border with `inset` highlight, layered on gradient meshes (indigo→violet,
low-opacity). Used for: sidebar, cards, modals, streaming panels.

---

## 2. Screens (high-level wireframes)

### W1 — Landing
```
┌────────────────────────────────────────────────────────────┐
│  Soch◆ai   ›   Product  Roadmap  Pricing  About   [Sign in] │
│                                                            │
│        Think. Learn. Build.                                │
│   Five answers. A living roadmap. Your career on rails.    │
│              [ Start free ]   [ Watch demo ]               │
│   → streaming progress preview behind hero (glass card)    │
│   → module badges: Roadmap · Projects · Mentor · Interview │
│   → social proof row                                       │
└────────────────────────────────────────────────────────────┘
```

### W2 — Onboarding (5-step wizard, one card, stepper on top)
```
┌── Step 1/5 ──────────────────────────────── ✓✓✓✓○ ──────┐
│  What do you want to learn?                              │
│  [ AI Engineering ------▾ ]  (combobox + suggestions)    │
│   ✓Suggested: AI Engineering · Python · Web Dvlp · UiUX… │
│  [ Continue → ]                                          │
└──────────────────────────────────────────────────────────┘
   Steps: subject → level (segmented 3) → goal (chips, multi)
          → time (4 tiles) → style (5 tiles)
   Auto-saves draft to localStorage; back/forward editable.
```

### W3 — "Building your roadmap" (the money shot)
```
┌────────────────────────────────────────────────────────────┐
│  Building your AI Engineering roadmap…            (ref ch) │
│                                                            │
│  ✓ Profile understood        (1.1s)                        │
│  ▸ Researching web   ●●●○○  (6 sources live, ticker)       │
│     » Latest roadmaps  » trending skills  » salaries       │
│  ▸ Ranking resources   ○○○○○                              │
│  ▸ Writing Beginner roadmap  (live diff appearing)          │
│  ● Generating projects (3/6)                               │
│                     [ progress ring 42% ]                  │
│  (SSE events render as animated checklist; partial topics  │
│   appear before the run finishes — feels alive, ~30s)      │
└────────────────────────────────────────────────────────────┘
```

### W4 — Roadmap workspace (tree/timeline/mindmap)
```
┌─ Sidebar ───────────┬──────────────────────────────────────────┐
│ Logo    Soch ◆      │  [Tree] [Timeline] [Mind]    ✓ 24h  ☀7 │
│                     │                                         │
│ ● Dashboard         │   Beginner (est. 12 wk)   ▓▓▓▓▓░░ 62%   │
│ ● Roadmap           │   ├─ Python foundations   ✓ 6 topics    │
│   ▸ Beginner        │   ├─ Data structures      ● 4/8 done    │
│   ▸ Intermediate    │   └─ Intro to LLMs        ○ next        │
│   ▸ Advanced        │                                         │
│ ● Projects          │   Intermediate (est. 16 wk)  ░░░░░ 8%   │
│ ● Coach             │   Advanced    …
│ ● Interviewer       │                                         │
│ ● Portfolio         │   Timeline view: milestone dots +        │
│ ● Analytics        ⚡  |   progress line connecting them        │
│ ● Mentor            │   Mind-map: center=goal, spokes=stages   │
└─────────────────────┴──────────────────────────────────────────┘
 (glass sidebar, collapsible nodes, `DONE` nodes animate ✔+XP toast)
```

### W5 — Topic detail & resources
```
  Python Foundations                 {status: in-progress}
  ● Subtopics  7        ● Quiz 2/8
  ● Resources — ranked with rationale + cite + "official" badge
      1. Python Docs (official)   ⭐4.8  freshness 9.8  [Start]
      2. Real Python …   (why: beginner-friendly, 2026 update)
  ● Assignment: build a terminal quiz game → [Mark done]
  Right rail: revision plan (spaced) + estimated time + difficulty
```

### W6 — Daily Coach
```
  Today — Wed, 24 Sep      (from your 10h/week plan, 1h25m)
  [✓] Topic: Python loops           30m
  [ ] Quiz: list comprehensions     10m
  [ ] Project: CLI quiz step 2      45m
  [ ] Revision: OOP notes (4d due)  10m
  [ ] Practice Q: 3 problems        20m
  → streak ▲12 · XP earned today +320 · [Open schedule calendar]
```

### W7 — Mentor (chat with context)
```
┌─ Mentor ──────────────────────────────┬─ Context pane ────────┐
│  You: Should I learn React or Next?   │ › Roadmap: Beginner   │
│  M: [streaming tokens…]               │ › Your goal: FAANG    │
│  └─ [citation] Next.js vs React 2026   │ › 2 saved notes match │
│  You: Review my project…              │ › Interview score 7.1 │
└───────────────────────────────────────┴───────────────────────┘
 (citations clickable; code blocks with copy; `/ask-roadmap` hints)
```

### W8 — Interviewer
```
  Session: Technical · ML  (Run 3 of 6)
  Q3/10   "Explain attention. What's cached in KV cache?"
  [ record answering ]  [ type answer ]  [ SKIP → ]
  ○●●●○○○○○○  difficulty adjusts ●
  Feedback card (post-answer): 7.6/10
     Accuracy 8 · Clarity 7 · Depth 7
     +  Suggested improvement (1 line, cited)
     [ Next question → ]   · overall score trend at right
```

### W9 — Analytics & Gamification
```
  Dashboard                             [Week ▾]
  ┌────┬────┬────┬────┬────┬────┬────┐
  │4.2h│3/8 │ 7.1│+18 │ 12d│ 84%│x1280│  hours·topics·interview
  │hours│topics│score│XP │streak│ready│+repo │  ·streak·readiness·XP
  └────┴────┴────┴────┴────┴────┴────┘
  Skill graph (radar, mastery%) █  Consistency calendar heatmap █
  Readiness ▲ 62% (gaps list)     Leaderboard (my lane)  █ badges ═
```

### W10 — Portfolio & readiness
```
  Job Readiness           62%                                [Rebuild]
  Projects (6) → shown as deliverable cards + links
  Resume health 78%  → [Fix]  LinkedIn summary [Copy]
  GitHub action items (3)   Salary band: ₹8–15 LPA (IN, mid)
  → [Generate portfolio] live renders site preview + S3 link
```

---

## 3. Interaction & UX principles
- **Zero dead-ends:** every "AI thinks" moment has a progress story (labels +
  live evidence), not a spinner.
- **Optimistic UI** for check-offs with rollback + toast on failure.
- **Reduced motion:** toggle; streak/XP animations become static badges.
- **Keyboard:** full tab flow; `/` opens command palette (search roadmap, notes,
  resources — hybrid KB search).
- **Empty states are coaching:** no empty charts — show "start project 1" action.