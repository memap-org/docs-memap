# Roadmap AI Service API Documentation

> **Branch:** `main` | **Port:** 8085
> All production endpoints should use JWT auth via the API gateway.
> `TestRoadmapGrpcController` and `TestSendMessageController` are development-only endpoints.

---

## Base URLs

```
Via API Gateway:  (not yet routed through gateway — direct access only)
Direct:           http://localhost:8085/...
Swagger UI:       http://localhost:8085/swagger-ui.html
```

---

## REST Endpoints

### Roadmap Generation — `RoadmapGeneratorController`

Base: `/roadmap`

| Method | Path | Description |
|---|---|---|
| POST | `/roadmap/generate` | Generate a complete roadmap (nodes + edges) from a topic/prompt |
| GET | `/roadmap/generate` | Generate roadmap by topic (query param) |

**POST request body:**
```json
{
  "topic": "string",
  "level": "BEGINNER | INTERMEDIATE | ADVANCED",
  "language": "string"
}
```

**Response:** `GenerateRoadmapResponse` — structured roadmap with nodes and edges ready for import into roadmap-service via `POST /api/roadmap/ai-generated`.

### Node AI — `NodeAIController`

Base: `/node`

| Method | Path | Description |
|---|---|---|
| POST | `/node/summary` | Generate AI summary for a node (by content in request body) |
| GET | `/node/summary/{roadmapId}/{nodeId}` | Get cached summary for a node by roadmap + node ID |

Summaries are generated using the Spring AI RAG pipeline with pgvector and cached in Redis.

### Chat / Q&A — `ChatSessionController`

Base: `/chat/sessions`

| Method | Path | Description |
|---|---|---|
| POST | `/chat/sessions/query` | Submit a question in a chat session (RAG-powered) |

**Request:**
```json
{
  "sessionId": "string (optional, creates new if absent)",
  "question": "string",
  "roadmapId": "string (optional, scopes RAG to a roadmap)"
}
```

Chat history is persisted via `spring-ai-starter-model-chat-memory-repository-jdbc`.

### General AI — `AIController`

Base: `/ai`

| Method | Path | Description |
|---|---|---|
| POST | `/ai` | Generic AI prompt processing |

### Dev/Test Endpoints (not for production)

| Controller | Path | Description |
|---|---|---|
| `TestRoadmapGrpcController` | GET `/test/grpc/roadmap/{id}` | Test gRPC call to roadmap-service |
| `TestSendMessageController` | POST `/test/ingest-node` | Manually publish an ingest-node AMQP message |

---

## Inbound Messaging (RabbitMQ)

**Queue:** `${rabbitmq.queue.ai-ingest-node}`
**Listener:** `IngestNodeListener`

Receives `IngestNodeMessage` from roadmap-service when a node's content changes. Passes it to `DocumentIngestFactory` which chunks and embeds content into the pgvector store for future RAG queries.

---

## Outbound Messaging (Redis Stream)

**Publisher:** `NotificationStreamPublisher`
**Stream key:** `notification_stream`

When an AI operation completes (e.g., roadmap generation), the service publishes a notification event to the Redis Stream. The notification-serice consumes this stream and delivers the notification to the user via SSE.

```json
{
  "template_key": "string",
  "origin_service": "ai-service",
  "receiver_user_id": "keycloak-user-id",
  "reference_type": "ROADMAP | NODE",
  "reference_id": "string",
  "metadata": { "key": "value" }
}
```

---

## Outbound gRPC (Client)

**Target:** roadmap-service :9083
**Client:** `RoadmapGrpcClient`
**RPC:** `RoadmapLookupService.GetRoadmapName(roadmap_id)` → `{name, found}`

Used to enrich notification messages with human-readable roadmap names.

---

## Tech Stack Details

- **Spring AI** — LLM integration, RAG advisors
- **pgvector** — Vector similarity search for RAG
- **Spring AI chat memory** — JDBC-backed conversation history
- **RabbitMQ** — Async node ingestion
- **Redis** — Result caching + notification stream publishing
- **gRPC** — Roadmap metadata lookups from roadmap-service
