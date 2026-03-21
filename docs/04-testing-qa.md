# Deliverable 4: Testing and QA Considerations

**System:** Task Management System — enterprise task platform with API, web UI, and notifications.

**Scope:** CI/CD setup, QA and test framework recommendations, and strategies to minimize manual QA effort while maintaining high quality.

---

## 1. CI/CD setup

### 1.1 Platform choice

| Option | Recommendation | Rationale |
|--------|----------------|-----------|
| **Azure DevOps Pipelines** | **Recommended** | Native Azure integration, App Service deployment tasks, variable groups for secrets, service connections, approval gates, and Azure-hosted agents. Aligns with Azure-centric setup in [Deliverable 2](02-cloud-setup.md). |

### 1.2 Pipeline stages

```text
  +------------------+     +------------------+     +------------------+     +------------------+
  |   Build          | --> |   Test           | --> |   Deploy (Dev)   | --> |   Deploy (Prod)  |
  |   restore,       |     |   unit,          |     |   App Service    |     |   approval +      |
  |   compile,       |     |   integration,   |     |   staging slot   |     |   slot swap       |
  |   publish        |     |   API contract   |     |                  |     |                  |
  +------------------+     +------------------+     +------------------+     +------------------+
```

| Stage | Steps | Triggers |
|-------|-------|----------|
| **Build** | `dotnet restore` → `dotnet build` → `dotnet publish` (e.g. `linux-x64` for App Service); produce artifact | Every push to main and PRs |
| **Test** | Run unit tests; run integration tests (with test DB); run API contract tests; publish coverage | Every push and PR |
| **Deploy (Dev)** | Deploy to Dev App Service slot; run smoke tests (health, basic GET) | On merge to main |
| **Deploy (Prod)** | Deploy to staging slot; optional manual approval; run smoke tests; swap slots | On approval or scheduled |

### 1.3 Pipeline configuration (conceptual)

**Build matrix:** One pipeline for the core monolith; separate pipelines (or jobs) for the Notification microservice and SPA if they live in the same repo or are split.

**Secrets:** Use Azure Key Vault or pipeline variable groups; never commit connection strings or API keys.

**Deployment strategy:** Blue-green via App Service deployment slots (staging → production swap). Rollback by swapping back.

**Branching:** Trunk-based (main) with short-lived feature branches; PRs required; main is always deployable.

---

## 2. QA and test setup

### 2.1 Test pyramid

| Layer | Purpose | Volume | Framework (ASP.NET Core) |
|-------|---------|--------|--------------------------|
| **Unit** | Domain logic, event handlers, projections, validation in isolation | High | **xUnit** + **FluentAssertions** + **NSubstitute** (or Moq) |
| **Integration** | API + DB (in-memory or test container), Service Bus (emulator or stub) | Medium | **xUnit** + **WebApplicationFactory** + **Testcontainers** (PostgreSQL, Redis) |
| **API contract** | Verify OpenAPI spec vs implemented behavior; catch breaking changes | Low | **Pact** (consumer-driven) or **OpenAPI-based contract tests** |
| **E2E** | Critical user flows (e.g. create task, assign, comment) | Low | **Playwright** (or Cypress) for SPA + API |
| **Performance** | Baseline latency and throughput under load | Occasional | **Azure Load Testing** (or k6 for scripting flexibility) |

### 2.2 Recommended frameworks

| Concern | Framework | Rationale |
|---------|-----------|-----------|
| **Unit/integration** | **xUnit** | Widely used, parallel by default, good ASP.NET support. |
| **Assertions** | **FluentAssertions** | Readable assertions; good error messages. |
| **Mocks** | **NSubstitute** (or **Moq**) | Lightweight; easy to configure. |
| **Integration DB** | **Testcontainers** | Real PostgreSQL/Redis in Docker; no shared test DB drift. **Requires Docker** — use in-memory providers or dedicated test DB if Docker is unavailable (e.g. some CI agents). |
| **API integration** | **WebApplicationFactory** | In-process host; no network; full middleware stack. |
| **API contract** | **NSwag** + custom asserts or **Pact**.NET | OpenAPI-first or consumer-driven; fail build on drift. |
| **E2E (SPA)** | **Playwright** | Cross-browser; fast; built-in auto-wait; API mocking. |
| **Load** | **Azure Load Testing** | Azure-native; JMeter or URL-based; integrates with Application Insights. k6 for scriptable flexibility. |

### 2.3 What to test

| Area | Focus |
|------|-------|
| **Event sourcing** | Event handlers apply correctly; projection matches replayed state; concurrency (optimistic locking). |
| **API** | Happy path for CRUD; validation (400); auth (401/403); rate limit (429); idempotency. |
| **Notifications** | Outbox → queue → delivery path; retries and DLQ behavior. |
| **Security** | Input validation, auth bypass attempts, SQL injection (via parameterized queries + fuzzing). |

---

## 3. Reducing manual QA effort (while maintaining high quality)

### 3.0 QA involvement early (shift-left for people)

**QA participates from the start**, instead of waiting until the task is done to begin testing.

