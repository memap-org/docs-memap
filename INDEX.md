# MeMap Platform Documentation Index

> **Branch:** `main` | **Last updated:** 2026-05-08
> This index reflects the **currently active** services. Docs for `identity-service`, `learning-service`, `payment-service`, and `file-service` are outdated — those services have been removed or merged.

---

## Documentation Structure

```
docs/
├── INDEX.md                     # This file
├── AUDIT_REPORT.md              # 🆕 Cross-service bug + performance audit (2026-05-09)
├── API.md                       # API overview with gateway routes
├── ARCHITECTURE.md              # System architecture (updated)
├── DEPLOYMENT.md                # Deployment guides
├── ERROR_CODES.md               # Error code reference
├── GETTING_STARTED.md           # Quick start guide
├── SERVICE_COMMUNICATION.md     # gRPC, AMQP, Redis Stream, SSE (updated)
├── GRPC_TROUBLESHOOTING.md      # gRPC debugging guide
├── REFRESH_TOKEN_WORKFLOW.md    # Keycloak token refresh flows
│
├── profile-service/             # User profiles, invitations, gRPC server
│   ├── README.md
│   ├── API.md                   # ✅ Updated with actual endpoints
│   └── ARCHITECTURE.md
│
├── roadmap-service/             # Roadmaps + Learning (quizzes, assignments)
│   ├── README.md
│   ├── API.md                   # ✅ Updated — includes both roadmap + learning contexts
│   ├── ARCHITECTURE.md
│   ├── FEATURES.md
│   └── DATABASE_SCHEMA.md
│
├── roadmap-ai-service/          # AI generation, RAG, chat, node summaries
│   ├── README.md
│   ├── API.md                   # ✅ Updated with actual endpoints
│   ├── ARCHITECTURE.md
│   └── FEATURES.md
│
├── notification-service/        # Notifications, SSE real-time delivery
│   ├── README.md
│   ├── API.md                   # ✅ Updated — SSE not STOMP, Redis Stream inbound
│   └── ARCHITECTURE.md
│
├── storage-service/             # File upload/download, MinIO
│   ├── README.md
│   ├── API.md                   # ✅ Updated with actual endpoints
│   └── ARCHITECTURE.md
│
├── hocuspocus-server/           # Real-time collaborative editing (Yjs/WebSocket)
│   ├── README.md
│   └── ARCHITECTURE.md
│
├── memap-frontend/              # React 19 + Vite SPA
│   ├── README.md
│   ├── API.md
│   └── ARCHITECTURE.md
│
├── api-gateway/                 # Spring Cloud Gateway routing docs
│
└── [OUTDATED — do not use]
    ├── identity-service/        # Removed — auth is now Keycloak-only
    ├── learning-service/        # Merged into roadmap-service
    ├── payment-service/         # Removed
    └── file-service/            # Merged into storage-service
```

---

## Active Backend Services

### API Gateway (port 8090)
Routes all external traffic. JWT validation at the edge. Swagger aggregation at `/swagger-ui.html`.

### Profile Service (port 8082)

User registration, profile management, invitation flow, password management, gRPC server for profile lookups.

- [API Documentation](./profile-service/API.md) — Registration, invitations, avatar, gRPC contract
- [Architecture](./profile-service/ARCHITECTURE.md)

### Roadmap Service (port 8083 + gRPC 9083)

Two bounded contexts in one deployable:
- **Roadmap context** — CRUD, permissions, reactions, comments, notes, focus/progress tracking, categories
- **Learning context** — Quizzes, questions, quiz attempts (student + teacher), assignments, grading

- [API Documentation](./roadmap-service/API.md) — All endpoints for both contexts
- [Architecture](./roadmap-service/ARCHITECTURE.md)

### Roadmap AI Service (port 8085)

Roadmap generation via LLM, node summaries, RAG-powered Q&A chat, async node ingestion into pgvector.

