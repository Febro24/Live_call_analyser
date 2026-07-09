# Sales Call Intelligence Platform — Architecture & Build Plan

---

## 1. What we are building

A B2B multi-tenant SaaS tool for sales teams and their managers. The system ingests sales call recordings (and, later, live calls), turns them into text, judges how each call went, and turns that judgment into dashboards for managers and coaching for salespeople.

The six product capabilities, in the order users experience them:

1. **Transcription** — every call recording is automatically converted to text with speaker labels. Nobody takes notes.
2. **Call judgment** — each call gets a sentiment reading (was the customer happy or frustrated?), risk flags, and an overall score.
3. **Dashboards & leaderboards** — managers see team performance at a glance; reps are ranked.
4. **Coaching** — each salesperson gets personalized feedback and linked learning material.
5. **Compliance** — the system watches for rule-breaking phrases or missing disclosures, per tenant-defined rules.
6. **Role-separated views** — managers see the big picture; salespeople see only their own scores, coaching, and progress.

**Two pipelines, one analysis brain.** The batch pipeline (recordings) is the core product and is built first. The live pipeline (real-time coaching alerts during a call) reuses the same analysis brain and is built last, because it is the hardest part and depends on everything else working.

---

## 2. System architecture overview

```
                         ┌────────────────────────────────────────────┐
                         │              FRONTEND (React)              │
                         │  Manager App View      Salesperson View    │
                         │  (team dashboards,     (my scores, my      │
                         │   compliance, ranks)    coaching, live UI) │
                         └───────────┬───────────────────┬────────────┘
                                     │ REST (TanStack Q)  │ WebSocket (live only)
                         ┌───────────▼───────────────────▼────────────┐
                         │           NESTJS API GATEWAY               │
                         │  JWT auth · BACKEND RBAC guards · tenant   │
                         │  resolution · rate limits · WS gateway     │
                         └──┬──────────┬──────────┬──────────┬────────┘
                            │          │          │          │
                ┌───────────▼──┐ ┌─────▼─────┐ ┌──▼───────┐ ┌▼────────────┐
                │ INGESTION    │ │ ANALYSIS  │ │ INSIGHTS │ │ LIVE        │
                │ recordings → │ │ scoring · │ │ rollups ·│ │ streaming   │
                │ storage →    │ │ sentiment·│ │ dashboards│ │ STT → 10-15s│
                │ batch STT    │ │ compliance│ │ leaderbds │ │ analysis →  │
                │ (BullMQ)     │ │ engine    │ │ coaching │ │ alerts      │
                └───────┬──────┘ └─────┬─────┘ └──┬───────┘ └┬────────────┘
                        │              │          │          │
              ┌─────────▼──────────────▼──────────▼──────────▼─────────┐
              │  PostgreSQL (multi-tenant) · Object storage (audio) ·  │
              │  Redis (cache, queues, socket adapter) · Central logs  │
              └─────────────────────────────────────────────────────────┘
```

External AI services: batch STT with diarization (recordings), streaming STT (live), fast LLM for classification (Gemini Flash class), larger LLM for deep post-call summaries (batch only, cost-controlled).

---

## 3. Component specifications

### 3.1 Ingestion service (batch — the core)

Accepts recording uploads (or pulls from telephony/meeting integrations later). Flow: upload → validate → store raw audio in object storage (S3-compatible) under a tenant-prefixed path → enqueue a transcription job in BullMQ → worker sends audio to batch STT with **speaker diarization** enabled → store transcript (per-utterance rows: speaker, timestamp, text) in Postgres → enqueue analysis job.

Design rules: the API returns immediately after enqueuing ("job accepted"), never blocks on transcription. Retries with exponential backoff on STT failures. Every job carries `tenantId`, `callId`, `requestId` for logging. Audio retention is a per-tenant setting (some tenants will demand transcript-only storage, deleting raw audio after processing — build this switch from day one).

### 3.2 Analysis engine (the shared brain)

One service consumed by both pipelines. Three sub-parts:

