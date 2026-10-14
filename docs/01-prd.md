# 01 — Product Requirements Document (PRD)

**Product:** Soch.ai — AI Career Intelligence Platform
**Tagline:** Think. Learn. Build.
**Status:** Draft v1.0
**Owner:** Product & Platform
**Last updated:** 2026-09-11

---

## 1. Executive summary

Soch.ai is **not a chatbot**. It is an **AI career intelligence platform** that
ingests just five answers from a learner and autonomously produces a complete,
living learning operating system: a research-backed, dynamically adaptive
roadmap from Beginner → Intermediate → Advanced, with generated projects,
milestones, interview preparation, daily coaching, portfolio automation, job
readiness scoring, and gamified analytics.

The product differentiates itself with **agentic research** (it reads the live
internet, GitHub, job boards, official docs, and communities before building
anything), **dynamic adaptation** (the roadmap rewrites itself from your
progress), and **outcome orientation** (every path converges on a measurable
career goal: job, freelance, startup, or skill mastery).

## 2. Problem statement

1. **Roadmap paralysis.** Learners spend weeks assembling scattered YouTube
   playlists, blogs, and bootcamp syllabi, most of which are already outdated.
2. **Static content rot.** The single most popular "how to learn X" content is
   months or years old; skills, tools, and salaries move fast.
3. **No feedback loop.** Nobody adapts a learning plan when the learner learns
   faster/slower, gets a job, or switches goals.
4. **No outcomes.** Generic courses track *completion*, not *job readiness*.
5. **Overwhelming choice.** 500 tutorials → zero signal about which one is
   current, high quality, and matches how the learner actually learns.

## 3. Target users & personas

| Persona | Profile | Core need | Willing to pay |
| --- | --- | --- | --- |
| **The Career Switcher** | 24–40, employed in unrelated field | Concrete job-ready path in ≤6 months | Yes |
| **The Student** | 18–24, CS/design/marketing student | Land internship/placements, syllabus+ | No/cheap |
| **The Freelancer** | 20–35, wants income via skills | Fast practical + portfolio | Yes |
| **The Founder** | 22–40, wants startup/SaaS | MVP skills + execution plan | Yes |
| **The Hobbyist** | any | Guided fun learning | No |

Primary region assumption: **India first** (Pune/Mumbai/Bengaluru/Delhi), then
global English markets. This drives currency, job-board research weighting
(LinkedIn India, Naukri, internal referral channels), and language support.

## 4. Goals & non-goals

### 4.1 Goals
- **G1.** Convert a 5-answer onboarding into a complete roadmap in **< 60 s**
  (streamed, not a spinner).
- **G2.** Every roadmap is built on **live research** with visible sources —
  never templated.
- **G3.** The roadmap continuously mutates with user progress **and** market
  data (adaptive).
- **G4.** A "job readiness %" that the learner can trust and improve weekly.
- **G5.** One platform, one login: roadmap, projects, mentor, interviewer,
  portfolio, analytics in a single workspace.

### 4.2 Non-goals (v1)
- No AI-generated certificates / degree awarding.
- No real job placement / recruiter marketplace (future, see docs/15).
- No native mobile apps in v1 (fully responsive PWA instead).
- No enterprise/team management in v1.
- No payments in v1 (feature-gate via free tiers; Stripe in v2).

## 5. Product principles
1. **Show the work.** Every recommendation cites a source and a "why".
2. **Adapt or die.** Roadmaps are hypotheses, re-planned weekly against reality.
3. **Outcome over completion.** KPIs measure readiness and projects, not video minutes.
4. **Human tool, not oracle.** The AI is a coach that explains its reasoning.
5. **Privacy by design.** Learning data is personal; process it with consent and
   encrypt it in transit and at rest.

## 6. User journey