- [API Documentation](./roadmap-ai-service/API.md) — Generate, summarize, chat, gRPC/AMQP details
- [Architecture](./roadmap-ai-service/ARCHITECTURE.md)

### Notification Service (port 8091)

Notification CRUD + real-time SSE delivery. Consumes from Redis Stream (ai-service) and HTTP internal endpoint (profile-service).

- [API Documentation](./notification-service/API.md) — REST endpoints + SSE stream + Redis Stream schema
- [Architecture](./notification-service/ARCHITECTURE.md)

### Storage Service (port 8088)

File upload/download/access via MinIO. Public access endpoint for inline image/PDF viewing.

- [API Documentation](./storage-service/API.md) — Upload, download, access, quota management
- [Architecture](./storage-service/ARCHITECTURE.md)

### Hocuspocus Server (port 1234)

WebSocket server for Yjs CRDT collaborative editing of roadmap documents. Loads state from MongoDB, syncs via Redis.

- [Architecture](./hocuspocus-server/ARCHITECTURE.md)
- [README](./hocuspocus-server/README.md)

---

## Frontend Applications

### memap-frontend (port 5173 dev)

React 19 + Vite + TypeScript. Feature-slice architecture. Redux Toolkit state. TipTap rich text with Yjs collaboration.

- [Architecture](./memap-frontend/ARCHITECTURE.md)

### memap-admin-frontend (port 4000)

Next.js 16 App Router + Ant Design 6 + Tailwind CSS 4. Admin panel for managing roadmaps, users, quizzes.

---

## Cross-Cutting References

| Document | Description |
|---|---|
| [ARCHITECTURE.md](./ARCHITECTURE.md) | Full system diagram, data stores, communication matrix |
| [SERVICE_COMMUNICATION.md](./SERVICE_COMMUNICATION.md) | gRPC contracts, AMQP schemas, Redis Stream format, SSE |
| [ERROR_CODES.md](./ERROR_CODES.md) | Error code ranges across all services |
| [GRPC_TROUBLESHOOTING.md](./GRPC_TROUBLESHOOTING.md) | Debugging gRPC between profile ↔ roadmap ↔ ai-service |
| [REFRESH_TOKEN_WORKFLOW.md](./REFRESH_TOKEN_WORKFLOW.md) | Keycloak token lifecycle and refresh flows |
| [GETTING_STARTED.md](./GETTING_STARTED.md) | Local dev setup with Docker Compose |

---

## By Feature

| Feature | Service |
|---|---|
| User registration / login | profile-service + Keycloak |
| Invitations | profile-service |
| Roadmap CRUD | roadmap-service (roadmap context) |
| Roadmap collaboration (real-time) | hocuspocus-server + memap-frontend |
| Roadmap permissions (teacher/student) | roadmap-service → profile-service (gRPC) |
| Progress / focus tracking | roadmap-service |
| AI roadmap generation | roadmap-ai-service → roadmap-service |
| Node RAG search / Q&A | roadmap-ai-service (pgvector) |
| Quizzes and assignments | roadmap-service (learning context) |
| Real-time notifications | notification-serice (SSE) |
| File uploads | storage-service |

## By Technology

| Technology | Used by |
|---|---|
| Spring Boot 3.x | All Java services |
| MySQL | profile-service |
| MongoDB | roadmap-service, notification-serice, hocuspocus-server |
| PostgreSQL + pgvector | roadmap-ai-service |
| Redis | All Java services + hocuspocus-server |
| RabbitMQ | roadmap-service → ai-service (node ingest) |
| MinIO | storage-service |
| gRPC | roadmap-service ↔ profile-service, ai-service ↔ roadmap-service |
| Redis Stream | ai-service → notification-serice |
| SSE | notification-serice → memap-frontend |
| WebSocket (Yjs) | memap-frontend ↔ hocuspocus-server |
| Keycloak | Auth for all services |
| React 19 + Vite | memap-frontend |
| Next.js 16 | memap-admin-frontend |
