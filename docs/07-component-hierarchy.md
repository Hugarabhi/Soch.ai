# 07 — Component Hierarchy (Frontend)

**Stack:** Next.js 15 (App Router) · React 19 · TypeScript · Tailwind v4 ·
shadcn/ui · Framer Motion · React Flow · TanStack Query · Zustand.

Design rules: **smart containers vs dumb components**, co-located feature
components, single source of truth for types (`packages/types`), and all server
state through TanStack Query (never fetch in effects).

---

## 1. Top-level composition

```
RootLayout (fonts, theme provider, Toaster, CommandPalette)
├── MarketingLayout (landing)
├── (app) layout  → AppShell
│   ├── Sidebar (nav + streak + XP mini) | TopBar (search, theme, avatar)
│   └── <Outlet/> → feature pages (below)
```

## 2. Feature page trees

### Onboarding
```
OnboardingWizard (client, Zustand-persisted draft)
├── Stepper (1..5, progress)
├── StepSubject (Combobox + suggestion chips)
├── StepLevel (SegmentedControl)
├── StepGoal (ChipMultiSelect)
├── StepTime (TimeTiles)
├── StepStyle (StyleTiles)
└── SubmitBar (validate → POST /onboarding/submit → /roadmap/:id)
```

### Building (roadmap generation)
```
BuildProgressView
├── AgentTimeline (AgentStep: state, duration, live labels)
├── ProgressRing
├── ResearchTicker (named sources w/ icons + hit counts)
├── PartialTopicCard (streamed pre-commits)
└── ErrorState (retryable with agent.failed payload)
```

### Roadmap workspace
```
RoadmapWorkspace
├── ViewSwitcher (Tree | Timeline | MindMap)
├── RoadmapTree (React Flow; RoadmapNode[collapsible], Edge with progress)
│   ├── StageNode → StageGauge, EstTimeChip
│   ├── TopicNode → SubtopicList, QuizLink, ResourceDrawer trigger
│   └── ProgressEdge (animated dash on completion)
├── RoadmapTimeline (MilestoneRail, ProgressLine)
├── RoadmapMindMap (RadialMindMap)
├── TopicDetailDrawer (routed `/roadmap/:id/topic/:topicId`)
│   ├── SubtopicChecklist
│   ├── ResourceList (ResourceCard: score, official badge, why, Start)
│   ├── AssignmentCard + MarkDone
│   └── RevisionPlan (spaced chips)
└── ContextMenu (regenerate topic, archive stage)
```

### Coach
```
DailyCoach
├── TodayTasks (TaskItem[optimistic complete + XP toast])
├── WeeklyPlan (day columns)
└── MonthlyGoals (GoalCard)
```

### Mentor
```
MentorWorkspace (thread list + chat + context rail)
├── ThreadList (ThreadRow: title, updated, snippet)
├── ChatScroll (MessageBubble; MentorMessage → MarkdownRenderer
│   ├── CodeBlock (Geist Mono, copy)
│   └── CitationChip (opens source)
│   └── StreamingCursor)
├── Composer (textarea, /hints, submit on Enter)
└── ContextRail (matching KB refs, GoalChip, ReadinessChip)
```

### Interviewer
```
InterviewerWorkspace
├── SessionMeta (type, topic, run count)
├── QuestionCard (QuestionText, CodePrompt if coding)
├── AnswerRecorder (media+transcript) | AnswerEditor
├── SkipButton
├── FeedbackCard (ScoreRing, RubricBars[accuracy|clarity|depth], SuggestionText)
├── NextQuestionButton (adaptivity armed)
└── SessionTrendMini (sparkline of running score)
```

### Portfolio & readiness
```
ReadinessPage
├── ReadinessRing (overall %) → ReadinessBreakdown
├── SkillGapList (GapRow: skill, level delta, suggested next topic)
├── SalaryBandCard
└── AssetActions (BuildPortfolioButton → PortfolioPreview, ResumeHealth, LinkedInCopy)
```

### Analytics & gamification
```
AnalyticsPage
├── KpiGrid (KpiCard ×6 with micro sparkline)
├── SkillRadar (RadarChart)
├── ConsistencyHeatmap (CalendarHeatmap)
├── ReadinessTrend (AreaChart)
├── Leaderboard (LaneTab: friends | region | global, opt-in)
└── BadgeShelf (Badge, earnedAt)
```

### Shared/global
```
CommandPalette (⌘K; fuzzy over roadmap, notes, bookmarks, resources)
Toast/Toaster (shadcn + sonner; success carries +XP)
StreakIndicators (TopBar flame, sidebar streak)
ThemeProvider (dark/light/system, system → sync)
QueryErrorBoundary (per-feature; Retry)
SseProvider  (EventSource wrapper: reconnect, Last-Event-ID)
```

## 3. Reusable primitives (built on shadcn/ui)
`Button`, `Card` (glass), `Input/Textarea`, `Select/Combobox`, `Checkbox`,
`Switch`, `Tabs`, `Dialog/Sheet/Drawer`, `Popover`, `Tooltip`, `Toast`,
`Skeleton`, `SegmentedControl`, `ProgressRing`, `Chip`, `Badge`, `Avatar`,
`Command`, `DropdownMenu`, `Table`, `SkeletonText`.

## 4. State management strategy

| Layer | Tool | Scope |
| --- | --- | --- |
| Server state | TanStack Query (+ mutation keys) | roadmaps, topics, resources, sessions, analytics |
| Streaming | SseProvider + Query (`useSse` hook) | agent progress, mentor, interview |
| Ephemeral UI | Zustand stores | wizard draft (persist), active roadmap view, theme |
| Router state | Next.js params/search | roadmaps/:id, topic drawer, view switcher |

## 5. Performance rules
- Route-level code splitting (Next auto); React Flow lazy-loaded only on
  visualizer mount; markdown renderer lazy on mentor.
- Windowing for chat + long lists (TanStack Virtual).
- Stable `useMemo`/`memo` on heavy nodes (roadmap tree); micro-animations
  budgeted to layout/will-change hints.
- Images via `next/image` + CDN; fonts via `next/font`.
- Core Web Vitals targets: LCP < 2.0s field p75, INP < 200ms, CLS < 0.05.