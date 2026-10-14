# 04 — API Specification

**Owner:** Backend (`apps/api`, NestJS). Contract generated via `@nestjs/swagger`
(OpenAPI 3) and published at `/api/docs` in every environment.

**Base URL (prod):** `https://api.soch.ai/api/v1`
**Content type:** `application/json; charset=utf-8`
**Auth:** `Authorization: Bearer <sessionToken>` (Clerk). Backend verifies Clerk
JWKS at the edge of each request, then resolves `userId` from the Clerk webhook
synced `User` record.

---

## 1. Conventions

- **Pagination:** cursor-based for time-sorted lists (`?before=<id>`, `?after=<id>`, `?limit=50` max 200); page-based for static lists.
- **Errors:** RFC 7807 problem+json envelope:
  ```json
  { "type": "about:blank", "title": "Conflict", "status": 409,
    "detail": "Roadmap already generating for this profile", "code": "ROADMAP_IN_FLIGHT" }
  ```
- **Idempotency:** mutating AI triggers accept `Idempotency-Key` header; replays return the original record.
- **Streaming:** SSE endpoints below; `Accept: text/event-stream`; events carry `id` (Last-Event-ID resume), `event`, `data`.
- **Rate limits:** per user-token — reads 120 req/min, writes 30 req/min, AI triggers 5 req/10 min (see docs/11).
- **Fields:** snake_case JSON (matches DB); all timestamps ISO-8601 UTC; money in minor units (future).

## 2. Endpoint map

### 2.1 Identity & profile
| Method | Path | Description |
| --- | --- | --- |
| POST | `/auth/webhooks/clerk` | Clerk user.created/user.deleted sync (signed, whitelisted IP) |
| GET | `/users/me` | Current user + config |
| PATCH | `/users/me` | Update basics (name, locale, avatar) |
| GET | `/users/me/profile` | AI profile (`UserProfile`) |
| PUT | `/users/me/profile` | Refresh profile / trigger re-inference (`POST /users/me/profile/refine`) |

### 2.2 Onboarding
| Method | Path | Description |
| --- | --- | --- |
| POST | `/onboarding/subjects/search` | Subject alias→taxonomy resolution (e.g. "web dev"→`web-development`) |
| POST | `/onboarding/submit` | Validate the 5 answers; returns `profileId` (no AI yet) |

### 2.3 Roadmaps (M1–M4, M12)
| Method | Path | Description |
| --- | --- | --- |
| POST | `/roadmaps` | Start generation `{profileId}` → `{roadmapId, status}` (idempotent per profile) |
| GET | `/roadmaps/:id/stream` | SSE: `agent.progress`, `agent.stage`, `partial.roadmap`, `roadmap.ready`, `roadmap.failed`, `roadmap.resumed` |
| GET | `/roadmaps/:id` | Full roadmap (graph + materialized stages) |
| GET | `/roadmaps` | List mine `?status=&limit=` |
| PATCH | `/roadmaps/:id` | Manual tweaks (rename, reorder, archive) |
| POST | `/roadmaps/:id/regenerate` | Re-research a stale dashboard region (`?region=topic`) |
| POST | `/roadmaps/:id/visualize?view=tree|timeline|mindmap` | Server-side projection for the visualizer |
| PATCH | `/roadmaps/:id/stages/:stageId/topics/:topicId` | `{status: DONE}` → progress recalculation |
| GET | `/roadmaps/:id/revision-plan` | Next revision window per topic (spaced repetition) |

### 2.4 Resources & research (M2, M3)
| GET | `/roadmaps/:id/resources?topicId=&rank∈` | Ranked resources for a topic with rationale + citation |
| GET | `/research/snapshots/:cacheKey` | True-up research snapshot (staff/debug) |

### 2.5 Projects (M5) & milestones
| Method | Path | Description |
| --- | --- | --- |
| GET | `/roadmaps/:id/projects` | Generated projects for the roadmap |
| POST | `/projects/:id/start` | Mark in-progress, clone blueprint |
| POST | `/projects/:id/complete` | `{evidenceUrl}` → progress + XP + portfolio hook |
| PATCH | `/projects/:id` | Update status/urls |
| GET | `/projects/:id/readme` | Rendered README download (`.md`) |
| POST | `/projects/:id/github` | Create GitHub repo from blueprint (OAuth `repo` scope) |
| GET | `/roadmaps/:id/milestones` + PATCH `/milestones/:id` | Milestone list/check-off |

### 2.6 Daily coach & mentor (M6, M7)
| Method | Path | Description |
| --- | --- | --- |
| GET | `/coach/today` | Today's plan (tasks, schedule, revision, practice) |
| GET | `/coach/week` | Weekly task list + monthly goals |
| POST | `/coach/tasks/:id/complete` | Evidence-verified completion |
| GET | `/mentor/threads` / POST `/mentor/threads` | Thread list / create |
| GET | `/mentor/threads/:id/messages` | History (cursor) |
| POST | `/mentor/threads/:id/messages` | `{content}` → returns `messageId`, stream at `/mentor/threads/:id/stream` |
| GET | `/mentor/threads/:id/stream` | SSE token stream with `citations` event + optional `code` blocks |
| POST | `/mentor/project-review` | Attach project JSON → coach critique (streamed) |

