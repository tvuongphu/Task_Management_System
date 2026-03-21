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

```text
  Backlog → Refinement (QA + dev) → Sprint Planning → Development (QA in parallel) → PR Review → Merge → Deploy
       ↑       QA defines acceptance criteria, test scenarios                               |
       +—— Retrospective, metrics review —— feedback —— backlog adjustment ←——------------+
```

**QA early involvement:** QA participates in refinement and planning — defines acceptance criteria and test scenarios before dev starts. QA works in parallel during the sprint (test-case design, automation, exploratory) instead of waiting until the task is done to begin testing. See [Deliverable 4, §3.0](04-testing-qa.md#30-qa-involvement-early-shift-left-for-people).

**Branching:** Trunk-based with short-lived feature branches. PR required; main always deployable.

**Reviews:** At least one approval; optional: pair programming for complex changes. Linters and tests run on PR; no merge until green.

---

## 2. Speeding up delivery 2–3x (or more) without overtime

Speed-up comes from **removing waste**, **parallelizing work**, and **automating** — not from longer hours.

### 2.1 Techniques and tools

| Lever | Approach | Expected gain |
|-------|----------|---------------|
| **Reduce cycle time** | CI/CD (build < 10 min, deploy to Dev on merge); fail fast | 30–50% fewer “waiting for build/deploy” blocks. |
| **Parallelize features** | Feature flags; slice stories into independent pieces; avoid big-branch merges | 1.5–2x throughput when 2–3 devs work in parallel without stepping on each other. |
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
  Metric              | Baseline (Sprint 1) | Current (Sprint 4) | Change
  --------------------|--------------------|--------------------|-------
  Cycle time (days)   | 6                  | 3                  | -50%
  Throughput (pts)    | 24                 | 38                 | +58%
  Deploy frequency    | 2/sprint           | 8/sprint           | 4x
  PR cycle (hours)    | 18                 | 6                  | -67%
```

---

## 4. Project roadmap and risk management

### 4.1 High-level roadmap

**Cloud tier mapping:** Phase 1 → Start (~1k users); Phase 2–3 → Transition/Growth (~2k–10k). See [Deliverable 2, §4](02-cloud-setup.md#4-summary-of-monthly-cost-and-setup-tiers) for cost tiers.

```text
  Phase 1 (Weeks 1–4):  Foundation
  ├── CI/CD pipeline (build, test, deploy Dev)
  ├── Local dev environment (Docker Compose / dev container)
  ├── Core API skeleton + event store + projection
  └── Basic auth (Azure AD integration)

  Phase 2 (Weeks 5–8):  Core features
  ├── Task CRUD, assignments, status, history (event sourcing)
  ├── Comments, attachments (metadata + presigned URLs)
  ├── Notification outbox → Service Bus → Notification service
  └── Automated tests (unit, integration, key E2E)

  Phase 3 (Weeks 9–12):  Polish and scale
  ├── Web UI (SPA) for core flows
  ├── Overdue / SLA scheduler
  ├── Production deploy pipeline, monitoring, runbooks
  └── Performance baseline, load tests

  Phase 4 (Ongoing):  Iterate
  ├── Feature flags, external API polish
  ├── Scale-out (read replicas, Notification microservice)
  └── UAT, go-live, support
```

### 4.2 Risk identification

| Risk | Likelihood | Impact | Category |
|------|------------|--------|----------|
| **Scope creep** | High | High | Planning |
| **Key person dependency** | Medium | High | People |
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
| **Key person dependency** | Pair programming; docs and runbooks; cross-training; shared ownership of critical areas. |
| **Integration delays** | Stub/mock external services in dev; start integration early; parallel work on UI and API. |
| **Unclear requirements** | Refinement before sprint; spike or proof-of-concept for unknowns; clarify with stakeholders early. |
| **Technical debt** | Allocate 10–15% sprint capacity to tech debt; no skipping tests; refactor incrementally. |
| **Environment issues** | Document setup; automate with scripts; use Infrastructure as Code where possible; Testcontainers for tests. |
| **Vendor delays** | Identify dependencies early; have fallback or mock; buffer in timeline for external blockers. |
| **Regression from speed** | Quality gates stay; automated tests are non-negotiable; speed comes from automation, not shortcuts. |

### 4.4 Risk management process

| Activity | Frequency | Owner |
|----------|-----------|-------|
| **Risk review** | Every sprint planning | Scrum Master / Tech Lead |
| **Update risk log** | As new risks emerge | Anyone; log in backlog or wiki |
| **Mitigation check** | Mid-sprint | Stand-up; “is our mitigation working?” |
| **Escalation** | When impact threatens timeline | Tech Lead → Product Owner → Stakeholders |

All tooling is Azure-based: Azure DevOps (Boards, Repos, Pipelines, Wiki), Teams, Application Insights, Azure Monitor.

**Risk log (minimal):** Risk | Likelihood | Impact | Mitigation | Status

---

## 5. Summary

| Area | Approach |
|------|----------|
| **Process** | Scrum (2-week sprints), trunk-based branching, PRs, Definition of Done. |
| **Speed-up (no overtime)** | CI/CD, automated tests, feature flags, parallel work, AI tooling, reduce meetings and context switching. |
| **Measurement** | DORA-inspired metrics (cycle time, throughput, deploy frequency, PR cycle); baseline and track changes. |
| **Roadmap** | Foundation → Core features → Polish → Iterate; phases of 4 weeks. |
| **Risk management** | Identify risks; mitigate proactively; review each sprint; escalate when timeline is threatened. |
