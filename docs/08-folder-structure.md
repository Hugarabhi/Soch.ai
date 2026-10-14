# 08 — Folder Structure

Enterprise monorepo: **pnpm workspaces + Turborepo**. Three deployable apps,
shared packages, infrastructure-as-code, and a docs home. The tree below IS the
repo layout (see root `package.json`).

```
soch-ai/
├── package.json                  # turbo scripts, workspace deps
├── pnpm-workspace.yaml
├── turbo.json                    # task graph & caching
├── .env.example                  # env template (never commit .env)
├── .gitignore
├── docker-compose.yml            # local infra: postgres, redis, qdrant, ollama, otel
├── Makefile                      # developer task runner (db, dev, test, lint)
├── README.md                     # entry doc (link to /docs)
│
├── apps/
│   ├── web/                      # Next.js 15 frontend (SSR/edge, PWA)
│   │   ├── package.json
│   │   ├── next.config.mjs
│   │   ├── tailwind.config.ts
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── app/              # App Router: (marketing), (onboarding),
│   │       │                     #  dashboard, roadmap/[id], mentor,
│   │       │                     #  interviewer, projects, analytics, portfolio
│   │       ├── components/
│   │       │   ├── ui/           # shadcn primitives (button, card, dialog…)
│   │       │   ├── onboarding/   # wizard steps
│   │       │   ├── roadmap/      # visualizer, topic drawer, resource cards
│   │       │   ├── dashboard/    # KPI grid, charts
│   │       │   ├── mentor/       # chat, markdown, citations
│   │       │   ├── interviewer/  # question, recorder, feedback
│   │       │   ├── projects/     # blueprint cards, README viewer
│   │       │   └── analytics/    # radar, heatmap, leaderboard
│   │       ├── lib/              # api client, sse provider, auth hooks
│   │       ├── stores/           # zustand (wizard draft, view state)
│   │       └── styles/           # tokens, glass, tailwind layers
│   │
│   └── api/                      # NestJS product API
│       ├── package.json
│       ├── nest-cli.json
│       └── src/
│           ├── main.ts           # bootstrap, global prefix /api/v1, otel
│           ├── app.module.ts     # module graph
│           ├── modules/
│           │   ├── auth/         # Clerk webhook, guard, user resolution
│           │   ├── users/        # /users/me, profile
│           │   ├── roadmaps/     # CRUD + SSE controller + gen trigger
│           │   ├── progress/     # progress logs, revision, streaks
│           │   ├── projects/     # blueprints, README, GitHub bridge
│           │   ├── mentor/       # threads, message proxy → AI service
│           │   ├── interviews/   # sessions, answers, scoring proxy
│           │   ├── gamification/ # XP/coins/badges/leaderboard (worker-fed)
│           │   ├── notifications/ # in-app/email/push + prefs
│           │   ├── portfolio/    # build + resume/linkedin/site
│           │   ├── reports/      # weekly report + readiness endpoints
│           │   └── agents/       # typed client to AI service (gRPC/REST)
│           ├── common/           # guards, filters, dto decorators, cache
│           └── prisma/           # client init, seed scripts
│
├── services/
│   └── ai/                       # FastAPI + LangGraph (Python 3.12)
│       ├── pyproject.toml
│       ├── Dockerfile            # GPU + browser-pool images
│       ├── main.py               # FastAPI app, routers, SSE relays
│       ├── config.py
│       ├── graph.py              # LangGraph state machine (10 agents)
│       ├── agents/
│       │   ├── user_profiling.py # A1
│       │   ├── planning.py       # A2
│       │   ├── web_research.py   # A3 (fan-out, sources, cache)
│       │   ├── resource_ranking.py # A4
│       │   ├── roadmap_generation.py # A5
│       │   ├── project_generation.py  # A6
│       │   ├── progress_tracking.py   # A7
│       │   ├── career_advisor.py # A8
│       │   ├── interview_coach.py     # A9
│       │   └── report_generation.py   # A10
│       ├── tools/
│       │   ├── search.py         # Brave/Tavily search client
│       │   ├── browser_research.py # Playwright + CDP pool, SSRF-safe
│       │   ├── scrapers.py       # per-domain extractors (github, yt, docs)
│       │   ├── embeddings.py     # BGE-M3 → Qdrant
│       │   └── citations.py      # normalize + verify links
│       ├── llm/
│       │   ├── provider.py       # router (ollama/local + hosted fallback)
│       │   └── prompts.py        # versioned prompt templates (VCS-tracked)
│       ├── models/
│       │   └── schemas.py        # Pydantic contracts (shared w/ API gateway)
│       ├── memory/
│       │   ├── guardian.py       # summaries → MemorySummary
│       │   └── qdrant_store.py
│       └── evals/                # golden sets + judge harness (docs/10)
│
├── packages/
│   ├── shared/src/index.ts       # TS types + schema constants (web/api consumers)
│   ├── types/src/                # api & ai contract types (generated from OpenAPI)
│   ├── config/src/               # env parsing (zod), shared config shape
│   └── database/
│       ├── prisma/schema.prisma  # SINGLE schema source of truth
│       └── src/                  # client, migrations helpers, seed
│
├── infra/
│   ├── docker/                   # Dockerfiles, otel-collector config
│   ├── k8s/
│   │   ├── base/                 # deployments/services/ingress per app
│   │   └── overlays/production/  # replica counts, HPA, PDB, env (SealedSecrets)
│   ├── terraform/                # AWS/GCP resources, RDS, Redis, Qdrant policy
│   └── cloudflare/               # workers rules, DNS, WAF, caching
│
├── scripts/                      # dev tooling: reindex, partition maint, seed
├── .github/workflows/
│   ├── ci.yml                    # lint·typecheck·test·build·e2e
│   ├── cd.yml                    # docker build → env promote · K8s rollout
│   └── ai-eval.yml               # weekly agent evals gate
│
└── docs/                         # ← the 15 design docs (this set)
```

## Rules
- A package may only import from its dependency-only packages (`packages/*`)
  and sibling apps via declared workspace deps. Enforced by ESLint
  `import/no-extraneous` + Turborepo topological build.
- The AI service communicates with the world through `services/ai/` only; it
  never imports app code and never writes the product DB directly.
- Any change touching `packages/database/prisma/schema.prisma` requires a
  migration + doc update (docs/03) in the same PR.