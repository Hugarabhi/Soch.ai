# 05 — AI Agent Workflows

**Owner:** AI service (`services/ai`, FastAPI + LangGraph + LangChain).
This document defines the agentic architecture, the shared state machine, every
agent's contract, memory, guardrails, and the evaluation strategy.

---

## 1. Operating model

- **LangGraph** `StateGraph` with a supervisor that routes between specialized
  agents. Durable checkpointer (Postgres-backed) makes every run resumable and
  replayable.
- **Cost & latency budget** per run is fixed at the start
  (`EstimateBudget(profile)`) and enforced by the model router and tool budget.
- Agents **never talk to each other as free text**; they exchange typed state
  through the shared `AgentState`. This keeps the graph testable, cheap, and
  fully auditable in OTel traces.

## 2. Shared state (`AgentState`)

```python
class AgentState(TypedDict):
    thread_id: str                     # roadmap id / session id
    profile: UserProfileData           # M1 output, validated
    research: list[ResearchDatum]      # M2 raw findings
    ranked_resources: list[RankedResource]  # M3
    roadmap_graph: RoadmapGraph | None # M4 staging output
    projects: list[ProjectBlueprint]   # M5
    progress_delta: ProgressDelta | None   # M6/M10 deltas
    reports: list[ReportBlock]         # M11/M9/M10
    memory_refs: list[MemoryRef]
    events: list[AgentEvent]           # emitted for SSE fan-out
    errors: list[str]
```

All free fields are **Pydantic-validated**; a node may only write fields it is
declared to own (narrow contracts prevent data races in parallel branches).

## 3. Agent registry

| # | Agent | Node(s) | Tools | Writes | Streams |
| --- | --- | --- | --- | --- | --- |
| A1 | **User Profiling** | `n_profile` | taxonomy resolver, skill-ontology | profile (pace, budget, country, difficulty, gap) | `agent.progress` |
| A2 | **Planning** | `n_plan` | scheduler, dependency solver | run budget, stage timing plan | `agent.progress` |
| A3 | **Web Research** | `n_research` (parallel fan-out) | Brave Search, Playwright/Chrome pool, GitHub API, YouTube, community, official docs, arXiv | `research` (fresh, cited) | `research.hit`, `research.done` |
| A4 | **Resource Ranking** | `n_rank` | scoring pipeline | `ranked_resources` | `rank.batch` |
| A5 | **Roadmap Generation** | `n_roadmap` | graph-builder, citation linker | `roadmap_graph` | `partial.roadmap` |
| A6 | **Project Generation** | `n_projects` (parallel per stage) | code-gen, best-practice templates | `projects` | `project.ready` |
| A7 | **Progress Tracking** | `n_track` (cron/on-event) | replanner, spaced-repetition | `progress_delta` | `track.updated` |
| A8 | **Career Advisor** | `n_career` (mentor sub-graph) | readiness calc, salary model, skill-gap | `reports` (career) | streamed mentor tokens |
| A9 | **Interview Coach** | `n_interview` | judge model, question bank, rubric | `InterviewSession` scoring | `question`, `evaluation` |
| A10 | **Report Generation** | `n_report` | analytics aggregation | `WeeklyReport` | `report.ready` |

## 4. Roadmap generation sub-graph (M1→M5)

```
  start → n_profile ──▶ n_plan ──▶ n_research (fan-out: k concurrent) ──▶ n_rank ──▶ n_roadmap ──▶ n_projects ──▶ END
                                    │   (each branch: search + scrape + summarize)
                                    └─▶  cache hit? skip to n_rank (ResearchSnapshot.ttl valid)
```

- **Fan-out cap** = 6 concurrent research branches; each branch capped at 4 tool
  calls before summarization (guards runaway cost).
- **Partial commits:** after `n_roadmap` commits, a `partial.roadmap` event is
  flushed so the UI can draw the Beginner stage before Advanced finishes.
