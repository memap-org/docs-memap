# MeMap Backend Architecture

> **Current branch:** `main`
> **GitNexus index:** 10 636 symbols · 19 684 relationships · 300 execution flows

---

## Active Services

| Service | Port | Tech Stack | Database |
|---|---|---|---|
| api-gateway | 8090 | Spring Cloud Gateway MVC 4.x | — |
| profile-service | 8082 (REST) + gRPC | Spring Boot 3.x, MySQL | MySQL + Redis (token cache) |
| roadmap-service | 8083 (REST) + 9083 (gRPC) | Spring Boot 3.x, MongoDB | MongoDB + Redis (Caffeine dual-layer) |
| roadmap-ai-service | 8085 | Spring Boot 3.x, Spring AI, pgvector | PostgreSQL (pgvector) + Redis |
| notification-serice | 8091 | Spring Boot 3.x, MongoDB | MongoDB + Redis |
| payment-service | 8087 | Spring Boot 3.x, MongoDB | MongoDB + Redis + RabbitMQ |
| storage-service | 8088 | Spring Boot 3.x | MinIO (object storage) |
| hocuspocus-server | 1234 | Node.js + TypeScript | MongoDB + Redis |
| memap-frontend | 5173 (dev) | React 19 + Vite + TypeScript | — |
| memap-admin-frontend | 4000 | Next.js 16 + Ant Design | — |

> **Note:** The directory `notification-serice` has a typo in its name — do not rename without updating docker-compose and all build configs.

---

## System Architecture Diagram

```
                        ┌──────────────────────────────────┐
                        │             Clients               │
                        │  memap-frontend  (React 19)       │
                        │  memap-admin-frontend (Next.js 16)│
                        └────────────────┬─────────────────┘
                                         │ HTTPS
                                         ▼
                        ┌──────────────────────────────────┐
                        │           API Gateway             │
                        │     Spring Cloud Gateway MVC      │
                        │            port 8090              │
                        │  • JWT validation (Keycloak JWKS) │
                        │  • Path-based routing             │
                        │  • Swagger UI aggregation         │
                        └───┬───┬───┬───┬───┬──────────────┘
                            │   │   │   │   │
          ┌─────────────────┘   │   │   │   └────────────────────┐
          │          ┌──────────┘   │   └──────────┐             │
          ▼          ▼              ▼              ▼             ▼
   ┌───────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
   │ profile   │ │ roadmap  │ │notif.    │ │ storage  │ │   ai     │
   │ service   │ │ service  │ │ service  │ │ service  │ │ service  │
   │ :8082     │ │ :8083    │ │ :8091    │ │ :8088    │ │ :8085    │
   └─────┬─────┘ └────┬─────┘ └──────────┘ └──────────┘ └────┬─────┘
         │             │ gRPC (client)                          │ gRPC (client)
         │ gRPC        └──────────────────────────────────────►│
         │ (server)    calls profile-service                    │ calls roadmap-service
         └─────────────────────────────────────────────────────┘

  Shared identity:
    All services ──JWT validation──► Keycloak (external IdP)
    profile-service ──Feign HTTP──► Keycloak Admin API (user management)

  Async channels:
    roadmap-service ──RabbitMQ AMQP──► ai-service   (node ingest for RAG)
    ai-service      ──Redis Stream──►  notification-service (AI-triggered notifications)
    notification-service ──Redis Pub/Sub──► SSE ──► frontend

  Real-time collaboration:
    memap-frontend ──WebSocket (Yjs/Hocuspocus)──► hocuspocus-server :1234
                                                       ├─ MongoDB (document state)
                                                       └─ Redis   (pub/sub presence)
```

---

## Bounded Contexts in roadmap-service

`roadmap-service` hosts **two bounded contexts** in a single deployable JAR:

