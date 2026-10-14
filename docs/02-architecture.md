# 02 — System Architecture

**Goal of this document:** give every engineer a shared mental model of Soch.ai
and justify the technology choices at the scale trajectory of 1k → 1M users.

---

## 1. Architecture principles

1. **Streams over spinners.** Any AI process longer than 1 s streams progress
   events over SSE. Never block a request for agent work.
2. **Everything is an event.** State mutations flow through the database
   (source of truth); queues fan out side effects (notifications, emails,
   gamification, cache invalidation, analytics).
3. **Owned seams.** Frontend (Next.js) ← REST/SSE → Backend (NestJS) ← gRPC/protobuf
   or REST → AI service (FastAPI/LangGraph). Each service owns its data.
4. **AI is a service, not a layer.** No LLM call happens in the product backend.
   The AI service owns models, prompts, agents, and evaluations behind a typed
   contract.
5. **Gradual degradation.** If the AI service is down, users still see their
   cached roadmap, progress, and knowledge base.
6. **Cost is a first-class concern.** Model routing (local vs hosted) and
   per-request token budgets are designed in, not bolted on.

## 2. Context diagram (C4 Level 1)

```
                        ┌───────────────────────────────────┐
      Learner           │           Soch.ai Platform        │
  ┌────────────┐        │                                   │
  │ Browser/PWA│───────▶│  Next.js Web (CDN + edge)         │
  └────────────┘        │        │                          │
                        │        ▼                          │
                        │  NestJS API  ──▶ Worker/Queue      │
                        │        │                          │
                        │        ▼                          │
                        │  FastAPI AI Service (LangGraph)   │
                        └───────┬───────────┬───────────────┘
                                │           │
                     ┌──────────▼───┐   ┌───▼───────────────┐
                     │  3rd party   │   │  Infra: Postgres,  │
                     │  (AI hosted, │   │  Redis, Qdrant,    │
                     │  search API, │   │  Ollama, S3, ...   │
                     │  OAuth, GitHub)│  └───────────────────┘
                     └──────────────┘
```

## 3. Container diagram (C4 Level 2)

### 3.1 `web` — Next.js 15 (App Router) frontend
- React 19 + TypeScript, Tailwind CSS v4, `shadcn/ui` primitives.
- **Framer Motion** for micro-interactions; **React Flow** for the roadmap
  visualizer; **TanStack Query** for server state; **Zustand** for ephemeral
  client state; **Zustand `persist`** for draft onboarding answers.
- Streaming UI: native SSE consumption via `EventSource` wrapper backed by
  TanStack Query `useStream` pattern.
- Renders on Vercel/edge-compatible hosts or self-hosted node containers;
  served through Cloudflare CDN.
- **Why Next.js over SPA:** SSR/SEO for marketing, route-based code splitting,
  RSC for fast product pages, optional middleware at the edge for auth.

### 3.2 `api` — NestJS 10 (Node 20, TypeScript) product API
- REST/JSON with OpenAPI 3 generation (`@nestjs/swagger`) as the contract.
- SSRF-safe internal rest client to the AI service; SSE controller for streaming.
- BullMQ-backed workers for side effects: notifications, gamification scoring,
  weekly reports, re-research cron, exports (resume/portfolio builds).
- **Why NestJS over Express/Fastify-string:** module boundaries, DI, decorators,
  mature queue/websocket/validation ecosystem, aligned TS types with frontend
  via `packages/types`.

### 3.3 `ai` — FastAPI (Python 3.12) agentic AI service
- **LangGraph** orchestrates the agent state machine; **LangChain**-style tools
  for search, browser automation, embeddings.
- **Ollama** for local inference (Llama 3.1 70B / Qwen2.5 / DeepSeek-R1) with a
  **model router** falling back to hosted providers (OpenAI/Anthropic/OpenRouter).
- **Playwright + CDP** in a separate headless-browser pool (container) for web
  research; **Brave Search API** as primary deterministic search to keep costs
  low and rate limits safe.
- Embeddings via `BGE-M3` (local) with Qdrant as the vector store.
- Exposes: gRPC for high-volume micro-calls (embed, rank, score) and REST/SSE
  for long-running agent runs. v1 starts REST+SSE; gRPC is the documented path
  for scale (docs/14).

### 3.4 Infrastructure services
| Service | Purpose | Notes |
| --- | --- | --- |
| PostgreSQL 16 | System of record | Prisma ORM; single writer + read replicas; partitioning for event tables |
| Redis 7 | Cache, queues (BullMQ), rate limits, streaming fan-out | ElastiCache/Valkey for managed ops |
| Qdrant | Vector search for knowledge base + semantic routing | Optimized HNSW configs |
| Supabase Storage (S3) | Portfolios, resume exports, project screenshots, uploads | Signed URLs; CDN integration |
| Ollama pool | Local LLM inference | GPU nodes in k8s |
| OpenTelemetry Collector | Traces/metrics/logs → Grafana/CloudWatch/GCP | Standard instrumentation |

## 4. Communication patterns

| Pattern | Where | Why |
| --- | --- | --- |
| REST/JSON | web ↔ api; contiguous reads/writes | Simple, cacheable, tooling-rich |
| SSE | api → web for roadmap gen, mentor chat, interview streaming | Server→client push, HTTP-friendly, reconnectable with `Last-Event-ID` |
| REST+SSE | api ↔ ai for agent runs | Same SSE ergonomics server-to-server |
| gRPC (future) | api ↔ ai hot paths | Lower overhead at high RPS (embedding/ranking micro-calls) |
| BullMQ/Redis | api workers (notifications, reports, gamification) | At-least-once semantics, visibility/retry, DLQs |
| Webhooks (outbound) | api → Clerk, GitHub, Stripe, email providers | Event-driven account/portfolio sync |
| Cron | re-research cadence, weekly report, streak/tier rollup | Distributed lock via Redis to avoid duplicate runs |