- **Sentiment analyzer.** Per-segment customer sentiment plus a whole-call sentiment arc (started frustrated, ended satisfied, etc.). Use the STT vendor's built-in sentiment where available; LLM fallback otherwise.
- **Compliance rule engine.** Two layers. Layer 1: deterministic rules (regex/keyword lists per tenant — banned phrases, required disclosures). Cheap, instant, explainable. Layer 2: LLM judgment for fuzzy rules ("did the rep pressure the customer?"). Tenants define rules in an admin UI; rules are versioned so a flag can always be traced to the rule version that raised it.
- **Scoring engine.** Produces a 0–100 call score from a weighted rubric: talk/listen ratio (computable from diarized transcript timestamps — no LLM needed), question rate, compliance adherence, sentiment trajectory, and LLM-judged dimensions (discovery quality, objection handling). Weights are tenant-configurable with sane defaults. Every score stores its component breakdown so it is explainable — a score nobody can explain is a score nobody trusts.

LLM calls in this engine always request strict JSON output, are validated against a schema, and fall back to "needs review" status (never fake data) on parse failure.

### 3.3 Insights service (dashboards, leaderboards, coaching)

**Aggregation jobs** (BullMQ, scheduled + triggered on new analysis) write rollup tables: per-rep daily/weekly stats, per-team stats, compliance-flag counts, score trends. Dashboards read only rollups — never raw call data — so a manager loading a dashboard is a cheap indexed read regardless of call volume.

**Leaderboards** rank reps within a team/tenant by configurable metric (score avg, calls handled, improvement rate). Cache in Redis with short TTL.

**Coaching module.** After each analyzed call, a job generates rep-facing feedback (strengths, one improvement focus, suggested phrasing) from the analysis output — reusing existing analysis, not re-analyzing. Feedback links to learning material via a tags table (e.g., a call flagged weak on objection handling links to the objection-handling module). Tracks per-rep progress: focus areas over time, whether scores in those areas improve.

### 3.4 Live pipeline (built last)

Browser mic → WebSocket → NestJS gateway → streaming STT → rolling transcript buffer → analysis job every 10–15s (sliding ~30s window) → compliance/sentiment via the **same analysis engine** in "live mode" (deterministic rules + fast LLM only; no deep scoring mid-call) → alert pushed back on the same socket. On call end, the accumulated transcript feeds the normal batch pipeline for full scoring, so a live call ends up in dashboards identically to an uploaded one.

Live-specific rules: tenant resolved at socket-open; DB connections acquired briefly per write, never pinned to the socket; concurrent-session cap + per-tenant minute budget enforced at socket-open; Redis Socket.io adapter for multi-instance.

### 3.5 Access control (cross-cutting, built first)

Backend-enforced RBAC in NestJS guards on every endpoint and socket event. Minimum roles: `TENANT_ADMIN`, `MANAGER`, `SALES_REP`. Enforcement is two-dimensional — role (what actions) and scope (whose data): a rep can read only `calls.rep_id = self`; a manager only their team's rows; every query filtered by `tenantId` at the repository layer, never trusted from the client. Frontend route guards remain for UX only. The two "different screens" are one React app rendering role-appropriate routes; the API guarantees a rep physically cannot fetch team data even with hand-crafted requests.

### 3.6 Platform services

- **Storage:** Postgres (all structured data), S3-compatible object storage (audio), Redis (cache, BullMQ, socket adapter, session counters).
- **Logging:** structured JSON to a central store (Loki/CloudWatch/Datadog), every line tagged `tenantId`, `callId`/`sessionId`, `requestId`. Replaces the local combined log file before anything else ships.
- **Cost guardrails:** per-tenant monthly AI budget (STT minutes + LLM tokens) tracked in Redis counters, enforced at job-enqueue time; alerts at 80%.

---

## 4. Core data model (summary)

