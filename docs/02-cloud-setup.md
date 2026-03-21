# Deliverable 2: Cloud Setup

**System:** Task Management System — Azure deployment for enterprise context; **starts** at ~1,000 users and **grows slowly** toward 3,000–10,000.

**Scope:** How to set up the application in **Azure**, which services and components to choose, and trade-offs in **performance**, **cost**, and **complexity**.

---

## 1. Azure services and components

This section maps each part of the architecture (see [Deliverable 1](01-solution-architecture.md)) to specific Azure services and explains the choices.

| Component | Azure Service | Rationale |
|-----------|---------------|-----------|
| **Core monolith (API)** | **Azure App Service** | Hosts the main API and domain logic; supports auto-scale, managed identity, easy deployment. |
| **Notification microservice** | **Azure App Service** | Runs notification delivery; separate App Service plan to scale independently from the core API. |
| **Relational database** | **Azure Database for PostgreSQL — Flexible Server** | Hosts event store (append-only `task_events`), projections (tasks, comments), and outbox; read replicas, automated backups, Azure Monitor integration. |
| **Cache** | **Azure Cache for Redis** | Session cache, rate limiting, distributed locks. |
| **Object storage** | **Azure Blob Storage** | Attachments; lifecycle policies, optional Azure CDN in front. |
| **Message queue** | **Azure Service Bus** (Standard or Premium tier) | Async notifications; topics/subscriptions, DLQ (dead-letter queue) built-in. |
| **Identity** | **Azure AD** (Microsoft Entra ID) | SSO, OIDC/SAML, B2B for external users. |
| **Email and SMS** | **Azure Communication Services** | Email, SMS; single service, pay-per-use. |
| **Teams integration** | **Microsoft Graph API** | Sends to Microsoft Teams; same tenant, no extra charge for Graph. |
| **Scheduler (overdue, SLA)** | **Azure Functions** (Timer trigger) or **Azure Logic Apps** | Runs rules on schedule; publishes to Service Bus. |
| **Static assets (SPA)** | **Azure Blob Storage** + **Azure CDN** | Serves web UI; CDN reduces latency for global users. |
| **Edge (load balancer, WAF)** | **Azure Front Door** (Premium) or **Azure Application Gateway** + **WAF** | TLS termination, routing, DDoS protection, WAF. |
| **Observability** | **Azure Monitor**, **Log Analytics**, **Application Insights** | Logs, metrics, traces, dashboards, alerts. |
| **Optional search** | **Azure Cognitive Search** | Full-text and dashboard search if needed. |
| **CI/CD** | **Azure DevOps Pipelines** | Build, test, deploy to App Service; Azure-native. |
| **Secrets** | **Azure Key Vault** | API keys, DB connection strings, third-party credentials. |
| **Backup (cross-region)** | **Geo-redundant backup** for PostgreSQL, Blob GRS, Recovery Services vault | Protects data if the primary region fails. |

---

## 2. Deployment topology

### 2.1 Diagram (text)

```text
  +------------------+     +------------------+
  |   Web browser    |     |  External API    |
  |   (SPA)          |     |  clients         |
  +--------+---------+     +--------+---------+
           |                         |
           v                         v
  +--------+-------------------------+--------------------------+
  |                    Azure Front Door (or App Gateway + WAF)   |
  |                    TLS, routing, WAF, DDoS protection        |
  +--------+-------------------------+--------------------------+
           |                         |
           v                         v
  +--------+-------------------------+--------------------------+
  |        Azure CDN (static assets) |  API path → App Service   |
  +----------------------------------+--------------------------+
                                              |
                    +-------------------------+-------------------------+
                    |                                                   |
                    v                                                   v
  +---------------------------------+                 +---------------------------------+
  |  Azure App Service              |                 |  Azure App Service              |
  |  — Core monolith                |                 |  — Notification microservice     |
  |  — Tasks, users, comments,      |                 |  — Email, SMS, Teams delivery    |
  |    attachments, outbox          |                 |  — Consumes from Service Bus    |
  +--------+------------------------+                 +---------------------------------+
           |            |            |                              |
           v            v            v                              v
  +--------+   +--------+   +--------+                    +--------------------+
  | Azure  |   | Azure  |   | Azure  |                    | Azure Service Bus  |
  | DB for |   | Cache  |   | Blob   |                    | (queue/topic)      |
  | Postgres|   | Redis  |   | Storage|                    +----------+---------+
  +--------+   +--------+   +--------+                               ^
           |            |            |                               |
           +------------+------------+                    +----------+---------+
                                                                     |
                                                          +----------+---------+
                                                          | Azure Functions    |
                                                          | (Timer - overdue,  |
                                                          |  SLA scheduler)    |
                                                          +--------------------+

  Identity: Azure AD (Entra ID) — token validation on each request
  Notifications: Azure Communication Services (email, SMS) + Microsoft Graph (Teams)
  Observability: Azure Monitor, Log Analytics, Application Insights
  Secrets: Azure Key Vault
```

