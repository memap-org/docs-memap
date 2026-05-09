# Service Communication Patterns

> Describes all inter-service communication channels in the current (`main`) deployment.
> Derived from GitNexus execution flow traces and source code inspection.

---

## 1. gRPC — profile-service (server)

**Proto file:** `profile-service/src/main/proto/profile/profile_lookup.proto`

Service: `ProfileLookupService`

| RPC | Request | Response | Used by |
|---|---|---|---|
| `GetUserProfile` | `user_id: string` | `{user_id, display_name, email, avatar_url, found, role}` | roadmap-service |
| `GetUserProfiles` | `user_ids: [string]` | `map<user_id, ProfileData>` | roadmap-service |

**Roles returned:** `STUDENT`, `TEACHER`, `ADMIN`

**Consumer:** `roadmap-service/src/main/java/com/techmap/roadmap/grpc/client/ProfileGrpcClientImpl.java`

**Triggered by these flows (GitNexus):**
- `ToggleRoadmapReaction` — fetches reactor's display name
- `AddRoadMapPermission` → `resolveSenderName` — fetches sender display name for notification
- `AddTeacher` → `notifyMemberAdded` — same pattern
- `AddStudent` → `notifyMemberAdded` — same pattern

**Resilience:** `ProfileGrpcClientImpl` catches `StatusRuntimeException` and returns `found=false` instead of throwing. All callers in roadmap-service handle the `found=false` case gracefully.

---

## 2. gRPC — roadmap-service (server, port 9083)

**Consumer:** `roadmap-ai-service/src/main/java/com/techmap/aiservice/grpc/client/RoadmapGrpcClient.java`

| RPC | Request | Response | Used by |
|---|---|---|---|
| `GetRoadmapName` | `roadmap_id: string` | `{name, found}` | ai-service |

Used by ai-service to enrich notification messages and AI-generated content with human-readable roadmap names.

---

## 3. RabbitMQ AMQP — Node Ingest

**Queue:** `${rabbitmq.queue.ai-ingest-node}` (configured in ai-service `application.yml`)

**Publisher:** `roadmap-service` — publishes `IngestNodeMessage` when a roadmap node's content changes
**Consumer:** `roadmap-ai-service/src/main/java/com/techmap/aiservice/messaging/listener/IngestNodeListener.java`

```
Message schema: IngestNodeMessage {
  roadmapId: String
  node: {
    nodeId: String
    ... (node content fields for RAG ingestion)
  }
}
```

**Flow:** When a node is updated in roadmap-service → AMQP message published → `IngestNodeListener` receives it → `DocumentIngestFactory.ingestNode()` embeds node content into pgvector for RAG queries.

---

## 4. Redis Stream — AI → Notification

**Stream key:** `notification_stream` (default, configurable via `notification.stream.key`)

**Publisher:** `roadmap-ai-service/src/main/java/com/techmap/aiservice/messaging/publisher/NotificationStreamPublisher.java`
**Consumer:** `notification-serice` Redis Stream consumer

```
Stream record fields:
  template_key:      String   (notification template identifier)
  origin_service:    String
  receiver_user_id:  String   (Keycloak user ID)
  reference_type:    String
  reference_id:      String
  metadata:          JSON string (Map<String, String>)
```

**Flow:** ai-service completes an operation (e.g., roadmap generation) → publishes record to Redis Stream → notification-serice consumer reads the stream → persists notification to MongoDB → publishes to Redis Pub/Sub → SSE fan-out to connected browser.

---

## 5. Redis Pub/Sub — Notification Fan-out (internal)

**Used by:** notification-serice internally between `SendNotificationService` and `SseRedisSubscriber`

After a notification is persisted, it is published on a Redis Pub/Sub channel. `SseRedisSubscriber` listens on that channel and pushes the event to all active `SseEmitter` instances for the target user.

**Frontend consumption:** `GET /notification/api/v1/notifications/stream` — Server-Sent Events (SSE), `text/event-stream`.

---

## 6. HTTP Feign — profile-service → Keycloak

**Client:** `profile-service/src/main/java/com/techmap/profileservice/repository/identity/IdentityClient.java`

Operations performed against Keycloak Admin REST API:

| Operation | When |
|---|---|
| `exchangeClientToken` | Bootstrap — obtain service client token (cached in Redis) |
| `exchangeUserToken` | User login / token refresh |
| `createUser` | New user registration or invitation provisioning |
| `enableUser` + `updateUser` | Accept invitation via OAuth flow |
| `invalidateToken` | Logout / password change |

**Key flows (GitNexus traces):**
- `ProvisionDisabledIdentity → ExchangeClientToken`: invitation creates disabled Keycloak user
- `AcceptInvitationOAuth → ExchangeClientToken`: enable + update user on invite accept
- `ChangePassword → EvictClientTokenCache`: evict cached token after password change

---

## 7. HTTP — profile-service → notification-serice

**Triggered by:** `EmailController.sendEmail()` in profile-service

**Flow (GitNexus trace `SendEmail → Notification`):**
1. `EmailController.sendEmail()` — profile-service controller
2. `EmailService.sendHtmlEmail()` — sends email via Spring Mail AND
3. `SendNotificationService.send()` — notification-serice creates a notification record

This is a **direct HTTP call** from profile-service to notification-serice's internal endpoint (`/notification/api/v1/notifications/internal`).

---

## 8. RabbitMQ — payment-service → downstream consumers

**Exchange:** `payment.events` (topic)

| Routing Key | Event | Published when |
|---|---|---|
| `credit.granted` | Credits added to a user | Webhook confirm or AddCredits call |
| `credit.consumed` | Credits deducted | Successful ConsumeCredits |
| `subscription.created` | New paid subscription | Stripe subscription.created webhook (stub) |
| `subscription.cancelled` | Subscription cancelled | Stripe subscription.deleted webhook (stub) |

**Subscribers:** None yet — notification-serice and other services will subscribe in follow-up changes.

---

## 9. WebSocket — Frontend → hocuspocus-server

**Protocol:** WebSocket + Yjs (CRDT) via `@hocuspocus/provider` on the frontend  
**Server:** `hocuspocus-server` port 1234

The frontend (`memap-frontend`) uses `@hocuspocus/provider` with `yjs` to maintain a shared document state for collaborative roadmap/node editing. The hocuspocus server persists Yjs binary state to MongoDB and uses Redis pub/sub for multi-instance presence coordination.

---

## Communication Matrix

| From ↓ \ To → | Keycloak | profile | roadmap | ai-service | notification | storage | hocuspocus |
|---|---|---|---|---|---|---|---|
| **api-gateway** | JWT validate | route | route | — | route | route | — |
| **profile-service** | Feign HTTP | — | — | — | HTTP (internal) | — | — |
| **roadmap-service** | JWT validate | **gRPC** | — | AMQP publish | — | — | — |
| **ai-service** | JWT validate | — | **gRPC** | — | Redis Stream | — | — |
| **notification-serice** | JWT validate | — | — | — | — | — | — |
| **memap-frontend** | JWT (cookie) | REST/GW | REST/GW | REST/GW | SSE/GW | REST/GW | **WebSocket** |