| When | QA involvement |
|------|-----------------|
| **Backlog refinement** | QA joins refinement; helps define acceptance criteria, edge cases, and test scenarios before dev starts. |
| **Sprint planning** | QA reviews stories for testability; identifies missing scenarios; can start test-case design or automation in parallel with development. |
| **Development** | QA pairs with dev on complex flows; reviews PRs for test coverage; automation (E2E, API) is built alongside the feature. |
| **Definition of Done** | "QA reviewed" or "QA sign-off" is part of Done — not a separate phase after dev finishes. |

**Benefit:** Fewer surprises at the end; acceptance criteria are testable from day one; QA and dev work in parallel rather than sequentially.

---

Manual QA is reduced by automating regression; quality is maintained through gates, coverage, and targeted manual work where automation cannot cover.

| How we reduce manual effort | How we maintain high quality |
|----------------------------|------------------------------|
| Automate regression (unit, integration, contract, E2E) | PR gate: no merge without green tests; coverage threshold on changed code |
| Run full suite on every PR — no manual "run all tests" | E2E after deploy to Dev — critical flows must pass before prod |
| Reserve manual QA for exploratory, UAT, compliance | Contract tests — API drift breaks build; flaky tests quarantined and fixed |
| Shift-left: lint, format, tests before merge | Quality gates in pipeline — fail fast, no silent regressions |

### 3.1 Shift-left automation

| Practice | Implementation |
|----------|-----------------|
| **Pre-commit hooks** | Linting (e.g. ESLint, stylecop) and formatting (e.g. Prettier, dotnet format). |
| **PR gate** | All tests (unit, integration, contract) must pass; coverage threshold (e.g. ≥70% on new code); no merge without green. |
| **Fail fast** | Run fastest tests first; fail pipeline on first failure for local feedback. |
| **Parallelization** | Run unit and integration in parallel where possible; split by project or layer. |

### 3.2 Automated regression

| Practice | Implementation |
|----------|-----------------|
| **Full test suite on every PR** | No manual “run all tests”; CI runs them on every change. |
| **E2E on main** | Run Playwright E2E after deploy to Dev; fail deploy if critical flows break. |
| **Visual regression** | Optional: Playwright screenshots or Percy for UI changes; catch layout breaks. |
| **Contract tests** | OpenAPI or Pact; clients and API stay in sync; break build on drift. |

### 3.3 Targeted manual QA

| Effort | Scope |
|--------|-------|
| **Exploratory** | New features, UX flows, edge cases; time-boxed sessions. |
| **UAT** | Business sign-off on releases; predefined scenarios, not regression. |
| **Compliance / accessibility** | Manual audit for WCAG, keyboard nav, screen readers; automate where possible (e.g. axe-core). |

### 3.4 Quality gates (summary)

```text
  Commit → Lint / Format → Unit Tests → Integration Tests → Contract Tests
                                                                  |
                                                                  v
  Merge to main → Deploy Dev → Smoke Tests → E2E (Playwright) → Deploy Prod (with approval)
```

**Coverage:** Enforce minimum coverage (e.g. 70%) on modified code; use diff coverage to avoid blocking on legacy.

**Flaky tests:** Quarantine flaky tests; fix or remove within a sprint; never leave them in main pipeline.

---

## 4. AI-assisted workflows

Teams can use **AI coding assistants** (e.g. Azure OpenAI, Copilot in Azure DevOps) to accelerate the three areas in this deliverable. AI acts as a co-author: it drafts and suggests; humans review and decide.

| Area | How AI helps |
|------|---------------|
| **CI/CD setup** | Generate or refine Azure DevOps pipeline YAML from a short description; troubleshoot build and deploy errors; suggest stages, parallelization, and secret handling. |
| **QA and test setup** | Compare frameworks (xUnit vs NUnit, Playwright vs Cypress); scaffold test projects, `WebApplicationFactory`, and Testcontainers wiring; generate unit and integration tests from API signatures or domain code. |
| **Reducing manual QA** | Map manual steps to automated tests; suggest PR gates and coverage thresholds; propose ways to avoid flaky tests and brittle automation. |

**Example prompts:**
- "Generate an Azure DevOps pipeline for ASP.NET Core: build, test, deploy to App Service with a staging slot."
- "Generate integration tests for this controller using WebApplicationFactory and Testcontainers for PostgreSQL."
- "List 5 ways to reduce manual regression testing for a REST API and SPA."

**Principle:** AI accelerates drafting and implementation. Design decisions (what to test, when to deploy, quality thresholds) remain human-owned.

---

## 5. Summary

| Area | Approach |
|------|----------|
| **CI/CD** | Azure DevOps Pipelines; build → test → deploy; App Service slots for blue-green; secrets from Key Vault. |
| **Unit** | xUnit + FluentAssertions + NSubstitute; focus on domain, event handlers, projections. |
| **Integration** | WebApplicationFactory + Testcontainers (PostgreSQL, Redis); full API + DB flows. |
| **Contract** | OpenAPI or Pact; fail build on API drift. |
| **E2E** | Playwright; critical user flows after Dev deploy. |
| **Manual QA** | Exploratory and UAT only; regression automated in CI. |
| **QA early involvement** | QA in refinement and planning; acceptance criteria co-authored; QA sign-off part of Definition of Done. |
| **Reducing manual effort** | PR gates, full regression in CI, E2E on main, contract tests; manual reserved for exploration and compliance. |
| **AI assistance** | Use AI coding assistants to draft pipelines, scaffold tests, and propose automation strategies; humans review and own design decisions. |
