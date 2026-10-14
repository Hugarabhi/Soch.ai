# 09 — Development Roadmap (how we build Soch.ai)

Sizing: 1 platform team (6–9 engineers split web/backend/AI) + 1 designer + 1 DPE.
Timeline assumes full-time; months below are effort months.

---

## Phase 0 — Foundations (M1)
**Goal:** standing monorepo, local infra, CI green, one vertical slice runs end-to-end.

| Deliverable | Owner |
| --- | --- |
| Repo scaffold (turbo, workspaces, eslint, prettier, tsconfig) | Web |
| Docker compose stack (pg, redis, qdrant, ollama, otel) | Infra |
| Prisma schema v1 + first migration + seed | Backend |
| Next.js shell (AppShell, theme, glass system) | Web |
| NestJS bootstrap + health + Clerk webhook sync | Backend |
| FastAPI hello + `/v1/agents/ping` contract | AI |

**Exit criteria:** `pnpm dev` runs all three; CI green; `curl /api/v1/health`.

## Phase 1 — Roadmap Core (M2–M3) 🎯 *first demo*
**Goal:** 5-question onboarding → streamed roadmap for ONE subject (web dev).
- Onboarding wizard (5 steps, drafts persisted) — Web
- `/roadmaps` POST + SSE relay + progress UI (agent timeline, partial cards) — Backend+Web
- Agent A1–A4 for single subject: profile, research (Brave + docs + GitHub),
  ranking, roadmap gen (3 stages) — AI
- Roadmap visualizer tree view + topic drawer + resources — Web
- Research snapshot cache + token budget guard — AI/Backend

**Exit criteria:** demoable "Web Development, Beginner, Job, 10h/wk, Mixed"
produces a full roadmap ≤ 60 s with ≥ 3 ranked resources per topic and citations.

## Phase 2 — Learning Loop (M4–M6)
- Projects module: blueprint generation (mini/portfolio), README export, GitHub create — AI+Backend+Web
- Progress tracking: topics/projects check-offs, progress %, revision plan (spaced repetition) — Backend+Web
- Daily coach: today/week/month planner from progress + weekly hours — AI+Backend
- Gamification v1: XP, streak, coins, achievement conditions, XP events, leaderboard opt-in — Backend+Web
- Analytics v1: KPI grid + heatmap + skill radar (worker-computed aggregates) — Backend+Web

**Exit criteria:** 24h cohort can learn 1 topic daily; XP/streak consistent; dashboards hot.

## Phase 3 — Intelligence layer (M7–M10)
- Mentor chat: threads, SSE, citations, memory guardian + Qdrant recall — AI+Web
- Interviewer: session flow, answer intake (text/audio), rubric judge, adaptive difficulty, trend — AI+Backend+Web
- Job readiness engine: factors model + skill-gap + salary band — AI
- Portfolio builder: resume/LinkedIn/GitHub audit + static portfolio export — AI+Backend+Web
- Reports: weekly report, re-research cadence — AI+Backend

**Exit criteria:** mentor helpfulness CSAT ≥ 80% (judge-evaluated); interview scores reproducible; readiness recompute ≤ 5 s.

## Phase 4 — Hardening & Scale (M5)
- Auth polish, RBAC, RLS readiness; PII minimization — Backend/Infra
- Load tests (K6) to 1k concurrent; HPA + read replicas + Redis cache — Infra
- GPU pool on Ollama; model router failover drills; LLM budget dashboards — AI/Infra
- PWA + push; smart notifications with quiet hours — Web/Backend
- Accessibility (WCAG AA pass) + perf budget enforcement in CI — Web

**Exit criteria:** p95 API < 200 ms under load; AI gen p95 < 60 s; no PII sent to hosted LLMs by default.

## Phase 5 — Beta & Iterate (M6)
- Invite-only beta (India), observability dashboards, error triage
- Evals harness live in CI (`ai-eval.yml`), golden sets extended weekly
- Onboarding funnel instrumentation (PostHog) + activation metric dashboards
- Feedback loop: dislike/flag on roadmap topics + mentor messages → fine-tune prompts

**Exit criteria:** activation ≥ 60%; W1 retention ≥ 40%; "source confirmed live" ≥ 95%.

## Phase 6 — GA + scale to millions (M7+)
- Payments (Stripe) → free/pro tiers
- Multi-region (APAC → EU/AEA for GDPR) with data residency
- Dedicated Qdrant cluster, Postgres partitioning + replica fan-out
- AI cost autoscaler (per-cohort budgets, cache-first strategies)
- Public API platform readiness (future roadmap)

## Dependency graph
```
Phase0 ▸ Phase1 ▸ Phase2 ▸ Phase3 ▸ Phase4 ▸ Phase5 ▸ Phase6
              └──────────────▶ (parallelize Phase0–2 docs/certs)
```

## Team & rituals
- Weekly AI evals review; prompt changes reviewed like code (PRs).
- Each phase ends with a **playbook writeup** (docs) + failure-mode runbook.
- Definition of Done: tests · types · observability · docs touched · eval gate · seedable.