- **Failure semantics:** non-critical branches (e.g., Reddit) degrade with a
  `warn` event; critical branches (`n_roadmap`) retry once on a different model
  class; final fallback = cached `ResearchSnapshot` + deterministic template,
  flagged `quality: "cached"`.

## 5. Mentor sub-graph (M7)

```
 user_msg → retrieve (Qdrant: roadmap context + notes + memory) 
          → context assembly (state + top-k) 
          → model turn (stream tokens) 
          → citation attach + guardrail judge 
          → memory upsert (summary if thread > cap)
```

Context assembly includes: current roadmap, last N progress events, the user's
stated goal, and 3–5 retrieved KB excerpts. Citations are always rendered.

## 6. Interview judge sub-graph (M8)

```
 answer → transcript normalize → rubric judge (criteria x3) → score+suggestions 
        → question selector (difficulty adapts to running score) → persist → stream
```

Debiasing: judge prompt uses a fixed rubric (accuracy, completeness, clarity,
depth), asks for a quote-of-evidence, and forbids score inflation; randomized
peer-review judge on 5% of sessions (see evals).

## 7. Memory architecture (3 layers)

1. **Thread state (short-term):** LangGraph checkpointer rows keyed by
   `thread_id` — full conversational context for the active run.
2. **Guardian summaries (long-term):** a small LLM (`qwen2.5:14b`) compresses
   finished threads into `MemorySummary(scope)` — goals, strengths, gaps,
   preferences. Updated on thread close or at 200 messages.
3. **Semantic memory:** every note, bookmark, resource, and summary is embedded
   (BGE-M3) into Qdrant with payload `{userId, scope, ts}`; retrieval filters by
   `userId` for cross-roadmap recall (e.g., "I read about RAG yesterday").

## 8. Model routing (`llm/provider.py`)

| Class of call | Preferred model | Fallback(s) |
| --- | --- | --- |
| UX-critical, low-latency (mentor tokens, judge, coach) | Hosted: `gpt-4o`/`claude-3.5` | `llama3.1:70b` (local), `deepseek` |
| Batch/background (summaries, embeddings, research digest) | Local: `qwen2.5:14b`, `bge-m3` | Hosted cheap (`gpt-4o-mini`) |
| Reasoning-heavy, low-frequency (roadmap planner, evaluation runs) | `deepseek-r1:32b` | `llama3.1:70b`, hosted strong |
| Research router (extraction only, deterministic JSON) | `function-calling` model | regex+heuristic extractor (zero-LLM mode) |

Router consults: queue depth on Ollama, per-cohort monthly token budget, latency
SLO for the call class, and a kill-switch feature flag.

## 9. Guardrails & safety

- **Prompt injection:** research/tool content is wrapped as *data* (delimited
  JSON) and never concatenated into system prompts unescaped; a `judge` agent
  flags suspicious content; tool outputs are size-capped (e.g., 4k tokens).
- **PII:** identity fields are never sent to hosted providers without signed
  consent + minimization; local-first routing keeps personal data on-prem.
- **Topic safety:** subject allow-list + category classifier blocks malicious or
  disallowed domains (medical-advice, doxing). Refusal stays neutral and offers
  alternatives.
- **Output validation:** every agent output must pass Pydantic schema +
  `assert_references_exist` (no hallucinated citations) before commit.
- **Abuse:** per-user daily token caps; anomaly detection on cost/velocity;
  cooldown for text-scraper flooding of the browser pool.

## 10. Evaluation & continuous improvement

- **Golden datasets** per agent (see `services/ai/evals/`): `roadmap_qa`,
  `mentor_qa`, `interview_rubric`, `rank_golden`.
- **LLM-as-judge** panels score sampled outputs on fidelity, usefulness,
  safety; cost & latency tracked per node.
- **A/B via feature flags:** prompts/model-versioned runs (`model=...`) compared
  on downstream KPIs (task completion, help rating).
- **Weekly eval cron** in CI (`ai-eval.yml`) blocks release on regressions of
  `fidelity >= 0.9` on golden set.