| Table | Purpose | Key fields |
|---|---|---|
| `tenants` | Customer orgs | id, name, settings (retention, budgets, weights) |
| `users` | All users | id, tenant_id, role, team_id |
| `teams` | Team grouping | id, tenant_id, manager_id |
| `calls` | One row per call | id, tenant_id, rep_id, source (upload/live), audio_url (nullable), status, duration |
| `transcript_segments` | Diarized utterances | call_id, speaker, start_ms, end_ms, text, sentiment |
| `call_analyses` | One per analyzed call | call_id, score, score_breakdown (jsonb), sentiment_arc, summary |
| `compliance_rules` | Tenant-defined rules | tenant_id, type (deterministic/llm), definition, version, active |
| `compliance_flags` | Raised flags | call_id, rule_id, rule_version, segment_ref, severity, status (open/reviewed) |
| `coaching_feedback` | Rep-facing feedback | call_id, rep_id, strengths, focus_area, material_ids |
| `learning_materials` | Content library | tenant_id, title, url/content, tags |
| `rollup_rep_daily` / `rollup_team_daily` | Dashboard aggregates | rep/team_id, date, calls, avg_score, flags, sentiment_avg |
| `live_sessions` | Live call sessions | id, tenant_id, rep_id, started_at, minutes_used, status |

Every tenant-owned table carries `tenant_id` with a composite index; repository layer injects the tenant filter automatically.

---

## 5. Build timeline

Assumes a team of 2–4 engineers. Each phase ends with a working, demoable increment and explicit acceptance criteria. Phases are sequential because each depends on the previous; within a phase, frontend and backend tracks can run in parallel.

### Phase 0 — Foundations & security hardening (Weeks 1–2)

The unglamorous phase that prevents every later phase from being painful.

Build: backend RBAC guards (roles + scope enforcement) replacing frontend-only guards as the security boundary; tenant/user/team data model and migrations; repository layer with automatic tenant filtering; structured JSON logging to a central store; Redis + BullMQ wiring; object storage bucket with tenant-prefixed paths; per-tenant settings scaffold; CI with a smoke-test suite.

Acceptance: a rep JWT calling a manager endpoint gets 403 (proven by automated test); a query missing a tenant filter fails a lint/test rule; logs for a synthetic request are findable in the central store by requestId.

### Phase 1 — Recording ingestion & transcription (Weeks 3–5)

Build: upload endpoint (chunked/multipart for large files) → object storage; BullMQ transcription worker calling batch STT with diarization; transcript_segments persistence; call status lifecycle (uploaded → transcribing → transcribed → failed) with retries; basic call list + transcript viewer UI (both roles, scope-filtered); audio retention setting honored (delete-after-transcribe path works).

Acceptance: upload a 30-minute recording; within minutes see a diarized transcript in the UI; kill the worker mid-job and see the job retry and complete; a rep sees only their own calls.

### Phase 2 — Analysis engine: scoring, sentiment, compliance (Weeks 6–9)

The heart of the product. Build: sentiment per segment + call arc; deterministic compliance rule engine with tenant rule admin UI (CRUD, versioning); LLM compliance layer (JSON-schema-validated, "needs review" fallback); scoring rubric with computable metrics (talk ratio, question rate) + LLM-judged dimensions + tenant-configurable weights; call detail page showing score with full breakdown, sentiment arc, and flags linked to transcript moments; AI budget counters enforced at enqueue.

Acceptance: an analyzed call shows a score whose breakdown a human can verify against the transcript; adding a banned phrase to tenant rules flags a re-analyzed call at the exact utterance; malformed LLM output produces "needs review", never a fabricated score.

### Phase 3 — Dashboards, leaderboards & role views (Weeks 10–12)

Build: rollup aggregation jobs (triggered + nightly reconciliation); manager dashboard (team score trends, call volume, compliance flag queue with review workflow, rep comparison); leaderboard (configurable metric, Redis-cached); salesperson home (my scores, my trend, my recent calls, my flags); the full role-split navigation.

Acceptance: manager dashboard loads under 500ms with 10k calls in the tenant (reads rollups only — verified by query logs); leaderboard updates within one aggregation cycle of a new analysis; a rep's view contains zero team-level data in any API response (checked, not assumed).

### Phase 4 — Coaching module (Weeks 13–15)