### 2.2 Resource group layout

Suggested structure for a single environment (e.g. production):

| Resource group | Contents |
|----------------|----------|
| `rg-taskmgmt-prod-net` | Virtual network (if needed), Front Door, CDN |
| `rg-taskmgmt-prod-app` | App Service (core + notification), Azure Functions |
| `rg-taskmgmt-prod-data` | PostgreSQL, Redis, Blob Storage, Service Bus |
| `rg-taskmgmt-prod-identity` | Azure AD (tenant-level; no RG), Key Vault |
| `rg-taskmgmt-prod-monitor` | Log Analytics workspace, Application Insights |

### 2.3 High-level setup steps

1. **Provision networking:** Create virtual network (optional; for private endpoints), subnets.
2. **Provision data layer:** PostgreSQL (Flexible Server), Redis, Blob Storage, Service Bus; configure firewall / VNet rules.
3. **Provision identity:** Configure Azure AD app registrations, B2B for external users; add Key Vault and secrets.
4. **Provision edge:** Front Door or Application Gateway + WAF; custom domain and TLS; route `/api/*` to App Service, static to Blob + CDN.
5. **Deploy core monolith:** App Service with managed identity; connection strings from Key Vault.
6. **Deploy Notification microservice:** Separate App Service; consumer for Service Bus.
7. **Deploy scheduler:** Azure Functions with timer trigger; publish events to Service Bus.
8. **Configure observability:** Log Analytics, Application Insights, alerts, dashboards.
9. **CI/CD:** Azure DevOps Pipelines for build and deploy.

---

## 3. Trade-offs: performance, cost, complexity

### 3.1 Performance

#### Compute: App Service — Single plan vs separate plans

| Option | What it means | Example |
|--------|---------------|---------|
| **Single plan** | Core monolith and Notification microservice run in one App Service plan; they share CPU, memory, and scaling rules. | One plan with 2 instances; both apps share those 2 instances. |
| **Separate plans** | Each app has its own plan; scaling is independent. | Core: 2–3 instances for API traffic. Notification: 1–2 instances for queue processing. |

**Example scenario:** A bulk assignment triggers 500 task assignments. Notification volume spikes. With a single plan, the core API can slow down as notifications consume CPU. With separate plans, the Notification app scales up without affecting the core API.

---

#### Database: Single server vs read replicas

| Option | What it means | Example |
|--------|---------------|---------|
| **Single server** | One PostgreSQL instance handles all reads and writes. | D2s_v3, 2 vCores; every query hits the primary. |
| **Read replicas** | Primary for writes; one or more replicas for read-only queries. | Primary for writes; 1 replica for dashboards, reports, and heavy list queries. |

**Example scenario:** A dashboard runs "tasks created this week" across 10,000 tasks. That query runs on the replica. Writes (create task, update status) stay on the primary, so they don't contend with heavy reads.

**When to add replicas:** When dashboards or reports noticeably slow the primary, or when read latency degrades during peak hours.

---

#### CDN: No CDN vs Azure CDN

| Option | What it means | Example |
|--------|---------------|---------|
| **No CDN** | Static assets served directly from App Service or Blob. | User in Sydney fetches `main.js` from East US → ~200–300 ms latency. |
| **Azure CDN** | Assets cached at edge locations worldwide. | User in Sydney fetches `main.js` from Sydney edge → ~20–50 ms latency. |

