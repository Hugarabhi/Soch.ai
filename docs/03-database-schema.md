# 03 — Database Schema & Data Strategy

**Source of truth:** `packages/database/prisma/schema.prisma`
This document explains the model, the reasoning behind the choices, and the
operational strategy (indexing, partitioning, hot paths).

---

## 1. Design principles

1. **Event-sourced progress, working-operational state.** Raw activity lives in
   append-only tables (`ProgressLog`, `XpEvent`, `InterviewSession`). Derived
   aggregates (dashboard, gamification, readiness) are recomputed by workers so
   read paths are single-row lookups.
2. **AI output is data, not code.** The generated roadmap is stored as a
   normalized graph for querying, plus a versioned JSON snapshot (`graph`) so
   views can render deterministically and be re-materialized.
3. **Postgres is the system of record.** Redis/Qdrant are caches/accelerators;
   both can be rebuilt from Postgres.
4. **RLS-ready.** Every multi-tenant table has `userId` and is queryable with
   `WHERE userId = :me` in all service queries; Postgres Row-Level Security can
   be enabled post-hoc without refactor (see docs/11).

## 2. Entity relationship overview

```
User 1─1 UserProfile
User 1─N Roadmap 1─N RoadmapStage 1─N Topic 1─N Resource
User 1─N Roadmap 1─N Milestone
User 1─N Project (N─1 Roadmap)
User 1─N ProgressLog (N─1 Project)
User 1─N InterviewSession
User 1─1 JobReadinessSnapshot
User 1─1 GamificationState ; User 1─N XpEvent ; User N─N Achievement (via UserAchievement)
User 1─N Notification
User 1─N Note / Bookmark / ChatThread 1─N ChatMessage
User 1─N MemorySummary
User 1─N WeeklyReport
ResearchSnapshot (cache, keyed "subject:market:locale")
```

## 3. Key type decisions

| Column/type | Decision | Why |
| --- | --- | --- |
| IDs | CUID (cuid2) on all PKs | Collision-safe, sortable-ish, URL-safe; avoids numeric enumeration attacks |
| Enums | Native Postgres enum via Prisma | Integrity at DB; maps cleanly to TS union + Python Literal |
| Roadmap `graph` | `Json` | Flexible agent output; queries hit relational children for dynamics |
| Subtopics/prereqs | `String[]` (array) | Read-mostly, bounded size; avoids join explosion |
| Goals | `GoalType[]` + `primaryGoal` | Multi-goal with a single driver |
| Scores/ratings | `Float` 0..1 | Normalized so the ranking engine can mix dimensions |
| Money/time | coin/XP as `Int`; hours as `Int` | No floats for money/hours |
| `ResearchSnapshot.ttl` | indexed | Cheap invalidation of cached research |

## 4. Index strategy (hot queries)

- `ProgressLog(userId, date)` — timeline + streaks (partial index `WHERE kind = ...` for heavy kinds).
- `ProgressLog(userId, kind)` — category rollups for dashboards.
- `XpEvent(userId, createdAt)` — XP history & leaderboard window queries.
- `InterviewSession(userId, createdAt)` — trend chart.
- `Roadmap(userId, status)` — "my roadmaps" list.
- `Resource(topicId)` partial index `WHERE rank <= 20` — top-resources lookup.
- `Search` on `UserProfile.subject, level` — exploration features.
- `ChatMessage(threadId, createdAt)` — mentor history pagination.
- `Notification(userId, readAt, scheduledFor)` — due-notification sweep.
- Full-text: use `tsvector` GIN index on `Note.title/body`, `Project.title` for KB search (semantic search via Qdrant for recall).

## 5. Partitioning & scale plan

| Table | Growth hazard | Strategy |
| --- | --- | --- |
| `ProgressLog` | Time-series, heavy writes | Partition by month on `date` (native range partitioning); hot partition stays reasonably small |
| `XpEvent` | Time-series | Partition by month; aggregate lossless rollups into `GamificationState` |
| `ChatMessage` | Streaming, high volume | Partition by month; enforce message count per thread (cap 200, summarize older via `MemorySummary`) |
| `InterviewSession`, `WeeklyReport` | Medium | Retain 18 mo, archive cold to S3 parquet for analytics |

**Read replicas:** dashboard aggregates, KB search indexing, and report crunching
read from `DATABASE_URL_REPLICA_READ` (Prisma `replicas` config).

## 6. Data integrity & cleanup

- All user-owned rows cascade on `User` delete; `ResearchSnapshot` TTL sweeper
  runs hourly; embeddings/vectors cleaned when parent row deleted (worker).
- Idempotency keys: `ResearchSnapshot.cacheKey` unique; `Resource.url` unique —
  ranking engine upserts, never duplicates.
- Audit: `ProgressLog.evidence` and `InterviewSession.feedback` keep provenance
  for gamification anti-cheat and AI-quality auditing.

## 7. Prisma migration workflow

```bash
pnpm --filter @soch/database exec prisma migrate dev    # local
pnpm --filter @soch/database exec prisma migrate deploy # CI/CD per docs/13
```

- Schema changes are reviewed via `prisma migrate diff` in PRs.
- Destructive operations are gated behind `dataMigrations/` + manual approval.