# Notification Service

## Overview

The Notification Service creates, stores, and delivers user notifications in real-time. It supports user-initiated REST calls, service-to-service internal API calls, Redis Stream event consumption, and real-time SSE (Server-Sent Events) push delivery.

## Port

| Protocol | Port | Context Path    |
| -------- | ---- | --------------- |
| HTTP     | 8091 | `/notification` |

## Technology Stack

- **Java** + **Spring Boot**
- **Spring Security** — OAuth2 Resource Server, Keycloak JWT
- **MongoDB** (Spring Data MongoDB) — notification persistence
- **Redis Streams** (Lettuce) — cross-service event consumer with retry and DLQ
- **Redis Pub/Sub** (Lettuce) — SSE fan-out
- **SSE** (`SseEmitter`) — real-time push to browser clients

## Ingestion Paths

| Path                 | Source                          | Transport                   | Status   |
| -------------------- | ------------------------------- | --------------------------- | -------- |
| REST (user-initiated)| Browser/client                  | `POST /api/v1/notifications`| Active   |
| Internal API         | roadmap-service (OutboxWorker)  | HTTP POST + `X-Internal-Api-Key` | Active |
| Redis Stream consumer| profile-service, future services| `notification_stream`       | Active   |
| Email delivery       | Any event type                  | SMTP/provider               | Deferred |

## Key Flows

### Send Notification (REST)

```
Client
  → POST /notification/api/v1/notifications  (JWT)
  → NotificationController
  → SendNotificationService.send()
  → NotificationMongoRepository.save()
  → MongoDB
```

### Real-Time SSE Delivery

```
(1) Client opens SSE stream
    GET /notification/sse/stream  (JWT or ?token=)
    → SseConnectionManager.register(userId, emitter)
    → Subscribe to Redis channel: user_notif:{userId}

(2) Internal service pushes event
    POST /notification/internal/notifications  (X-Internal-Api-Key)
    → SendNotificationService.sendInternal()
    → NotificationContentBuilder.build(eventType, metadata)
    → MongoDB.save()
    → Redis PUBLISH user_notif:{userId}

(3) Redis fan-out → SSE push
    Redis Pub/Sub → SseRedisSubscriber.onMessage()
    → SseConnectionManager.send(userId, event)
    → Browser receives SSE event name: "notification"
```

### Redis Stream Consumer

- Reads from stream `notification_stream` (consumer group `notification_group`)
- Processes events end-to-end: template render → MongoDB persist → SSE push
- Retries up to `MAX_RETRIES` (default: 3) on failure
- Failed messages go to dead-letter stream `notification_stream_dlq`

## API Endpoints

| Method | Path                                      | Auth         | Description                      |
| ------ | ----------------------------------------- | ------------ | -------------------------------- |
| POST   | `/notification/api/v1/notifications`      | JWT          | Send a notification              |
| GET    | `/notification/api/v1/notifications`      | JWT          | List current user's notifications|
| PATCH  | `/notification/api/v1/notifications/{id}` | JWT          | Mark as read                     |
| GET    | `/notification/sse/stream`                | JWT/token=   | Open SSE real-time stream        |
| POST   | `/notification/internal/notifications`    | API key      | Internal: push notification event|

## Environment Variables

| Variable                  | Default                       | Description                         |
| ------------------------- | ----------------------------- | ----------------------------------- |
| `MONGODB_URI`             | `mongodb://localhost:27017/notification_service` | MongoDB connection |
| `REDIS_HOST`              | `localhost`                   | Redis host                          |
| `REDIS_PORT`              | `6382`                        | Redis port                          |
| `INTERNAL_API_KEY`        | `changeme-local-dev-only`     | API key for internal service calls  |
| `SSE_TIMEOUT_MS`          | `300000`                      | SSE connection timeout (5 min)      |
| `STREAM_NAME`             | `notification_stream`         | Redis Stream to consume             |
| `GROUP_NAME`              | `notification_group`          | Redis consumer group name           |
| `CONSUMER_NAME`           | `notification-service-1`      | Consumer instance name              |
| `MAX_RETRIES`             | `3`                           | Max retry attempts before DLQ       |
| `DLQ_STREAM_NAME`         | `notification_stream_dlq`     | Dead-letter queue stream name       |

## Dependencies

| Dependency          | Role                                            |
| ------------------- | ----------------------------------------------- |
| MongoDB             | Persist notification documents                  |
| Redis (Pub/Sub)     | Fan-out SSE notifications to connected clients  |
| Redis (Streams)     | Consume cross-service events                    |
| API Gateway         | Routes `/notification/**` traffic here          |
| Roadmap Service     | Publishes internal events (OutboxWorker)         |
| Profile Service     | Publishes events to `notification_stream`        |
| Keycloak            | JWT token validation                            |

## Running Locally

```bash
cd notification-serice
./mvnw spring-boot:run
```

Or via Docker Compose:

```bash
docker-compose up notification-service
```

Swagger UI (via gateway): `http://localhost:8090/swagger-ui.html`
Direct: `http://localhost:8091/notification/swagger-ui.html`