```
Landing → Authenticate (Clerk, OAuth/social) 
  → 5-question onboarding wizard (< 3 min)
  → [AI deep-work: Profile Engine + Research Agent + Ranking + Roadmap Gen]
  → Roadmap workspace (live, streaming)
  → Daily Coach feed starts T+1
    ├── Learn (topics, resources, quizzes)
    ├── Build (auto-generated projects)
    ├── Train (AI interviewer)
    ├── Track (dashboard, gamification, revisions)
    └── Ready (job readiness engine → portfolio, resume, salary)
```

**Activation definition:** user reaches "roadmap generated" and opens one
project. **Retained definition:** user completes ≥ 2 weekly tasks or one
interview practice in calendar week.

## 7. Functional requirements (by module)

Each module below maps 1:1 to the workspace modules and to backend/AI service
components in docs/02–08.

| # | Module | Key requirements (v1) |
| --- | --- | --- |
| M1 | **AI Profile Engine** | Derive profile fields (goal, pace, budget, language, country, difficulty pref, learning style) from the 5 answers + optional refinement; persist to `UserProfile`; refreshable. |
| M2 | **Browser Research Agent** | Agentic web research across Google, GitHub, YouTube, Reddit, official docs, Stack Overflow, Medium, Dev.to, Hashnode, Product Hunt, Hacker News, LinkedIn, Coursera, Udemy, freeCodeCamp, Khan Academy, roadmap.sh, MS Learn, Google/AWS/Azure, OpenAI, Anthropic, LangChain, HF, PapersWithCode, arXiv. Extract: latest roadmaps, tech, trending skills, best resources, demand, salary, job market, emerging tools, breaking changes, community picks. Rate-limited, dehydrated to structured `ResearchSnapshot`. |
| M3 | **Resource Ranking Engine** | Score every resource on quality, freshness (age/recency), popularity (stars/views/shares), community rating, official-doc priority boost, learning efficiency (time-to-skill). Emit ranked top-N per topic with a score and a one-line rationale. |
| M4 | **Roadmap Engine** | Generate 3 stages (Beginner/Intermediate/Advanced); each stage = topics, subtopics, projects, assignments, quizzes, revision plan, resources, estimated time, milestones, difficulty, prerequisites. Serializable `RoadmapStage` graph. |
| M5 | **Project Generator** | Generate mini, portfolio, industry, startup idea, open-source contribution, capstone projects. Each: architecture, folder structure, feature list, difficulty, learning outcomes, deployment guide, ready-to-paste GitHub README. |
| M6 | **Daily AI Coach** | Produces today's tasks, weekly tasks, monthly goals, schedule (respecting weekly hours), revision plan, practice questions, coding challenges. Deterministic-ish against progress state. |
| M7 | **AI Mentor** | Streaming chat (SSE) with roadmap/topic context; supports: explain topic, what next, am I job ready, framework choice, project review. Guardrails & citations. |
| M8 | **AI Interviewer** | Technical + HR + behavioral mock interviews; coding challenges; scoring (correctness, communication, depth); structured feedback with per-answer scores; session history. |
| M9 | **Portfolio Builder** | Generate portfolio site (static export), project showcase, resume (ATS-friendly), LinkedIn summary, GitHub profile optimization, personal brand blurb. |
| M10 | **Job Readiness Engine** | Compute readiness % from weighted signals (projects done, skills mastered, portfolio presence, interview scores, resume quality, GitHub activity). Output also: skill-gap analysis, salary prediction band. |
| M11 | **Analytics Dashboard** | Progress, learning hours, projects completed, certificates, GitHub contributions (via API), interview score trend, consistency, weekly growth, skill graph. |
| M12 | **Roadmap Visualizer** | Interactive tree/timeline/mind-map views, collapsible nodes, progress lines, animated completions (React Flow). |
| M13 | **Gamification** | XP, coins, achievements, badges, daily streaks, weekly challenges, leaderboards (opt-in public). Anti-cheat: reward *evidence* (projects done, hours logged with proof), not merely activity. |
| M14 | **Smart Notifications** | Email + in-app + push (PWA): study time, revision, deadlines, interview practice, project updates. Rate-limited, respects quiet hours. |
| M15 | **Knowledge Base** | Personal vault: notes, bookmarks, resources, projects, progress, chat history, roadmaps; semantic search over it (embeddings + Qdrant). |