## 5. End-to-end flow: "generate my roadmap"

```
1. User completes 5-step onboarding  (web, validated)
2. POST /roadmaps {profileId}        (api)
3. api validates quota/tier → returns roadmapId + initial
   roadmap shell (pending status, streaming URL)
4. api enqueues research trigger (BullMQ) OR calls ai synchronously
   long-poll style; ai service locks the job (Redis exclusive lock)
5. ai: AgentGraph.run(profile, context)
   ├─ user_profiling (expand profile: pace, budget, country, difficulty)
   ├─ web_research (parallel tool calls: search APIs + docs + GitHub/YouTube)
   ├─ resource_ranking (score+sort, cite sources)
   ├─ roadmap_generation (structure stages→topics→subtopics→resources)
   └─ project_generation (top N projects with README stubs)
   (each node publishes progress events to Redis stream)
6. api SSE relay streams events → web renders live progress
7. ai persists final RoadmapGraph JSON to Postgres via api
   (ai service never writes the product DB directly;
    it calls POST /roadmaps/:id/result with signature)
8. web shows roadmap + streams "first topic" preview early via partial commits
9. Side-effects: notifications, gamification reward, analytics events, embeddings → Qdrant
```

**Idempotency:** `roadmapId` + per-node `stateId` make node results idempotent;
replays are safe because every node is a pure function of `(state_snapshot)`
and stores `result_hash`. **Resumability:** LangGraph checkpointer persists
`(threadId = roadmapId)` state to Postgres; a crashed run resumes from the last
committed node.

## 6. Agentic collaboration model

See docs/05 for the full agent spec. Summary:

- 10 specialized agents under a **supervisor-graph** (LangGraph).
- Shared **AgentState** struct: `profile`, `research`, `rankedResources`,
  `roadmapGraph`, `projects`, `progressDelta`, `reports`, `memoryRefs`, `errors`.
- **Memory:** 3 layers — (a) LangGraph short-term thread state, (b) long-term
  `UserMemoryGuardian` summaries written to Postgres, (c) semantic memory via
  embeddings in Qdrant.
- Agents collaborate through **well-defined node I/O contracts**, not chat
  between agents — cheap, testable, auditable.

## 7. Data & cache strategy

- **Cache layers:** CDN (static assets, public marketing), Redis (session-hot
  data, denormalized dashboard scores TTL 5 min), application L1 (none in v1,
  avoids stale-read bugs).
- **Write-behind denormalization:** Dashboard/readiness/gamification aggregates
  are maintained by workers and served hot; the raw event tables stay
  append-only (docs/03 partitioning).
- **Research cache:** `ResearchSnapshot` cached 24–72 h per (subject, market)
  to keep token + search costs bounded and answers consistent.

## 8. Observability

- **Tracing:** OTel auto-instrumentation across all three apps; trace IDs
  propagate through SSE event payloads so users can paste a `ref` for support.
- **Metrics:** RED for API, tokens consumed per agent/node, agent latency
  percentiles, research cache hit-rate, queue depth, model-routing split
  (local vs hosted), costs per cohort.
- **Logs:** structured JSON; sentry/Rollbar for error tracking on web+api;
  evals history for AI regressions.

## 9. Technology decision records (key calls)

| Decision | Choice | Alternatives rejected | Why |
| --- | --- | --- | --- |
| Repo | pnpm + Turborepo monorepo | Multi-repo | Shared types, atomic cross-stack changes, single CI |
| Product API | NestJS | Express, Fastify, tRPC | Typed modules + DI + ecosystem; tRPC couples clients |
| AI service | Python FastAPI + LangGraph | Node agents (LangGraphJS), crewAI | LangGraph = durable, resumable, testable graphs; Python = first-class LLM tooling |
| Orchestration | LangGraph state machine | Pure orchestration scripts, AutoGen swarm | Deterministic flow, checkpointing, eval-able |
| Auth | Clerk (+ webhook → backend) | Auth.js/Better-Auth self-hosted | Ship velocity, SSO, security hardening; self-host considered for data-locality later |
| ORM | Prisma | TypeORM, Drizzle | Migrations DX + type safety shared w/ frontend |
| Vector DB | Qdrant | Chroma, Pinecone, pgvector | Managed-optional, HNSW perf, payload filters; pgvector = fallback cost-saver |
| Search | Brave API + Playwright fallback | Tavily-primary, SerpAPI | Cost + coverage; Playwright for niche/deep research |
| LLM hosting | Ollama local-first + hosted fallback | Hosted-only | Cost control + data locality + availability |
| Frontend host | Cloudflare CDN + edge (Vercel optional) | S3+CloudFront only | Full-stack render flexibility, DDoS protection |

## 10. Configuration & secrets

- All secrets via environment → injected by k8s `ExternalSecrets`/vault (AWS
  Secrets Manager / Doppler). Never in repo (`.env.example` only).
- Feature flags via PostHog/Unleash; migration-safe rollout of prompts/models.

## 11. Environment topology (single region v1)

```
web (CDN/edge)  →  api (k8s deployment, HPA)  →  ai (k8s deployment + browser pool + GPU pool)
        api HPA  →  Postgres writer + replicas, Redis, Qdrant, S3, Ollama pool
```

Single region (ap-south-1 / ap-south-2 for India-first) with multi-AZ spanning.
DR strategy in docs/12.