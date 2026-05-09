# Notification Service Architecture

## Role in the System

The Notification Service handles the full lifecycle of user notifications: receiving them from multiple sources (REST, internal HTTP, Redis Streams), persisting them in MongoDB, and pushing them in real-time to connected browser clients via SSE.

## High-Level Architecture

```
External Sources                 Internal Sources                  Streams
─────────────                    ────────────────                  ───────
Browser  ──POST /api/v1──┐       Roadmap Svc ──/internal──┐       profile-svc
                          │       (OutboxWorker, API key)  │       ──Redis Stream──┐
                          ▼                               ▼                       ▼
              ┌───────────────────────────────────────────────────────────────┐
              │                    Notification Service                        │
              │                                                                │
              │  NotificationController   InternalNotificationCtrl             │
              │          │                        │                            │
              │          └────────────┬───────────┘         StreamConsumer ◄──┘
              │                       │                      (XREADGROUP)
              │                       ▼
              │           SendNotificationService
              │                  │        │
              │    ContentBuilder│        │PubSubPublisher
              │                  ▼        ▼
              │              MongoDB   Redis PUBLISH
              │              (save)    (user_notif:{userId})
              └───────────────────────────────────────────────────────────────┘
                                                 │
                                                 │ Redis Pub/Sub
                                                 ▼
                                       SseRedisSubscriber
                                                 │
                                                 ▼
                                       SseConnectionManager
                                                 │
                                                 ▼
                                       Browser SSE (EventSource)
```

## Package Structure

```
src/main/java/com/memap/notificationservice/
├── controller/
│   ├── NotificationController       # REST: user-facing endpoints
│   ├── InternalNotificationController # Internal API (X-Internal-Api-Key)
│   └── SseNotificationController    # SSE: GET /sse/stream
├── service/
│   ├── SendNotificationService      # Core: create + persist + publish
│   ├── NotificationContentBuilder   # Render templates by eventType
│   ├── NotificationPubSubPublisher  # Redis PUBLISH to user_notif:{userId}
│   ├── stream/
│   │   ├── StreamConsumer           # XREADGROUP consumer loop
│   │   ├── StreamRecoveryWorker     # Reclaim idle PEL entries
│   │   └── StreamProperties        # Stream config binding
│   └── ...
├── sse/
│   ├── SseConnectionManager        # Per-user emitter registry
│   └── SseRedisSubscriber          # Redis Pub/Sub → SseEmitter.send()
├── repository/
│   ├── NotificationMongoRepository
│   └── document/                   # MongoDB document schemas
├── model/                          # Domain objects and enums
├── dto/                            # Request/response/event payloads
├── config/
│   ├── RedisConfig                 # Lettuce client, pub/sub container
│   ├── MongoConfig                 # MongoDB connection
│   └── security/                  # JWT filter chain
└── exception/                      # Error codes and handlers
```

## SSE Connection Lifecycle

```
Client connects GET /notification/sse/stream
  → Auth: JWT via Authorization header or ?token= param
  → SseConnectionManager.register(userId, SseEmitter)
  → Subscribe Redis channel: user_notif:{userId}
  → Heartbeat every 30s to keep connection alive

On disconnect / timeout / error:
  → SseConnectionManager removes emitter
  → Redis Pub/Sub listener removed for that channel
```

## Redis Streams Consumer

The `StreamConsumer` reads from `notification_stream` using consumer groups (`XREADGROUP`), ensuring at-least-once processing:

```
notification_stream (written by profile-svc, future services)
         │
         │ XREADGROUP GROUP notification_group notification-service-1
         ▼
    StreamConsumer
         │
         ├─ Template render (ContentBuilder)
         ├─ MongoDB persist
         ├─ Redis PUBLISH (fan-out to SSE)
         └─ XACK (on success) or retry (on failure)
         
Max retries: 3 → then XADD to notification_stream_dlq
```

`StreamRecoveryWorker` periodically reclaims stale PEL entries (idle > `IDLE_THRESHOLD_MS`) from other consumers that crashed mid-processing.

## Security

- **External endpoints**: Keycloak JWT (OAuth2 Resource Server)
- **SSE endpoint**: supports `?token=` query param for `EventSource` connections that cannot set headers
- **Internal endpoints**: `X-Internal-Api-Key` header (shared secret, set via `INTERNAL_API_KEY` env var)

## Data Store

MongoDB collection: `notifications`

Key fields: `recipientId`, `senderId`, `type`, `content`, `channel`, `read`, `createdAt`

Indexes initialized on startup (`repository.init`).