**Example scenario:** The SPA loads ~2 MB of JS/CSS on first load. Without CDN, all of that flows through the origin. With CDN, the first request populates the cache; subsequent loads (and users in other regions) hit the edge, improving load time and reducing origin load.

---

#### Service Bus: Standard vs Premium

| Option | What it means | Example |
|--------|---------------|---------|
| **Standard** | Shared throughput; pay per million messages; public endpoint. | ~$10 base + ~$0.05 per million operations; suitable for moderate volume. |
| **Premium** | Dedicated throughput units; VNet integration; private endpoints. | Fixed monthly cost; better for compliance (private network) and higher, predictable throughput. |

**Example scenario:** Standard handles roughly 1,000–2,000 messages/second. Premium is for when you need network isolation, stricter SLAs, or higher throughput.

---

#### Redis: Basic vs Standard

| Option | What it means | Example |
|--------|---------------|---------|
| **Basic (C0)** | Single node, no replication; 250 MB. | Cheapest; if the node fails, cache is lost until restart. |
| **Standard (C1)** | Primary + replica; automatic failover; 1 GB. | More reliable; brief failover if primary goes down; preferred for production. |

**Example scenario:** Redis is used for rate limiting and session cache. With Basic, a node failure wipes the cache and resets rate limits. With Standard, failover keeps the cache available (with brief interruption).

---

### 3.2 Cost

#### Component-by-component examples

| Component | Lower-cost choice | Higher-performance choice | Rough cost difference |
|-----------|-------------------|---------------------------|------------------------|
| **App Service** | Basic B1, 1 instance (~$13/mo) | Standard S2, 3 instances (~$210/mo) | ~16× |
| **PostgreSQL** | Burstable B2ms, 128 GB (~$50/mo) | General Purpose D4s_v3, 256 GB (~$400/mo) | ~8× |
| **Redis** | Basic C0 (~$16/mo) | Standard C1 (~$75/mo) | ~5× |
| **Service Bus** | Standard, 1M msgs/mo (~$15/mo) | Premium, 1 unit (~$700/mo) | ~47× (Premium is a step change) |
| **Blob Storage** | Hot, 100 GB (~$2) + lifecycle to Cool | Hot only, 500 GB (~$10) | Depends on access patterns and lifecycle |
| **Front Door** | Standard (~$35/mo base) | Premium (~$330/mo base) | ~9× |

#### Example monthly cost ranges

- **Start (~1k users):** ~$120–250/month — App Service B1 or S1, PostgreSQL B1ms or B2ms (geo-redundant backup), Redis C0 or skip, Service Bus Basic, Blob GRS, no CDN or Front Door. Monolith with async workers. Includes cross-region backup.
- **Growth (~2k–10k users):** ~$1,550–2,600/month — App Service S2 (core) + S1 (notification), PostgreSQL D2s_v3 (geo-redundant backup), Redis Standard C1, Service Bus Standard, Blob GRS + CDN, Front Door Standard, Application Insights. Macroservices. Includes cross-region backup.
- **Enterprise (high SLA, HA):** ~$3,000–5,000/month — Premium tiers, read replicas, multi-region, private endpoints.

---

### 3.3 Complexity

#### Private networking: Public vs private endpoints

| Option | What it means | Example |
|--------|---------------|---------|
| **Public endpoints** | Services reachable over the internet; secured by firewall rules and auth. | PostgreSQL: "Allow Azure services"; App Service: public URL. Simpler to set up. |
| **Private endpoints** | Services get private IPs in your VNet; no public exposure. | PostgreSQL and Redis only reachable from your VNet. Better for compliance (e.g. PCI-DSS). |

**Example scenario:** With public endpoints, you rely on firewall rules and authentication. With private endpoints, database traffic never leaves the private network — useful when auditors require no public DB access.

---

#### Compute: Single plan vs multiple plans + Functions