| Context | Package root | Responsibility |
|---|---|---|
| `roadmap` | `com.techmap.roadmap` | Roadmap CRUD, access permissions, reactions, comments, notes, focus sessions, progress tracking, categories |
| `learning` | `com.techmap.learning` | Quizzes, questions, quiz attempts (student + teacher views), assignments, grading |

Both contexts share the same MongoDB instance and Redis cache but have independent controllers, services, repositories, and entity packages.

---

## Authentication & Authorization

- **Identity Provider:** Keycloak (external, not in this monorepo)
- **Pattern:** OAuth2 Resource Server — every Spring Boot service validates JWT via Keycloak's JWKS URI
- **User identity:** `jwt.getSubject()` = Keycloak user ID used everywhere
- **Roles:** Extracted from `realm_access.roles` claim via `CustomAuthoritiesConverter`
- **Gateway layer:** Validates JWT at the edge; downstream services each re-validate independently
- **profile-service ↔ Keycloak:** OpenFeign `IdentityClient` for admin operations (create/enable/update users, token exchange). Client credentials tokens are Redis-cached to avoid redundant token requests

---

## API Gateway Routing

All external traffic enters on port 8090. Routes defined in `api-gateway/src/main/resources/application.yml`:

| Path prefix | Downstream service | Notes |
|---|---|---|
| `/profile/**` | profile-service :8082 | Direct pass-through |
| `/roadmap/**` | roadmap-service :8083 | Rewritten: `/roadmap/{x}` → `/road-map/{x}` |
| `/notification/**` | notification-serice :8091 | Direct pass-through |
| `/storage/**` | storage-service :8088 | Direct pass-through |

Swagger UI is aggregated at `http://localhost:8090/swagger-ui.html` pulling `/api-docs` from each downstream service.

---

## Service-to-Service Communication

See `SERVICE_COMMUNICATION.md` for details with message schemas. Summary:

| Channel | Producer → Consumer | Purpose |
|---|---|---|
| gRPC (profile-service embedded port) | roadmap-service → profile-service | Single + batch user profile lookups |
| gRPC (roadmap-service :9083) | ai-service → roadmap-service | Roadmap name/metadata lookups |
| RabbitMQ AMQP (`ai-ingest-node` queue) | roadmap-service → ai-service | Async RAG vector ingestion of node content |
| Redis Stream (`notification_stream`) | ai-service → notification-serice | Trigger user notifications from AI events |
| Redis Pub/Sub | notification-serice (internal) | Fan-out persisted notifications to active SSE connections |
| HTTP Feign | profile-service → Keycloak | User CRUD + token exchange |
| HTTP (internal endpoint) | profile-service → notification-serice | Email-triggered notification records |

---

## Data Stores

| Store | Used by | Purpose |
|---|---|---|
| MySQL | profile-service | Users, invitations |
| MongoDB | roadmap-service | Roadmaps, nodes, edges, quizzes, attempts, assignments |
| MongoDB | notification-serice | Notification records |
| MongoDB | hocuspocus-server | Collaborative document state (Yjs binary) |
| PostgreSQL + pgvector | roadmap-ai-service | RAG vector embeddings, chat sessions |
| Redis | profile-service | Keycloak client token cache |
| Redis | roadmap-service | Caffeine + Redis dual-layer query cache |
| Redis | roadmap-ai-service | Response caching |
| Redis | notification-serice | SSE pub/sub, Redis Stream consumer |
| Redis | hocuspocus-server | Document presence and pub/sub |
| RabbitMQ | roadmap-service, ai-service | Async node ingest messaging |
| MinIO | storage-service | Binary object storage |

---

## Service-Specific Architecture Docs

- [Profile Service](./profile-service/ARCHITECTURE.md)
- [Roadmap Service](./roadmap-service/ARCHITECTURE.md)
- [Roadmap AI Service](./roadmap-ai-service/ARCHITECTURE.md)
- [Notification Service](./notification-service/ARCHITECTURE.md)
- [Storage Service](./storage-service/ARCHITECTURE.md)
