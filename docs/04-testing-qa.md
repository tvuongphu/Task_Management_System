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
  +------------------+     +------------------+     +------------------+     +------------------+     +------------------+
  |   Build          | --> |   Test + Security| --> |   Deploy (Dev)    | --> | Deploy (Staging)  | --> | Deploy (Prod)    |
  |   restore,       |     |   unit, integ,   |     |   App Service     |     | QA/UAT slot       |     | approval +        |
  |   compile,       |     |   contract, SAST,|     |                   |     | smoke + a11y      |     | slot swap         |
  |   publish        |     |   dependency scan|     |                   |     |                   |     |                   |
  +------------------+     +------------------+     +------------------+     +------------------+     +------------------+
```

| Stage | Steps | Triggers |
|-------|-------|----------|
| **Build** | `dotnet restore` → `dotnet build` → `dotnet publish`; produce artifact | Every push to main and PRs |
| **Test + Security** | Unit, integration, contract tests; coverage; **dependency scan** (e.g. `dotnet list package --vulnerable`, Dependabot); **SAST** (e.g. Security Code Scan, SonarQube); container scan if Docker images are built | Every push and PR |
| **Deploy (Dev)** | Deploy to Dev App Service; smoke tests (health, basic GET) | On merge to main |
| **Deploy (Staging)** | Deploy to **Staging/QA slot** for UAT; run smoke + E2E; **automated accessibility** (axe + Playwright) | On merge to main or scheduled |
| **Deploy (Prod)** | Deploy to production slot; manual approval; smoke tests; slot swap | On approval or scheduled |

**Time targets:** Build < 10 min; deploy to Dev < 15 min end-to-end; deploy to Staging < 20 min.

### 1.3 Pipeline configuration (conceptual)

**Build matrix:** One pipeline for the core monolith; separate pipelines (or jobs) for the Notification microservice and SPA if they live in the same repo or are split.

**Secrets:** Use Azure Key Vault or pipeline variable groups; never commit connection strings or API keys.

**Deployment strategy:** Blue-green via App Service deployment slots. **Environments:** Dev (merge to main) → Staging/QA (UAT slot; business sign-off before prod) → Prod (approval + slot swap). Rollback by swapping back.

**Branching:** Trunk-based (main) with short-lived feature branches; PRs required; main is always deployable.

### 1.4 Sample pipeline (Azure DevOps YAML)

```yaml
# azure-pipelines.yml (simplified)
trigger:
  - main
  branches:
    include: [main]
pool:
  vmImage: ubuntu-latest
stages:
  - stage: Build
    jobs:
      - job: BuildAndTest
        steps:
          - task: UseDotNet@2
            inputs: { packageType: sdk, version: 8.x }
          - script: dotnet restore
          - script: dotnet build --no-restore
          - script: dotnet test --no-build --collect:"XPlat Code Coverage"
          - task: PublishPipelineArtifact@1
            inputs:
              targetPath: $(Build.ArtifactStagingDirectory)
              artifact: drop
              publishLocation: pipeline
  - stage: DeployDev
    dependsOn: Build
    jobs:
      - deployment: Deploy
        environment: dev
        strategy: runOnce
        deploy:
          steps:
            - task: AzureWebApp@1
              inputs:
                azureSubscription: $(serviceConnection)
                appName: $(appServiceName)
                package: $(Pipeline.Workspace)/drop