| Option | What it means | Example |
|--------|---------------|---------|
| **Single App Service plan** | Core + Notification in one plan; optionally Functions for scheduler. | One plan to manage; both apps share scaling. |
| **Two App Service plans + Functions** | Core on one plan, Notification on another; Functions for timer jobs. | Core and Notification scale independently; Functions handles scheduled work. |

**Example scenario:** Notification volume grows. With a single plan, you scale for the combined peak. With separate plans, you scale the Notification app without over-provisioning the core API.

---

#### Multi-region: Single vs multi-region

| Option | What it means | Example |
|--------|---------------|---------|
| **Single region** | All resources in one Azure region (e.g. East US). | Simpler ops; users far from the region see higher latency. |
| **Multi-region** | App deployed in multiple regions; geo-replicated DB; global load balancing. | Lower latency for global users; more complex deployment and failover. |

**Example scenario:** Users in Europe and Asia. Single region in East US → 100–200 ms from Europe, 200–300 ms from Asia. Multi-region (e.g. West Europe + East Asia) → 20–50 ms from those regions, but requires replication and failover logic.

---

#### CI/CD: Single vs multiple pipelines

| Option | What it means | Example |
|--------|---------------|---------|
| **Single pipeline** | One pipeline builds and deploys both core and Notification. | Simpler to maintain; any change triggers a full deploy. |
| **Separate pipelines** | One pipeline per app. | Independent releases; deploy Notification without touching core. |

**Example scenario:** Single pipeline: change the core API → full build and deploy for both. Separate pipelines: update email templates in Notification only → deploy just the Notification service.

---

#### Observability: Application Insights only vs full stack

| Option | What it means | Example |
|--------|---------------|---------|
| **Application Insights only** | Default App Service integration; traces, metrics, basic dashboards. | Quick to set up; sufficient for many use cases. |
| **Application Insights + Log Analytics + workbooks** | Centralized logs, custom queries, dashboards. | Deeper analysis; custom alerts; "How many notifications failed in the last hour?" queries. |

**Example scenario:** Application Insights gives basic metrics. Log Analytics lets you query structured logs and build dashboards around failed notifications or slow endpoints.

---

## 4. Summary of monthly cost and setup tiers

### 4.0 Monthly cost at a glance

| Phase | Users | Architecture | Estimated cost/month |
|-------|-------|---------------|----------------------|
| **Start** | ~1,000 | Monolith (single App Service, async workers) + cross-region backup | **~$120–250/month** |
| **Transition** | ~2,000–3,000 | First step: extract Notification microservice; separate App Service plans; add CDN | **~$400–700/month** |
| **Growth** | ~3,000–10,000 | Full Macroservices (core + Notification); Front Door, read replicas, larger DB | **~$1,550–2,600/month** |

*Cross-region backup (PostgreSQL geo-redundant, Blob GRS) adds ~$20–80/month — included in the ranges above. See Section 5 for details.*

**At ~1k users (start):** Monolith with App Service S1 or B1, PostgreSQL Burstable, optional Redis, Service Bus Basic. No Front Door or CDN initially. Lower cost, simpler to operate.

**At ~2–3k users (transition):** Extract Notification microservice to separate App Service plan; add CDN for SPA; scale database. Bridge between Monolith and full Macroservices setup.

**Component breakdown (~$100–200/month total):**

| Component | Choice | Rough cost/month |
|-----------|--------|------------------|
| App Service | B1 or S1 (1 instance) | ~$13 (B1) or ~$70 (S1) |
| Azure Database for PostgreSQL | Burstable B1ms or B2ms | ~$25–50 |
| Azure Cache for Redis | Skip or Basic C0 | $0 or ~$16 |
| Azure Blob Storage | Standard LRS | ~$2–5 |
| Azure Service Bus | Basic tier | ~$10 |
| Azure AD | Free tier | $0 |
| Application Insights | Free tier / included | $0 |
| Azure Communication Services | Pay-per-use (email, SMS) | ~$5–25 (usage-based) |
| Cross-region backup (PostgreSQL geo-redundant, Blob GRS) | See Section 5 | ~$20–40 |
| **Total** | | **~$120–250** |

**At ~10k users (grown):** Macroservices with separate App Service plans, Front Door, CDN, larger database, read replicas as needed. Higher cost, production-grade resilience and global performance.

