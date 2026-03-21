# Deliverable 3: API Design

**System:** Task Management System — API for web UI and external integrators.

**Scope:** Structured API endpoint design (`/tasks` CRUD), paradigm recommendation (REST, GraphQL, RPC), framework, and tooling with rationale.

---

## 1. API paradigm: REST recommended

| Paradigm | Recommendation | Rationale |
|----------|----------------|-----------|
| **REST** | **Recommended** | Simple for integrators; standard HTTP verbs and status codes; easy to document with OpenAPI; widely supported; good fit for CRUD-heavy task management. |
| **GraphQL** | Optional later | Reduces over-fetching and chatty calls for complex UIs; steeper learning curve for external partners; add if the web UI needs flexible queries. |
| **RPC** (gRPC, etc.) | Not recommended for public API | Optimized for internal microservice-to-microservice; less familiar to web and external clients; overkill for this use case. |

**Choice:** **REST** for the primary API. It aligns with the architecture (see [Deliverable 1](01-solution-architecture.md)) and suits both the web UI and external integrators. Use **JSON** for request and response bodies.

**Event sourcing:** Task state is stored as events (see [01-solution-architecture.md §3.1](01-solution-architecture.md#31-event-sourcing-for-task-state)). From the client’s perspective the API remains RESTful: POST creates, PATCH updates, GET reads. Under the hood, writes append events to the event store and update the projection; reads use the projection. The full event history is exposed via `GET /v1/tasks/{id}/history`.

---

## 2. `/tasks` endpoint — CRUD operations

Base path: `https://api.{tenant}.taskmgmt.example.com/v1/tasks`

**Tenant isolation:** The `{tenant}` subdomain routes requests to the correct organization; tenant context is derived from the domain and validated against the user's token.

All requests require authentication: `Authorization: Bearer <JWT>` or `X-API-Key: <key>` for machine clients.

**External users:** External users (e.g. B2B guests) may be constrained by role — for example, `GET /v1/tasks` returns only tasks where the user is assignee, creator, or watcher. Internal users with broader roles see all tasks in their org. Enforce via RBAC in the API layer; same endpoints, filtered by token claims and DB membership.

### 2.1 Create task

```
POST /v1/tasks
Content-Type: application/json
```

**Request body:**
```json
{
  "title": "Review Q1 report",
  "description": "Final review before stakeholder presentation",
  "assigneeId": "user-123",
  "dueAt": "2025-04-15T17:00:00Z",
  "priority": "high",
  "status": "open",
  "tags": ["q1", "report"]
}
```

**Response:** `201 Created`
```json
{
  "id": "task-456",
  "title": "Review Q1 report",
  "description": "Final review before stakeholder presentation",
  "assigneeId": "user-123",
  "assignee": {
    "id": "user-123",
    "displayName": "Jane Doe",
    "email": "jane@example.com"
  },
  "dueAt": "2025-04-15T17:00:00Z",
  "priority": "high",
  "status": "open",
  "tags": ["q1", "report"],
  "createdAt": "2025-03-21T10:30:00Z",
  "createdBy": "user-789",
  "updatedAt": "2025-03-21T10:30:00Z"
}
```

---

### 2.2 Read (list tasks)

```
GET /v1/tasks?status=open&assigneeId=user-123&sort=-dueAt&page=1&limit=20
```

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `status` | string | Filter: `open`, `in_progress`, `done`, `cancelled` |
| `assigneeId` | string | Filter by assignee |
| `priority` | string | Filter: `low`, `medium`, `high`, `urgent` |
| `sort` | string | Sort: `dueAt`, `-dueAt`, `createdAt`, `-createdAt`, `priority` |
| `page` | int | Page number (default: 1) |
| `limit` | int | Items per page (default: 20, max: 100) |

**Response:** `200 OK`
```json
{
  "data": [
    {
      "id": "task-456",
      "title": "Review Q1 report",
      "assigneeId": "user-123",
      "assignee": { "id": "user-123", "displayName": "Jane Doe" },
      "dueAt": "2025-04-15T17:00:00Z",
      "priority": "high",
      "status": "open",
      "createdAt": "2025-03-21T10:30:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 20,
    "total": 42,
    "totalPages": 3
  }
}
```

---

### 2.3 Read (single task)

```
GET /v1/tasks/{taskId}
```

**Response:** `200 OK`
```json
{
  "id": "task-456",
  "title": "Review Q1 report",
  "description": "Final review before stakeholder presentation",
  "assigneeId": "user-123",
  "assignee": {
    "id": "user-123",
    "displayName": "Jane Doe",
    "email": "jane@example.com"
  },
  "dueAt": "2025-04-15T17:00:00Z",
  "priority": "high",
  "status": "open",
  "tags": ["q1", "report"],
  "createdAt": "2025-03-21T10:30:00Z",
  "createdBy": "user-789",
  "updatedAt": "2025-03-21T10:30:00Z",
  "comments": [
    {
      "id": "comment-1",
      "body": "Started the review.",
      "authorId": "user-123",
      "author": { "id": "user-123", "displayName": "Jane Doe" },
      "createdAt": "2025-03-22T09:00:00Z"
    }
  ],
  "attachmentCount": 2
}
```

**Error:** `404 Not Found` if task does not exist or requester has no access.

---

### 2.4 Update task

```
PATCH /v1/tasks/{taskId}
Content-Type: application/json
```

**Request body** (partial update; only include changed fields):
```json
{
  "title": "Review Q1 report (updated)",
  "status": "in_progress",
  "assigneeId": "user-456"
}
```

**Response:** `200 OK` — full task object (same shape as `GET /v1/tasks/{taskId}`). The API appends one or more events (e.g. `TitleChanged`, `StatusChanged`, `TaskAssigned`) and updates the projection.

**Error:** `404 Not Found`, `409 Conflict` (e.g. optimistic locking).

---

### 2.5 Task history (event replay)

```
GET /v1/tasks/{taskId}/history?page=1&limit=50
```

Returns the ordered event stream for the task. Supports audit trails and temporal queries.

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `page` | int | Page number (default: 1) |
| `limit` | int | Events per page (default: 20, max: 100) |
| `since` | datetime (ISO 8601) | Optional: events on or after this time |
| `until` | datetime (ISO 8601) | Optional: events on or before this time |

**Response:** `200 OK`
```json
{
  "taskId": "task-456",
  "data": [
    {
      "eventId": "evt-001",
      "type": "TaskCreated",
      "payload": { "title": "Review Q1 report", "status": "open", "assigneeId": "user-123", ... },
      "occurredAt": "2025-03-21T10:30:00Z",
      "actorId": "user-789"
    },
    {
      "eventId": "evt-002",
      "type": "StatusChanged",
      "payload": { "previousStatus": "open", "newStatus": "in_progress" },
      "occurredAt": "2025-03-22T09:15:00Z",
      "actorId": "user-123"
    },
    {
      "eventId": "evt-003",
      "type": "TaskAssigned",
      "payload": { "previousAssigneeId": "user-123", "newAssigneeId": "user-456" },
      "occurredAt": "2025-03-23T14:00:00Z",
      "actorId": "user-789"
    }
  ],
  "pagination": {
    "page": 1,
    "limit": 50,
    "total": 3,
    "totalPages": 1
  }
}
```

**Error:** `404 Not Found` if task does not exist or requester has no access.

---

### 2.6 Delete task

```
DELETE /v1/tasks/{taskId}
```

**Response:** `204 No Content` (empty body).

**Alternative (soft delete):** `PATCH` with `"status": "cancelled"` — appends `TaskCancelled` event and retains full history.

**Error:** `404 Not Found`, `403 Forbidden` if requester lacks permission.

---

### 2.7 Error response format

All errors use a consistent structure:

```json
{
  "error": {
    "code": "TASK_NOT_FOUND",
    "message": "Task with id 'task-999' was not found or you do not have access.",
    "details": { "taskId": "task-999" }
  }
}
```

**HTTP status codes:** `400` Bad Request, `401` Unauthorized, `403` Forbidden, `404` Not Found, `409` Conflict, `429` Too Many Requests, `500` Internal Server Error.

---

## 3. Related endpoints (summary)

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/v1/tasks/{taskId}/history` | GET | Task event history (event sourcing) |
| `/v1/tasks/{taskId}/comments` | GET, POST | List and add comments |
| `/v1/tasks/{taskId}/comments/{commentId}` | PATCH, DELETE | Update or remove comment |
| `/v1/tasks/{taskId}/attachments` | GET, POST | List and upload attachments (metadata; upload URL from presigned URL) |
| `/v1/tasks/{taskId}/attachments/{attachmentId}` | GET, DELETE | Get metadata or delete attachment |
| `/v1/users` | GET | List users (for assignee picker) |
| `/v1/users/{userId}` | GET | User detail |

---

## 4. Framework recommendation

| Choice | Recommendation | Rationale |
|--------|----------------|-----------|
| **Framework** | **ASP.NET Core** (C#) | Fits Azure; strong typing; built-in OpenAPI support; middleware for auth, logging, and validation; good performance and ecosystem. |
| **Alternative** | **Express** (Node.js) or **Spring Boot** (Java) | Use if the team is already committed to Node or Java. |

**Why ASP.NET Core:** Native integration with Azure AD, Azure Key Vault, and Application Insights; first-class OpenAPI/Swagger; easy to host on Azure App Service. Aligns with the Azure-centric setup in [Deliverable 2](02-cloud-setup.md).

---

## 5. Tooling

| Tool | Purpose |
|-----|---------|
| **OpenAPI (Swagger) 3.0** | API specification; generate docs and client SDKs; contract-first or code-first. |
| **Swagger UI** | Built-in with ASP.NET Core; self-hosted interactive docs at `/swagger`. No third-party required. |
| **Postman** or **Insomnia** | Manual and automated API testing; collection sharing. |
| **Azure API Management** (optional) | Rate limiting, versioning, developer portal when scaling external access. |
| **Newman** / **REST Assured** | Automated API tests in CI/CD. |

---

## 6. Security

Full authentication and authorization design is in [Deliverable 1, Section 4](01-solution-architecture.md#4-authentication--authorization-design-decisions). This section covers API-specific security practices.

| Practice | Recommendation |
|----------|----------------|
| **TLS** | All API traffic over **HTTPS only**; TLS 1.2 minimum; HSTS header on responses. |
| **Authentication** | `Authorization: Bearer <JWT>` for user clients; `X-API-Key: <key>` or OAuth2 client credentials for machine clients. Reject unauthenticated requests with `401 Unauthorized`. |
| **Input validation** | Validate and sanitize all inputs (body, query, path). Reject malformed or oversized payloads with `400 Bad Request`. Use schema validation (e.g. FluentValidation, Data Annotations) before processing. |
| **Rate limiting** | Per-client limits (e.g. 100 req/min for user tokens, 1000 req/min for API keys). Return `429 Too Many Requests` with `Retry-After` header. Use Redis for distributed counting. |
| **CORS** | Explicit `Access-Control-Allow-Origin` for known SPA origins; avoid `*` in production. |
| **API keys** | Store hashed; support rotation and per-tenant scoping; audit key creation and usage. |
| **Idempotency** | Support `Idempotency-Key` header on POST and PATCH; deduplicate within 24h window. |

---

## 7. Versioning and compatibility

- **URL versioning:** `/v1/tasks` — clear and widely used.
- **Deprecation:** Use `Deprecation` and `Sunset` HTTP headers when retiring versions.
- **Breaking changes:** New major version (e.g. `/v2/tasks`); keep `v1` supported for a defined period.

---

## 8. Summary

- **Paradigm:** REST with JSON. Task state backed by event sourcing (writes append events, reads use projection).
- **Endpoint:** `/tasks` with full CRUD (POST, GET, PATCH, DELETE), `GET /tasks/{id}/history` for event history, and consistent error format.
- **Security:** HTTPS only; JWT or API key auth; input validation; rate limiting; idempotency keys for writes.
- **Versioning:** URL versioning (`/v1/`); deprecation headers; major version for breaking changes.
- **Framework:** ASP.NET Core for Azure integration and productivity.
- **Tooling:** OpenAPI/Swagger for spec and docs; Postman for testing; optional Azure API Management for external scale.