```

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
| **Unit/integration** | **xUnit** | Industry standard for .NET; runs tests in parallel by default (faster suites); no shared mutable state; strong ASP.NET Core integration. Alternative: NUnit is equally mature but uses different conventions. |
| **Assertions** | **FluentAssertions** | Readable, fluent syntax (e.g. `result.Should().NotBeNull().And.Subject.Status.Should().Be("Done")`); clear failure messages when assertions fail; reduces debugging time. Alternative: xUnit.Assert is sufficient but less expressive. |
| **Mocks** | **NSubstitute** (or **Moq**) | Simple, lightweight API; easy to configure return values and verify calls; widely adopted. Moq is equally popular; NSubstitute often has a slightly more natural syntax. |
| **Integration DB** | **Testcontainers** | Spins up real PostgreSQL/Redis in Docker per test or per assembly; no shared test DB, no drift, no manual setup; tests against the real database engine. **Requires Docker** — use in-memory providers (e.g. EF InMemory, SQLite) or a dedicated test DB if Docker is unavailable (e.g. some hosted CI agents). |
| **API integration** | **WebApplicationFactory** | Built into ASP.NET Core; hosts the app in-process with full middleware (auth, validation, etc.); no HTTP server or port binding; fast, isolated. Preferred over launching a real server for integration tests. |
| **API contract** | **NSwag** + custom asserts or **Pact**.NET | **OpenAPI/NSwag** when you have a single API and want spec-as-source-of-truth; validates implementation vs spec. **Pact** when multiple consumers (frontend, mobile, partners) integrate—consumer-driven contracts; each consumer defines expectations; provider tests against all. Both fail the build on drift. |
| **E2E (SPA)** | **Playwright** | Cross-browser (Chromium, Firefox, WebKit); fast execution; built-in auto-wait (reduces flakiness); can mock API calls; good for SPA + API flows. Microsoft-maintained; aligns with .NET ecosystem. Alternative: Cypress has a simpler model but is Chromium-centric by default. |
| **Load** | **Azure Load Testing** | Azure-native; integrates with Application Insights for correlation; supports JMeter and URL-based tests; no separate infra. Fits the Azure-centric setup. Alternative: k6 for more scriptable, code-based load tests. |

### 2.3 What to test

| Area | Focus |
|------|-------|
| **Event sourcing** | Event handlers apply correctly; projection matches replayed state; concurrency (optimistic locking). |
| **API** | Happy path for CRUD; validation (400); auth (401/403); rate limit (429); idempotency. |
| **Notifications** | Outbox → queue → delivery path; retries and DLQ behavior. |
| **Real-time (SignalR)** | Hub connection; join/leave task groups; message delivery when task is updated; reconnection handling. |
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

**How we reduce (or eliminate) manual QA — summary checklist:**

- **Automated regression** — Unit, integration, contract, and E2E tests run on every PR and merge; no manual "run all tests."
- **PR gate** — No merge without green tests and coverage threshold (e.g. ≥70% on new code); fail fast.
- **Contract tests** — OpenAPI or Pact; API drift breaks the build; clients stay in sync.
- **E2E on main** — Critical flows (create task, assign, comment) run after deploy to Dev; fail deploy if broken.
- **Automated accessibility** — axe-core in Playwright; fail build on critical a11y violations.
- **Manual reserved for** — Exploratory testing, UAT, and compliance audits only; regression is automated.

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
| **E2E on main** | Run Playwright E2E after deploy to Dev; fail deploy if critical flows break. **Auth in CI:** Playwright against real Entra ID is painful; use **test users** in a dev tenant, **client credentials** for API-only flows, or **mock auth** (e.g. bypass or stub token validation) in E2E to avoid interactive login. |
| **Visual regression** | Optional: Playwright screenshots or Percy for UI changes; catch layout breaks. |
| **Contract tests** | OpenAPI or Pact; clients and API stay in sync; break build on drift. |

### 3.3 Targeted manual QA

| Effort | Scope |
|--------|-------|
| **Exploratory** | New features, UX flows, edge cases; time-boxed sessions. |
| **UAT** | Business sign-off on releases; predefined scenarios, not regression. |
| **Compliance / accessibility** | Manual audit for WCAG, keyboard nav, screen readers. **Automated in CI:** Run **axe-core** (or @axe-core/playwright) in Playwright E2E tests for the SPA; fail build on critical a11y violations. |

### 3.4 Quality gates (summary)

| Gate | Implementation |
|------|----------------|
| **Lint** | stylecop, ESLint; fail on style violations. |
| **Format** | dotnet format, Prettier; enforce consistent formatting. |
| **Tests** | Unit, integration, contract; fail if any fail. |
| **Coverage** | ≥70% on new/modified code; use diff coverage for legacy. |
| **Security scans** | Dependency scan (vulnerable packages); SAST (Security Code Scan, SonarQube); container scan if applicable. |

```text
  Commit → Lint / Format → Unit / Integration / Contract Tests → Security Scans (dep + SAST)
                                                                          |
                                                                          v
  Merge → Deploy Dev → Smoke → E2E (Playwright) → Deploy Staging (UAT) → Deploy Prod (approval)
```

**Flaky tests:** Quarantine flaky tests; fix or remove within a sprint; never leave them in main pipeline.

**Security (beyond CI):** For the external API, run **annual penetration test** and **automated smoke security tests** (auth bypass, injection probes) in staging. DAST tools (e.g. OWASP ZAP) can run on schedule.

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
| **CI/CD** | Azure DevOps Pipelines; build → test → security scans → deploy; Dev → Staging (UAT) → Prod; App Service slots; secrets from Key Vault. |
| **Unit** | xUnit + FluentAssertions + NSubstitute; focus on domain, event handlers, projections. |
| **Integration** | WebApplicationFactory + Testcontainers (PostgreSQL, Redis); full API + DB flows. |
| **Contract** | OpenAPI or Pact; fail build on API drift. |
| **E2E** | Playwright; critical user flows after Dev deploy. |
| **Manual QA** | Exploratory and UAT only; regression automated in CI. |
| **QA early involvement** | QA in refinement and planning; acceptance criteria co-authored; QA sign-off part of Definition of Done. |
| **Reducing manual effort** | PR gates, full regression in CI, E2E on main, contract tests, automated a11y (axe + Playwright); manual reserved for exploration and compliance. |
| **AI assistance** | Use AI coding assistants to draft pipelines, scaffold tests, and propose automation strategies; humans review and own design decisions. |
| **Security in CI** | Dependency scan, SAST, container scan (if Docker); annual pen test + automated smoke security tests for external API. |