---

### 4.1 Minimal — ~1k users (start)


- **App Service** (B1) — core + notification in one app.
- **Azure Database for PostgreSQL** (Burstable B1ms).
- **Azure Cache for Redis** (Basic C0) or skip initially.
- **Azure Blob Storage** (Standard LRS or GRS for cross-region backup).
- **Azure Service Bus** (Basic tier).
- **Azure AD** (free tier).
- **Application Insights** (included in App Service).
- No CDN, no Front Door; use App Service custom domain and TLS.

**Complexity:** Low. **Cost:** ~$120–250/month (includes cross-region backup). **Performance:** Suitable for ~1k users; Monolith with async workers.

---

### 4.2 Transition — ~2–3k users (optional intermediate)

- **App Service** (S1) — core monolith; separate plan for Notification microservice.
- **Azure CDN** — static assets (SPA).
- **Azure Database for PostgreSQL** (Burstable B2ms or General Purpose D2s_v3).
- **Azure Cache for Redis** (Basic C0 or Standard C1).
- **Azure Service Bus** (Basic or Standard).

**Complexity:** Medium. **Cost:** ~$400–700/month. **Use when:** Extracting Notification microservice before full Growth tier.

---

### 4.3 Recommended — growing toward 10k users

- **Azure Front Door** (Standard) — TLS, WAF, routing.
- **Azure CDN** — static assets.
- **App Service** (Standard S2 or Premium P1v2) — core monolith; 2–3 instances.
- **App Service** (Standard S1) — Notification microservice (separate plan).
- **Azure Functions** (Consumption) — scheduler.
- **Azure Database for PostgreSQL** (General Purpose D2s v3) — with read replica if needed.
- **Azure Cache for Redis** (Standard C1).
- **Azure Blob Storage** (Standard GRS for cross-region backup, lifecycle to Cool).
- **Azure Service Bus** (Standard).
- **Azure Communication Services** — email, SMS.
- **Azure AD** (Entra ID) — SSO, B2B.
- **Key Vault** — secrets.
- **Application Insights** + **Log Analytics** — observability.

**Complexity:** Medium. **Cost:** ~$1,550–2,600/month (includes cross-region backup). **Performance:** Suitable for 2k–10k users; Macroservices with independent scaling.

---

## 5. Backup and disaster recovery (cross-region)

For **safety**, backups should be stored in **another region** so data survives a regional outage. Recommended approach:

| Component | Backup strategy | How it works |
|-----------|-----------------|--------------|
| **PostgreSQL** | **Geo-redundant backup** | Azure Database for PostgreSQL stores automated backups in the **paired region** (e.g. East US → West US). Restore to the secondary region if the primary fails. |
| **Blob Storage** | **GRS** (Geo-Redundant Storage) instead of LRS | Data replicated synchronously to 3 copies in primary region, then **asynchronously to the paired region**. Survives regional failure. |
| **Optional: full backup** | **Azure Backup** + **Recovery Services vault** (cross-region) | Backup vault with **cross-region restore**; store backups in a different region. Covers app config, file backups, etc. |

**Cost impact:** Geo-redundant PostgreSQL backup is usually included. Blob GRS adds roughly 20–40% over LRS. Recovery Services vault adds ~$5–20/month depending on retained backup size. **Rough add-on:** ~$20–80/month for cross-region backup at 1k–10k user scale.

---

## 6. Summary

The Task Management System is deployed on **Azure** using **App Service** for the core monolith and Notification microservice (separate plans for independent scaling), **Azure Database for PostgreSQL** and **Azure Cache for Redis** for data, **Azure Blob Storage** for attachments, **Azure Service Bus** for async notifications, **Azure AD** for identity, **Azure Communication Services** and **Microsoft Graph** for delivery, and **Azure Monitor** for observability. **Backup** uses geo-redundant PostgreSQL backups and Blob GRS so data is replicated to another region. Trade-offs favor **simplicity and cost control** at the start, with clear paths to higher performance (read replicas, Premium tiers, CDN) and stricter security (private endpoints, multi-region) when requirements grow.
