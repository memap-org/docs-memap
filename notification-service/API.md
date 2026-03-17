# Notification Service API

## REST Endpoints

Full REST API documentation is available via Swagger UI at:

```
http://localhost:8091/notification/swagger-ui.html
```

---

## WebSocket / Real-time Notifications

The Notification Service provides real-time delivery of notifications to connected clients via **STOMP over WebSocket** (with SockJS fallback).

### Connection Details

| Property             | Value                                |
| -------------------- | ------------------------------------ |
| WebSocket URL        | `ws://<host>/notification/ws`        |
| SockJS URL           | `http://<host>/notification/ws`      |
| SockJS info endpoint | `http://<host>/notification/ws/info` |
| Protocol             | STOMP 1.2 over WebSocket / SockJS    |
| Server heartbeat     | 25 000 ms                            |

### Authentication

Authentication is performed at the STOMP CONNECT frame level. Include the JWT in one of the following native headers:

```
Authorization: Bearer <jwt-token>
```

or:

```
token: <jwt-token>
```

> Connections without a valid JWT are rejected immediately.

### STOMP Connection Flow

1. **Connect** to the WebSocket endpoint with a valid JWT:

```javascript
const socket = new SockJS('http://localhost:8091/notification/ws');
const stompClient = Stomp.over(socket);

stompClient.connect(
  { Authorization: 'Bearer <jwt-token>' },
  (frame) => {
    console.log('Connected:', frame);
    // 2. Subscribe to your personal notification channel
    stompClient.subscribe('/user/queue/notifications', (message) => {
      const notification = JSON.parse(message.body);
      console.log('Notification received:', notification);
    });
  },
  (error) => {
    console.error('Connection error:', error);
  },
);
```

2. **Subscribe** to `/user/queue/notifications` — the server automatically routes messages only to the authenticated user's session.

3. **Receive** notifications as JSON payloads matching the `NotificationResponse` schema (see below).

### Notification Payload Schema

```json
{
  "id": "string",
  "title": "string",
  "content": "string",
  "sender": "string",
  "receiver": "string (userId)",
  "channel": "SYSTEM | ROADMAP | QUIZ | PAYMENT | PROFILE",
  "readAt": "ISO-8601 timestamp or null",
  "createdAt": "ISO-8601 timestamp"
}
```

### Disconnect

Call `stompClient.disconnect()` when the session ends. The server automatically removes the Redis Pub/Sub subscription for the user on disconnect.

---

## Redis Stream Integration

Notifications are delivered to users via an internal Redis Stream → Pub/Sub pipeline:

1. **Producer** (another service) writes to Redis Stream `notification_stream` with fields: `type`, `userId`, `metadata` (JSON).
2. **Consumer** (`NotificationStreamConsumer`) reads from the stream, resolves the template, persists to the database, and publishes to `user_notif:{userId}` on Redis Pub/Sub.
3. **WebSocket publisher** (`WebSocketNotificationPublisher`) forwards the Pub/Sub message to the user's STOMP `/user/queue/notifications` destination.

### Stream Message Format

| Field      | Type        | Description                                                                          |
| ---------- | ----------- | ------------------------------------------------------------------------------------ |
| `type`     | String      | Template key (e.g., `SYSTEM`, `ROADMAP_ASSIGNED`)                                    |
| `userId`   | String      | Recipient's Keycloak subject ID                                                      |
| `metadata` | JSON String | Placeholder values for template substitution (e.g., `{"roadmapTitle":"My Roadmap"}`) |

### Retry & Dead-Letter Queue

Failed or unacknowledged messages are recovered automatically:

- **Recovery interval**: every 60 s
- **Idle threshold**: 30 s (message must be pending without ACK for 30 s before recovery)
- **Max retries**: 3 attempts
- **Dead-letter queue**: `notification_stream_dlq` (capped at 1 000 entries, approximate) — messages exceeding max retries are moved here with `original_id` and `failure_reason` fields.
