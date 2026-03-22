# Deliverable 5: Development Process & Delivery Speed

**System:** Task Management System — enterprise task platform with API, web UI, and notifications.

**Scope:** Development process setup, techniques to speed up delivery 2–3x (or more) without overtime, measurement of impact, project roadmap, and risk management.

---

## 1. Development process setup

### 1.1 Process model

| Choice | Recommendation | Rationale |
|--------|----------------|-----------|
| **Methodology** | **Scrum** (2-week sprints) with Kanban for flow visibility | Predictable cadence; clear sprint goal; backlog refinement; retrospectives for improvement. Kanban board for real-time status. |
| **Planning** | Sprint planning (day 1); refinement mid-sprint; daily stand-ups (15 min) | Keeps scope clear; reduces context switching; surfaces blockers early. |
| **Definition of Ready** | Story has acceptance criteria; dependencies identified; QA and dev have refined; estimated and prioritized | Prevents pulling unrefined work into the sprint. |
| **Definition of Done** | Code complete, tests passing, PR reviewed, QA sign-off, deployed to Dev, docs updated | Prevents “almost done” work from piling up; QA is part of Done, not a phase after dev. |
| **Backlog** | Epics → Stories → Tasks; prioritized by value and risk | Enables incremental delivery; dependencies visible early. |

### 1.2 Tooling

| Need | Tool | Purpose |
|------|------|---------|
| **Work tracking** | Azure DevOps Boards | Backlog, sprints, burndown, dependency tracking. |
| **Code** | Azure Repos (Git) | Version control; branch per feature; PRs for review. |
| **CI/CD** | Azure Pipelines | Build, test, deploy; see [Deliverable 4](04-testing-qa.md). |
| **Docs** | Azure DevOps Wiki | Architecture decisions, runbooks, API docs; Markdown in repo. |
| **Communication** | Microsoft Teams | Stand-ups, async updates, alerts. |

### 1.3 Team structure and flow

**Assumed team:** 1 leader (Tech Lead / Scrum Master), 7 developers, 3 QA — 10 people total. The leader owns priorities, blockers, phasing, and risk; devs and QA work in parallel with shift-left quality.

```text
  Backlog → Refinement (QA + dev) → Sprint Planning → Development (QA in parallel) → PR Review → Merge → Deploy
       ↑       QA defines acceptance criteria, test scenarios                               |
       +—— Retrospective, metrics review —— feedback —— backlog adjustment ←——------------+
```