### 2.7 Interviews (M8)
| Method | Path | Description |
| --- | --- | --- |
| POST | `/interviews` | `{type, topic}` → creates session + first question(s) |
| POST | `/interviews/:id/answer` | Submit answer audio/text; `{skip?: bool}`; stream feedback |
| GET | `/interviews/:id/stream` | SSE: `question`, `evaluation`, `score`, `session.complete` |
| POST | `/interviews/:id/complete` | Finalize; compute aggregate scores |
| GET | `/interviews/sessions?type=` | History + trend data |

### 2.8 Readiness, analytics, portfolio (M10, M11, M9)
| Method | Path | Description |
| --- | --- | --- |
| GET | `/readiness` | Snapshot: readiness%, interview%, skill gaps, salary band |
| POST | `/readiness/refresh` | Recompute (worker async; stream status) |
| GET | `/analytics/summary` | KPIs: hours, projects, interviews, streak, XP |
| GET | `/analytics/graph?subject=` | Skill mastery graph (nodes/edges for radar/chart) |
| GET | `/analytics/weekly-great` | Weekly growth composition |
| GET | `/portfolio` | Bundled: resume content, LinkedIn summary, GitHub tips, brand blurb |
| POST | `/portfolio/build` | Async generate/update portfolio HTML export (S3 signed URL result) |
| POST | `/portfolio/github-sync` | Pull public GitHub contributions from `/user/repos` (OAuth) |

### 2.9 Gamification (M13) & notifications (M14)
| Method | Path | Description |
| --- | --- | --- |
| GET | `/gamification/state` | XP, level, coins, streak, leaderboard rank |
| GET | `/gamification/leaderboard?scope=global|lane` | Top N (opt-in) |
| GET | `/achievements` | Catalogue + my earned |
| GET | `/notifications` + PATCH `/notifications/:id` | List/mark read; preferences `PUT /notifications/prefs` |

### 2.10 Knowledge base (M15)
| Method | Path | Description |
| --- | --- | --- |
| GET/POST | `/kb/notes` , PATCH/DELETE `/kb/notes/:id` | Notes CRUD (auto-embedded) |
| GET/POST/DELETE | `/kb/bookmarks` | Bookmarks |
| GET | `/kb/search?q=` | Hybrid: semantic (Qdrant) + keyword; returns sources with score |

## 3. Streaming contract (SSE) — roadmap generation

```
event: agent.progress      data: {"node":"web_research","step":3,"of":8,"pct":38}
event: agent.stage         data: {"stage":"roadmap_generation","msg":"Drafting Beginner stage"}
event: partial.roadmap     data: {"stage":"BEGINNER","topic":{"title":"..."}}
event: roadmap.ready       data: {"roadmapId":"ch_...","url":"/roadmap/ch_..."}
event: roadmap.failed      data: {"error":{"code":"AI_SERVICE_DOWN","retryable":true}}
```

Reconnect: client sends `Last-Event-ID`; server replays events after that id
from Redis stream (`RDB stream per roadmapId`, trimmed 500).

## 4. AI service contract (api ↔ ai)

Internal-only (mTLS across k8s). Key RPCs:

| Pseudonym | Path (REST/SSE) | Purpose |
| --- | --- | --- |
| `GenerateRoadmap` | POST `/v1/agents/roadmap` | Full agent run, streams `AgentEvent` proto/JSON |
| `MentorTurn` | POST `/v1/agents/mentor` | One mentor turn + memory update |
| `InterviewTurn` | POST `/v1/agents/interview` | Judge an answer; score + next question |
| `RankResources` | POST `/v1/agents/rank` | Score/rank resource list (batch) |
| `Embed` | POST `/v1/ml/embed` | Text→vector (batched) |
| `Readiness` | POST `/v1/agents/readiness` | Compute readiness + gaps + salary band |
| `DailyPlan` | POST `/v1/agents/daily-plan` | Deterministic-ish planner over progress state |

Auth: shared `AI_SERVICE_API_KEY` (service account), token budget headers, and
strict request/response schemas (validated by Pydantic on both sides).

## 5. Webhooks (outbound) & scheduled jobs

- **Clerk webhook** (inbound) — sync users.
- **GitHub App** — repo create events → project status/DEPLOYED.
- **Email provider + push provider** — notified by `NotificationWorker`.
- **Cron:** daily coach at 7:00 local; weekly report Sunday 18:00 local;
  re-research nightly (per market); streak/XP rollup every hour; readiness
  refresh weekly; notes/vectors GC nightly.

## 6. Versioning & evolution

- URL versioning (`/api/v1`); additive changes land in v1; breaking changes → v2 + 6-month deprecation header `Sunset`.
- All responses carry `ETag` where cacheable; SSE endpoints never cached.)