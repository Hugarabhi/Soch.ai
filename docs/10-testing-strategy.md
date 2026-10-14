# 10 — Testing Strategy

**Principle:** quality is a property of the pipeline, not the QA step. Tests run
in CI pre-merge; AI behavior is tested like software via standardized evals.

---

## 1. Test pyramid (Soch.ai flavor)

```
        ┌────────────────────────────┐
        │   E2E (Playwright)  ~4%    │  happy-path journeys + streaming resilience
        │   AI evals + contract  ~8% │  agents, judge, prompts, golden sets
        │   Integration ~35%         │  api ↔ db/queue, api ↔ ai mock, workers
        │   Unit ~53%                │  components, reducers, routers, utils
        └────────────────────────────┘
```

## 2. Tiers in detail

### 2.1 Unit (Vitest — web/api; pytest — ai)
- Reducers, formatting, guards, DTO validation, xp/coins math, ranking score
  union, spaced-repetition scheduler, salary band model.
- Component tests with Testing Library (jsdom) for wizard steps, chips, toasts.
- Deterministic time via `vi.setSystemTime` for streaks/reports.

### 2.2 Integration (NestJS e2e-Supertest; pytest-asyncio)
- **API × Postgres:** Prisma against local ephemeral DB (`DATABASE_URL` to a
  disposable schema); CRUD + idempotency + pagination + RLS-shaped queries.
- **Queue worker tests:** fake Redis (BullMQ in-memory) verifying notification,
  gamification, report jobs produce correct writes and DLQs.
- **api ↔ ai contract tests:** Pact-style — mock AI service responses; assert
  request/response JSON against OpenAPI + Pydantic schemas (breaking-change
  gate). `docker compose` OTEL not required.
- **SSE controller tests:** assert event sequence for generate/mentor/interview
  against recorded event fixtures.

### 2.3 E2E (Playwright)
- **Journey 1 — activation:** marketing → sign-in → onboarding 5 steps → sees
  "Building your roadmap" → roadmap ready (< 90 s CI timeout) → open topic.
- **Journey 2 — learning loop:** check topic DONE → XP toast → analytics KPI +1.
- **Journey 3 — mentor + interviewer:** ask mentor (mock AI via record/playback
  SSE fixture), record interview answer, see feedback card.
- **Journey 4 — portfolio/readiness:** build portfolio → S3 URL opens.
- **Streaming resilience:** `Last-Event-ID` reconnect retries and recovers.
- Mobile viewport pass + reduced-motion snapshot test.
- Dark/light theme snapshot regression (Storybook visual).

### 2.4 AI evals (`services/ai/evals/`)
| Suite | What it measures | Gate |
| --- | --- | --- |
| `roadmap_quality` | fidelity to subject taxonomy, stages present, prereq ordering, no invented citations | ≥ 0.9 fidelity |
| `mentor_qa` | usefulness, safety, citation accuracy (LLM-as-judge ×2 panel) | ≥ 0.85 |
| `interview_rubric` | score calibration vs human-labeled gold | MAE ≤ 0.5 |
| `resource_rank` | official-doc boost, freshness, precision@5 vs manual picks | p@5 ≥ 0.8 |
| `prompt_regression` | frozen-hash prompts changed → triggers review, runs diffs on fixed prompts dataset | no silent drift |
- Runs nightly + on PRs touching `services/ai/`.
- **Cost control:** eval suite capped by token budget; samples flagged not-cached
  random on 5% traffic (canary judge) for real-world drift.

### 2.5 Performance / load (K6 + Playwright)
- K6 suites: auth-heavy reads (200/s), roadmap list, analytics, SSE concurrent
  500 clients; soak runs with GC/trace analysis.
- Latency budgets per docs/01 §9. Autoscaling assertions (HPA reacts < 2 min).

### 2.6 Security tests (docs/11)
- ZAP/OWASP scan nightly on staging; `pip-audit`/`npm audit`/`trivy` in CI;
- Prompt-injection adversarial suite (research content crafted to hijack),
  SSRF check on research agents, authz negative tests (cross-user reads).

## 3. Fixtures & golden data
- **Recordings:** SSE event fixtures and AI responses are recorded from golden
  subjects (`web-development`, `ai-engineering`, `python`, `graphic-design`)
  and versioned in `services/ai/evals/fixtures/`.
- **Seed profiles** per persona for reproducible demos & e2e.
- Never cache personal data in fixtures — synthetic only.

## 4. Reporting & quality gates
- Coverage thresholds: unit ≥ 85% lines on web reducers/api modules; enforced in CI.
- Test parallelization via Turborepo + `--split`; e2e shards ×4 machines.
- Failure policy: flaky tests quarantined (not skipped silently), flake budget 1%.

## 5. What we deliberately do NOT test
- LLM internals; visual pixel equality (except key strokes); third-party
  provider behavior (mocked). We test OUR behavior around them.