## 8. The 5 onboarding questions (spec)

1. **What do you want to learn?** — free-text with suggestions (AI Engineering,
   Python, Web Dev, Cybersecurity, Graphic Design, UI/UX, Video Editing,
   Marketing, Finance, Business, Music, Content Creation, Photography, ...).
2. **Current skill level** — Beginner / Intermediate / Advanced.
3. **Goal** — Internship / Job / Freelancing / Startup / Launch SaaS / YouTuber /
   AI Engineer / Crack FAANG (multi-select allowed, primary picked).
4. **Weekly available time** — 5 / 10 / 20 / 40 hours.
5. **Learning style** — Videos / Projects / Reading / Docs / Mixed.

Rules: Q3 allows up to 3 sub-goals; Q1 free-text with normalized *subject
taxonomy* (an alias mapper: "web dev" → `web-development`). Everything beyond
these is AI-inferred or incrementally collected — never gating.

## 9. Non-functional requirements (NFRs)

| Area | Requirement (v1 target) |
| --- | --- |
| Performance (API p95) | Read APIs < 200 ms; roadmap streaming first byte < 2 s |
| Roadmap generation | End-to-end (first topic visible) < 30 s; full graph < 60 s; resumable |
| Availability | 99.9% target for web+API; AI service 99.5% (graceful degrade to cached roadmap) |
| Capacity | 1k → 1M users (see docs/14); design for horizontal scale from day one |
| Data privacy | GDPR + India DPDP Act 2023 compliant; data locality options |
| Accessibility | WCAG 2.2 AA; full keyboard nav; reduced-motion support |
| Browsers | Last 2 versions of Chrome/Safari/Firefox/Edge; mobile-first responsive |
| Security | OWASP Top 10; secrets in vaults; zero-trust on internal APIs |
| Cost ceiling | AI token cost/user/week budgeted & monitored (see docs/14) |

## 10. Success metrics (North Star + guardrails)

- **North Star:** **Weekly "Ready +1"** — count of users whose Job Readiness
  score and Evidence Score both increased week-over-week.
- Activation: onboarding completion %, roadmap-gen completion %, first-project-open rate.
- Engagement: DAU/MAU, weekly tasks completed/user, mentor sessions/user, interview sessions/user.
- Retention: W1/W4/W12 cohort curves; goal-attainment rate (self-reported + evidence).
- Quality: % of citations confirmed live at T+30; user-reported mentorship helpfulness (CSAT).
- Efficiency: tokens used per generated roadmap; latency percentiles; AI error/retry rate.
- Guardrail metrics: support ticket rate, refund/churn (post-pay), prompt-abuse rate.

## 11. Risks & mitigations

| Risk | Likelihood | Impact | Mitigation |
| --- | --- | --- | --- |
| Google Maps-style scraping bans / research blocks | High | Med | Rate-limit browser agent, rotate politely, use search APIs (Brave/Tavily) first, cache research 24–72 h |
| LLM hallucinations in roadmaps | Med | High | Ground every claim to a retrieved source; show sources; eval suite (docs/10) |
| Outdated research | Med | Med | Freshness-scored resources; re-research cadence; archive snapshots |
| Token costs explode | Med | High | Model routing (local-first batch, hosted for UX-critical), budget caps, caching |
| User abandons after onboarding | Med | Med | Stream the build; show working previews during research ("watching the agent work") |
| Prompt injection via research content | Low-Med | Med | Input sanitization, tool output as untrusted data, judge-agent pass (docs/11) |

## 12. Release plan

| Release | Scope |
| --- | --- |
| **Alpha (internal)** | Onboarding + roadmap gen (1 category), minimal dashboard |
| **Beta (invite, India)** | Full roadmap, projects, daily coach, gamification lite, analytics |
| **v1 GA** | All M1–M15 modules production-grade; payments gated |
| **v2 (future)** | See docs/15 |