**QA early involvement:** QA participates in refinement and planning — defines acceptance criteria and test scenarios before dev starts. QA works in parallel during the sprint (test-case design, automation, exploratory) instead of waiting until the task is done to begin testing. See [Deliverable 4, §3.0](04-testing-qa.md#30-qa-involvement-early-shift-left-for-people).

**Branching:** Trunk-based with short-lived feature branches. PR required; main always deployable.

**Reviews:** At least one approval; optional: pair programming for complex changes. Linters and tests run on PR; no merge until green.

### 1.4 Team allocation (7 devs + 3 QA)

| Phase | Devs (7) | QA (3) |
|-------|----------|--------|
| **Phase 1** | 2–3: event store, projection, task CRUD, GET /history. 1–2: comments, attachments. 1: auth, API skeleton. 1: CI/CD, local dev env. | 1: refinement + acceptance criteria. 1: API test automation. 1: exploratory + integration tests. |
| **Phase 2 (parallel)** | Team A (4): Phase 1 completion. Team B (3): Notification service + SignalR (against event contract). | 1: SignalR tests. 1: Notification flow tests. 1: E2E + exploratory. |
| **Phase 3** | 4–5: Web UI (SPA). 2: scheduler, prod pipeline, monitoring. | 3: E2E, UAT support, accessibility. |

**Speed-up with this team:** Use parallel delivery (§4.1.1): define event contract in week 1; 4 devs on Phase 1 core, 3 on Notification + SignalR. Limit WIP per person (e.g. 2 items); swarm on blockers. QA embedded per stream, not a bottleneck at the end.

---

## 2. Speeding up delivery 2–3x (or more) without overtime

**Target outcome:** We aim to **reduce cycle time by ~50%** (e.g. 6 days → 3 days from story start to Done) and **increase throughput by ~2x** (e.g. 24 → 40+ points per sprint) within **3–6 months** of adopting the techniques below. Measured via DORA-inspired metrics (Section 3): cycle time, deployment frequency, lead time for changes, and change failure rate.

Speed-up comes from **removing waste**, **parallelizing work**, and **automating** — not from longer hours.

**Parallel working approach:** Define the event contract in week 1, then run two streams in parallel. Team A (4 devs) builds Phase 1 core (event store, CRUD, comments, attachments); Team B (3 devs) builds the Notification service and SignalR against that contract. Both progress independently and integrate when the write path is ready; QA is split per stream. See [§1.4](05-development-process.md#14-team-allocation-7-devs--3-qa) and [§4.1.1](05-development-process.md#411-parallel-delivery-fast-timeline-multiple-teams).

### 2.1 Techniques and tools

*Expected gains below are **illustrative**; validate against DORA metrics in Section 3.*

| Lever | Approach | Expected gain |
|-------|----------|---------------|
| **Reduce cycle time** | CI/CD (build < 10 min, deploy to Dev on merge); fail fast | 30–50% fewer “waiting for build/deploy” blocks. |
| **Parallelize features** | Feature flags; slice stories into independent pieces; avoid big-branch merges; parallel delivery (§1.4, §4.1.1) | 1.5–2x throughput with 7 devs split across Phase 1 core and Notification + SignalR streams. |
| **Eliminate rework** | Test-first; PR reviews; clear acceptance criteria; contract tests | Fewer bugs in prod; less back-and-forth; 20–40% less rework. |
| **Reduce manual QA** | Automated regression (unit, integration, E2E); see [Deliverable 4](04-testing-qa.md) | QA focuses on exploratory; release cycles shrink. |
| **Faster feedback** | Local dev with Docker Compose or dev containers; Testcontainers for integration | Devs don’t wait for shared DB; fewer “works on my machine” issues. |
| **AI-assisted coding** | Azure OpenAI (or Copilot in Azure DevOps) for scaffolding, tests, boilerplate | 20–40% faster for repetitive tasks (CRUD, tests, pipeline YAML). |
| **Reduce meetings** | Async stand-up notes; short sync only when needed; “no-meeting” blocks for deep work | Reclaim 5–10+ hours per dev per sprint. |
| **Reduce context switching** | Swarm on high-priority items; limit WIP per person; clear sprint focus | Fewer half-done items; better flow. |
| **Templates and generators** | OpenAPI → client SDK; scaffold for controllers, events, tests | Less boilerplate; consistent patterns. |

### 2.2 Prioritization: what to do first

| Priority | Action | Why |
|----------|--------|-----|
| 1 | **CI/CD** — automated build, test, deploy | Biggest cycle-time reduction; enables everything else. |
| 2 | **Automated tests** — unit, integration, key E2E | Reduces manual QA; catches regressions early. |
| 3 | **Feature flags** | Enables parallel work; incomplete features can merge behind flags. |
| 4 | **Local dev environment** — Docker Compose or dev containers | Faster iteration; fewer environment issues. |
| 5 | **AI tooling** | Low-effort adoption; immediate productivity lift. |

### 2.3 What we avoid (no overtime)

- **No hero culture** — sustainable pace; overtime is a signal of process or scope failure.
- **No skipping tests** — quality gates stay; we speed up by automating, not bypassing.
- **No “finish everything”** — scope to capacity; defer low-value work.

---

## 3. Measuring impact

### 3.1 Metrics (DORA-inspired)

| Metric | Definition | Target | How to measure |
|--------|-------------|--------|----------------|
| **Deployment frequency** | How often we deploy to production | Increase (e.g. 2x per sprint → 1x per day) | Pipeline run history; release tags |
| **Lead time for changes** | Commit to production | Decrease (e.g. 5 days → 2 days) | Azure DevOps: story completion to release |
| **Mean time to recovery (MTTR)** | Incident detection to fix deployed | Decrease | Incident tracker; deployment logs |
| **Change failure rate** | % of deployments causing incidents | Low and stable (< 5%) | Post-deploy incident count / deployments |
| **Cycle time** | Story start to Done | Decrease | Board: in-progress → done duration |
| **Throughput** | Stories/points completed per sprint | Increase | Sprint velocity; burndown |
| **PR cycle time** | PR opened to merged | Decrease | Azure Repos: PR metrics |

### 3.2 Data sources

| Source | Data |
|--------|------|
| **Azure DevOps** | Cycle time, throughput, PR metrics, pipeline duration, burndown |
| **Azure Repos** | PR merge time, code review stats |
| **Application Insights** | Post-deploy error rates; correlate to release |
| **Azure Monitor** | Incidents, deployment logs |
| **Retrospectives** | Qualitative feedback: “what slowed us down?” |

### 3.3 Baseline and comparison

1. **Baseline (sprint 0 or 1):** Record current cycle time, throughput, deployment frequency, PR cycle time.
2. **After each change:** Re-measure; compare to baseline.
3. **Review in retro:** “We added CI; cycle time went from 6 days to 4 days” — quantify, adjust.

**Example dashboard (conceptual):**

```text
  Metric               | Baseline (Sprint 1) | Current (Sprint 4) | Change
  ---------------------|---------------------|--------------------|-------
  Cycle time (days)    | 6                   | 3                  | -50%
  Throughput (pts)     | 24                  | 38                 | +58%
  Deploy frequency     | 2/sprint            | 8/sprint           | 4x
  PR cycle (hours)     | 18                  | 6                  | -67%
```

---

## 4. Project roadmap and risk management

### 4.1 High-level roadmap

**Cloud tier mapping:** Phase 1–3 align with cost tiers Start, Transition, Growth in [Deliverable 2, §4](02-cloud-setup.md#4-summary-of-monthly-cost-and-setup-tiers) — use those for infrastructure sizing when you scale. Phases here are feature-based, not user-count-based. **Weeks 1–12 are illustrative**; adjust for real portfolio and resource availability. **Phasing rationale:** Event sourcing ships first in the monolith; SignalR and Notification microservice are added in Phase 2. **When does the business see a clickable product?** Phase 1 delivers API + Swagger; external integrators and internal tools can use it. A minimal UI slice (task list + create) is available as an optional Phase 1 item for early demos. The full web UI (SPA) ships in Phase 3. See [Deliverable 1, §1.5](01-solution-architecture.md#15-decision-best-option-for-this-system--option-b-macroservices-with-core--notification).

```text
  Phase 1 (Weeks 1–4):  Event sourcing in monolith (ship first)
  ├── CI/CD pipeline (build, test, deploy Dev)
  ├── Local dev environment (Docker Compose / dev container)
  ├── Event store + projection; task CRUD, GET /tasks/{id}/history
  ├── Comments, attachments (metadata + presigned URLs)
  ├── Basic auth (Azure AD integration)
  ├── Optional: minimal UI slice (task list + create) for early demo
  └── Automated tests (unit, integration)
  No SignalR, no Notification service — all in one deployable
  MVP demo: API + Swagger; optional minimal UI for early demos. Full web UI in Phase 3.

  Phase 2 (Weeks 5–8):  Real-time + notifications
  ├── Azure SignalR Service — in-view alerts when task is edited
  ├── Notification outbox → Service Bus → Notification microservice
  ├── Event handlers publish to queue and SignalR on append
  └── Key E2E tests (including SignalR, notification flow)

  Phase 3 (Weeks 9–12):  Polish and scale
  ├── Web UI (SPA) for core flows (SignalR integration for live updates)
  ├── Overdue / SLA scheduler
  ├── Production deploy pipeline, monitoring, runbooks
  └── Performance baseline, load tests

  Phase 4 (Ongoing):  Iterate
  ├── Feature flags, external API polish
  ├── Scale-out (read replicas)
  └── UAT, go-live, support
```

### 4.1.1 Parallel delivery (fast timeline, multiple teams)

**When to use:** Fast delivery is required and **multiple teams** are available. Phase 1 and Phase 2 can run in **parallel** after defining the event contract upfront.

**Dependency:** SignalR and the Notification service both need the event-sourcing write path (append event → update projection) to publish and push. Define the event contract (event types, message shapes) in **week 1** so teams can work independently.

**Parallel workstreams** (7 devs: 4 on Team A, 3 on Team B; 3 QA split across streams):

| Team / stream | Work | Can start |
|---------------|------|------------|
| **Team A (4 devs)** | Phase 1: event store, projection, task CRUD, GET /history, comments, attachments | Day 1 |
| **Team B (3 devs)** | Notification microservice, SignalR hub, client wiring. Test with mock publisher against contract. | Week 1 (after contract) |
| **QA (3)** | 1 per stream: refinement, automation, exploratory; plus cross-cutting E2E when integrated | Day 1 / Week 1 |

**Integration (sequential):** When Phase 1’s write path is ready, add in event handlers: publish to Service Bus, push to SignalR. Small merge; typically 1–2 days.

**Timeline (example):**

```text
  Week 1:  Contract (TaskCreated, TaskAssigned, StatusChanged, etc.)
           Team A: Phase 1 (event store, projection, CRUD)
           Team B: Notification microservice (mock publisher)
           Team B/C: SignalR hub + client

  Weeks 2–3:  Teams continue in parallel; Phase 1 write path completes

  Week 4:    Integration: wire publish + push into event handlers
             E2E verification
```

**Prerequisites:** Contract agreed in sprint 0 or week 1; mock/stub tooling for independent testing. With 7 devs + 3 QA, split as above; see §1.4 for full allocation.

### 4.2 Risk identification

**Prioritization:** Act first on High likelihood × High impact; then High × Medium. Use risk score (likelihood × impact) to order mitigation effort.

| Risk | Likelihood | Impact | Category |
|------|------------|--------|----------|
| **Scope creep** | High | High | Planning |
| **Key person dependency** | Medium | High | People |
| **Event contract drift** (Team A/B parallel delivery) | Medium | High | Technical |
| **Parallel integration surprises** (merge/integration when Team A + B join) | Medium | Medium | Technical |
| **Integration delays** (Azure AD, Service Bus, etc.) | Medium | Medium | External |
| **Unclear requirements** | High | Medium | Requirements |
| **Technical debt** (rushed code) | Medium | Medium | Technical |
| **Environment issues** (DB, secrets, networking) | Medium | Medium | Operations |
| **Vendor/third-party delays** | Low | High | External |
| **Regression from speed push** | Medium | High | Quality |

### 4.3 Mitigation strategies

| Risk | Mitigation |
|------|------------|
| **Scope creep** | Hard sprint capacity; backlog prioritization; “no” is acceptable; defer low-value items. |
| **Key person dependency** | Pair programming; docs and runbooks; cross-training; shared ownership. With 7 devs, bus factor is low — spread critical knowledge. |
| **Event contract drift** | Freeze contract in week 1; change-control: any change requires both Team A and Team B lead approval; document in OpenAPI or shared schema. |
| **Parallel integration surprises** | Integration spike in week 2–3 (stub only); reserve 1–2 days buffer for merge; test against contract in both streams. |
| **Integration delays** | Stub/mock external services in dev; start integration early; parallel work on UI and API. |
| **Unclear requirements** | Refinement before sprint; spike or proof-of-concept for unknowns; clarify with stakeholders early. |
| **Technical debt** | Allocate 10–15% sprint capacity to tech debt; no skipping tests; refactor incrementally. |
| **Environment issues** | Document setup; automate with scripts; use Infrastructure as Code where possible; Testcontainers for tests. |
| **Vendor delays** | Identify dependencies early; have fallback or mock; buffer in timeline for external blockers. |
| **Regression from speed** | Quality gates stay; automated tests are non-negotiable; speed comes from automation, not shortcuts. |

### 4.4 Risk management process

With 1 leader (Tech Lead / Scrum Master), they own risk review and escalation. All tooling is Azure-based: Azure DevOps (Boards, Repos, Pipelines, Wiki), Teams, Application Insights, Azure Monitor.

| Activity | Frequency | Owner |
|----------|-----------|-------|
| **Risk review** | Every sprint planning | Tech Lead (leader) |
| **Update risk log** | As new risks emerge | Anyone; Tech Lead consolidates in Azure DevOps Wiki |
| **Mitigation check** | Mid-sprint stand-up + retrospective | “Is our mitigation working?” |
| **Steering summary** | Bi-weekly / monthly steering | High-impact risks summarized for stakeholders |
| **Escalation** | When: 2+ sprints behind; critical path blocked >1 sprint; High-impact risk materializes | Tech Lead → Product Owner → Stakeholders |

**Risk log format and example:**

| Risk | Likelihood | Impact | Mitigation | Owner | Status | Next review |
|------|------------|--------|------------|-------|--------|-------------|
| Scope creep threatens Phase 1 timeline | High | High | Hard sprint capacity; defer low-value items; PO says "no" | Tech Lead | Active | Sprint 2 planning |
| Key person dependency on auth module | Medium | High | Pair programming; document runbook; cross-train | Tech Lead | Mitigating | Mid-sprint 2 |

**Status lifecycle:** Active → Mitigating → Resolved (or Accepted, if we consciously accept the risk).

### 4.5 Stakeholder and governance (PO lens)

| Practice | Recommendation |
|----------|-----------------|
| **Sprint review demos** | Demo working software every sprint; Product Owner accepts or rejects; backlog adjusted from feedback. |
| **Steering cadence** | Bi-weekly or monthly steering with sponsors/stakeholders; show roadmap progress, risks, and decisions needed. |
| **Go-live criteria** | Explicit checklist before production: SLO thresholds met (e.g. p95 latency < 500ms, error rate < 0.1%); UAT sign-off; rollback owner assigned; runbook validated. |
| **Rollback owner** | Tech Lead or on-call owns rollback decision and execution; document in runbook; practice slot-swap rollback in staging. |

---

## 5. Summary

| Area | Approach |
|------|----------|
| **Process** | Scrum (2-week sprints), trunk-based branching, PRs, Definition of Done. |
| **Speed-up (no overtime)** | CI/CD, automated tests, feature flags, parallel work, AI tooling, reduce meetings and context switching. |
| **Measurement** | DORA-inspired metrics (cycle time, throughput, deploy frequency, PR cycle); baseline and track changes. |
| **Roadmap** | Foundation → Core features → Polish → Iterate; phases of 4 weeks. |
| **Risk management** | Prioritize High×High; review each sprint; Tech Lead owns; log in Azure DevOps Wiki; escalate when 2+ sprints behind or critical path blocked. |
| **Stakeholder / governance** | Sprint review demos; steering cadence; explicit go-live criteria (SLO, UAT, rollback owner). |
