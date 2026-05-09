# Notification Service API

> **Branch:** `main` | **Port:** 8091 | **Context path:** `/notification`
> **Note:** The service directory is `notification-serice` (typo in name) but the Docker container and service are `notification-service`.

---

## Base URLs

```
Via API Gateway:  http://localhost:8090/notification/...
Direct:           http://localhost:8091/notification/...
Swagger UI:       http://localhost:8091/notification/swagger-ui.html
```

---

## REST Endpoints

### `NotificationController` — User-facing CRUD

Base: `/api/v1/notifications`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/notifications` | JWT | Send a notification (manual trigger) |
| GET | `/api/v1/notifications` | JWT | Get notifications for current user (paginated, default page size 20) |
| GET | `/api/v1/notifications/{id}` | JWT | Get notification by ID |
| PATCH | `/api/v1/notifications/{id}/read` | JWT | Mark notification as read |
| PATCH | `/api/v1/notifications/{id}/unread` | JWT | Mark notification as unread |
| PATCH | `/api/v1/notifications/read-all` | JWT | Mark all notifications as read |

### `InternalNotificationController` — Service-to-service

Base: `/api/v1/notifications/internal`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/api/v1/notifications/internal` | Internal (no user JWT) | Receive a notification from another service (e.g., profile-service sending email confirmation) |

This endpoint is called by profile-service when an email event should also create a notification record.

### `SseNotificationController` — Real-time stream

Base: `/api/v1/notifications`

| Method | Path | Produces | Description |
|---|---|---|---|
| GET | `/api/v1/notifications/stream` | `text/event-stream` | SSE stream — subscribe to real-time notifications |

The SSE connection is identified by `jwt.getSubject()` (Keycloak user ID). When a new notification is persisted for this user, it is delivered immediately over the open SSE connection via Redis Pub/Sub fan-out.

---

## Notification Delivery Architecture

```
  [Producer]                      [notification-serice]              [Client]
  ─────────                       ─────────────────────              ────────

  profile-service  ──HTTP──►  InternalNotificationController
                                      │
  ai-service  ──Redis Stream──►  StreamConsumer
                                      │
                                SendNotificationService
                                      │ persist to MongoDB
                                      │ publish to Redis Pub/Sub
                                      │
                              SseRedisSubscriber ──SSE──►  memap-frontend
```

---

## Notification Data Model

```json
{
  "id": "string",
  "title": "string",
  "content": "string",
  "channel": "SYSTEM | EMAIL | ...",
  "sender": "userId",
  "receiver": "userId",
  "readStatus": "UNREAD | READ",
  "createdAt": "ISO-8601 instant",
  "updatedAt": "ISO-8601 instant"
}
```

---

## Redis Stream Consumer (Inbound from ai-service)

The service consumes from the `notification_stream` Redis Stream.

Expected fields on each record:

| Field | Type | Description |
|---|---|---|
| `template_key` | String | Notification template identifier |
| `origin_service` | String | Which service published the event |
| `receiver_user_id` | String | Keycloak user ID of the recipient |
| `reference_type` | String | Type of the referenced entity |
| `reference_id` | String | ID of the referenced entity |
| `metadata` | JSON string | Additional key-value context |

---

## Health Check

```
GET http://localhost:8091/notification/actuator/health
```

Used by Docker Compose health check with 30s interval, 40s start period.

---

## Architecture Notes

- Clean/Hexagonal architecture — see `notification-serice/CODE_CONVENTIONS.md` for full details
- Domain models in `domain/model/` are pure Java (no Lombok, no Spring)
- Flyway manages MySQL schema at `src/main/resources/db/migration/`
- All timestamps use `Instant`
- Table names follow `tbl_` prefix convention