Build: post-analysis feedback generation job (strengths, one focus area, suggested phrasing); learning material library + tagging admin; feedback→material linking; rep progress tracking (focus areas over time, improvement per area); manager view of team coaching status (who is improving, who is stuck).

Acceptance: after a call is analyzed, the rep sees feedback linked to at least one relevant material; a rep's progress view shows focus-area history across multiple calls; managers see coaching engagement without seeing another team's data.

### Phase 5 — Live real-time analysis (Weeks 16–20)

Built last, deliberately: it reuses the Phase 2 engine and Phase 3 UI, and it is the highest-risk component. Build: frontend mic capture + WebSocket client + live coaching UI; NestJS WS gateway (auth + tenant at socket-open, session lifecycle); streaming STT integration; rolling buffer + 10–15s sliding-window analysis jobs (live mode: deterministic rules + fast LLM only); alert push on the same socket; concurrent-session cap + minute budgets; Redis socket adapter for multi-instance; call-end handoff into the batch pipeline so live calls get full scoring and appear in dashboards.

Acceptance: speak a banned phrase into a live session and see an alert within ~15s; two backend instances behind a load balancer deliver alerts correctly (adapter proven); exceeding the tenant session cap rejects a new session with a clear client error; a finished live call appears in dashboards identically to an uploaded one.

### Phase 6 — Hardening & launch readiness (Weeks 21–22)

Load testing (target: 100 concurrent live sessions + batch throughput), security review (tenant isolation penetration checks, JWT/socket auth edge cases), data retention & deletion (tenant offboarding wipes audio, transcripts, analyses), monitoring dashboards + alerts (queue depth, STT/LLM error rates, budget burn), runbooks, pilot-tenant onboarding.

Acceptance: load test passes without cross-tenant leaks or alert delays beyond SLA; deleting a test tenant leaves zero rows/objects; on-call runbook exists for the top five failure modes.

### Timeline summary

| Phase | Weeks | Delivers |
|---|---|---|
| 0 | 1–2 | Security foundation, RBAC, logging, infra |
| 1 | 3–5 | Recordings → diarized transcripts |
| 2 | 6–9 | Scores, sentiment, compliance flags |
| 3 | 10–12 | Manager dashboards, leaderboards, rep view |
| 4 | 13–15 | Coaching & learning material |
| 5 | 16–20 | Live call analysis & alerts |
| 6 | 21–22 | Hardening, launch |

Total: ~22 weeks to full platform; a sellable batch-only product exists at the end of Phase 3 (Week 12).

---

## 6. Key risks and mitigations

**Diarization quality.** Speaker labels drive talk-ratio and per-speaker sentiment; poor diarization silently corrupts scores. Mitigate: evaluate STT vendors on *your* real call audio during Phase 1, not on demos; store confidence values; surface "low confidence" on affected calls.

**Score trust.** If reps feel scores are arbitrary, adoption dies. Mitigate: every score ships with its breakdown (Phase 2 acceptance), tenants tune weights, and a human review/appeal flag exists on `call_analyses`.

**LLM output instability.** Mitigate: strict JSON schemas, validation, "needs review" fallback, and pinned model versions per tenant with controlled upgrades.

**Cost blowout.** STT is billed per minute, LLM per token, and both scale with tenant success. Mitigate: budgets enforced at enqueue (Phase 2), live-session caps (Phase 5), and cheap models for classification — the expensive model only ever runs once per call, in batch.

**Tenant data isolation.** The single most damaging possible failure for a B2B compliance product. Mitigate: repository-level automatic tenant filtering (Phase 0), isolation tests in CI, and a penetration pass in Phase 6.

**Sequencing risk.** If live analysis is attempted before the analysis engine is stable, both suffer. The phase order above is a dependency chain — do not parallelize Phase 5 into Phase 2.

---

## 7. Explicit non-goals for v1

Telephony/meeting-platform integrations (Zoom, dialers) — v1 ingests uploads; integrations are a v2 ingestion adapter. Mobile apps. Multi-language transcription beyond the STT vendor's defaults. Custom ML model training. Cross-tenant benchmarking.
