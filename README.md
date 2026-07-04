# MeMap — Architecture & Services Reference

MeMap is a roadmap/learning management platform built as a microservices monorepo.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              Client Applications                                │
│   memap-frontend :5173 (React 19+Vite)                         │
│   memap-admin-frontend :4000 (Next.js 16)                      │
└───────────────────────────┬─────────────────────────────────────┘
                            │ HTTP / WebSocket
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│   api-gateway :8090  (Spring Cloud Gateway MVC)                 │
│   JWT validation via Keycloak JWKS · Swagger aggregation        │
└──┬──────────┬──────────┬──────────┬──────────┬──────────┬──────┘
   │          │          │          │          │          │
   ▼          ▼          ▼          ▼          ▼          ▼
:8082      :8083      :8085      :8091      :8087      :8088
profile  roadmap   roadmap-ai  notif.   payment   storage
service  service    service   service   service   service
MySQL    MongoDB    pgvector  MongoDB   MongoDB   MongoDB
+Redis   +Redis     +AMQP     +Redis    +Redis    +MinIO
         gRPC:9083

                    hocuspocus-server :1234
                    (Node.js WebSocket · yjs CRDT)
```

**Inter-service communication:**
- REST via API Gateway (external → services)
- gRPC: roadmap-service (server :9083) ↔ roadmap-ai-service (client)
- RabbitMQ AMQP: roadmap-ai-service + payment-service publish events
- Redis Streams/Pub-Sub: notification-service real-time events

**Auth:** Keycloak JWT — all Spring services are OAuth2 Resource Servers. User ID from `jwt.getSubject()`. Frontends use `@react-keycloak/web`.

## Services

| Service | Port | Stack | DB |
|---|---|---|---|
| api-gateway | 8090 | Spring Cloud Gateway MVC, Java 21 | — |
| profile-service | 8082 | Spring Boot 3.x, JPA | MySQL + Redis |
| roadmap-service | 8083 (gRPC: 9083) | Spring Boot 3.x | MongoDB + Redis |
| roadmap-ai-service | 8085 | Spring Boot 3.x, Spring AI | pgvector + MongoDB + AMQP |
| notification-service | 8091 | Spring Boot 3.x | MongoDB + Redis Streams |
| payment-service | 8087 | Spring Boot 3.x, Stripe SDK | MongoDB + Redis |
| storage-service | 8088 | Spring Boot 3.x | MongoDB + MinIO |
| hocuspocus-server | 1234 | Node.js / TypeScript | MongoDB + Redis |
| memap-frontend | 5173 (dev) | React 19, Vite, TypeScript | — |
| memap-admin-frontend | 4000 (dev) | Next.js 16, Ant Design | — |
| spring-boot-admin-server | 8095 | Spring Boot Admin | — |

## Shared Infrastructure (docker-compose.yml)

| Component | Port | Purpose |
|---|---|---|
| Redis 6 | 6382 | Caching, Pub-Sub, Streams |
| RabbitMQ 4 | 5672 / 15672 | AMQP async messaging |
| MySQL 8 | 3306 | profile-service relational data |
| MongoDB | 27017 | roadmap/notification/payment/storage docs |
| PostgreSQL + pgvector | 5432 | roadmap-ai-service vector embeddings |
| MinIO | 9000 / 9001 | Object storage (S3-compatible) |
| Keycloak | 8080 | Identity Provider (JWT issuer) |

## Quick Start

```bash
# Start everything
docker compose up --build

# Or start just infrastructure
docker compose up -d redis rabbitmq mysql mongodb keycloak
```

## Running Services Individually

```bash
# Java services
cd <service-dir>
./mvnw spring-boot:run          # run
./mvnw test                     # all tests
./mvnw package -DskipTests      # build JAR

# memap-frontend
cd memap-frontend
yarn dev          # port 5173

# memap-admin-frontend
cd memap-admin-frontend
yarn dev          # port 4000

# hocuspocus-server
cd hocuspocus-server
yarn dev          # port 1234
```

## API Documentation

Swagger UI is aggregated at the gateway:

- **Gateway Swagger** — `http://localhost:8090/swagger-ui.html`

Individual service docs (when running locally):

| Service | Swagger URL |
|---|---|
| profile-service | http://localhost:8082/swagger-ui.html |
| roadmap-service | http://localhost:8083/swagger-ui.html |
| roadmap-ai-service | http://localhost:8085/swagger-ui.html |
| notification-service | http://localhost:8091/swagger-ui.html |
| payment-service | http://localhost:8087/swagger-ui.html |
| storage-service | http://localhost:8088/swagger-ui.html |

## Java Service Conventions

All Spring Boot services follow **Clean/Hexagonal Architecture**:

| Layer | Package | Rule |
|---|---|---|
| `domain` | Pure Java models | No Spring, no Lombok, no JPA |
| `application` | Use cases, ports, DTOs | Depends only on `domain` |
| `infrastructure` | JPA/MongoDB entities, adapters | Implements output ports |
| `presentation` | REST controllers | Depends only on input port interfaces |

Key conventions:
- **Constructor injection only** — `@RequiredArgsConstructor` + `final` fields
- **API envelope** — `ApiResponse<T>` with code `1000` for success
- **Error handling** — `AppException(ErrorCode)` + `GlobalExceptionHandler`
- **Tables** — `tbl_` prefix, snake_case columns, Flyway migrations
- **Timestamps** — `Instant` everywhere, never `LocalDateTime`
- **Pagination** — default page size 20

## Error Code Ranges

| Range | Owner |
|---|---|
| 1000 | Success |
| 3xxx | Auth / access errors |
| 4xxx | Client / validation errors |
| 7xxx | Payment / credit / subscription errors |
| 9999 | Uncategorized |

## Environment Variables

Each service reads from a `.env` file (not committed). Required vars per service:

### api-gateway
```
KEYCLOAK_ISSUER_URI
PROFILE_SERVICE_URI, ROADMAP_SERVICE_URI, AI_SERVICE_URI
NOTIFICATION_SERVICE_URI, PAYMENT_SERVICE_URI, STORAGE_SERVICE_URI
```

### profile-service
```
MYSQL_DATASOURCE_URL, MYSQL_USERNAME, MYSQL_PASSWORD
KEYCLOAK_ISSUER_URI, KEYCLOAK_JWK_SET_URI
REDIS_HOST, REDIS_PORT
GATEWAY_URL, PROFILE_SERVICE_URL, PAYMENT_SERVICE_URL
SPRING_BOOT_ADMIN_URL
```

### roadmap-service
```
MONGODB_URL
REDIS_HOST, REDIS_PORT
GATEWAY_URL, ROADMAP_SERVICE_URL
NOTIFICATION_STREAM_KEY, STORAGE_REST_URL
SPRING_BOOT_ADMIN_URL
```

### roadmap-ai-service
```
POSTGRES_URL, POSTGRES_USERNAME, POSTGRES_PASSWORD, POSTGRES_AI_SCHEMA
MONGODB_URL
REDIS_HOST, REDIS_PORT
RABBITMQ_HOST, RABBITMQ_PORT, RABBITMQ_USERNAME, RABBITMQ_PASSWORD
OPENAI_API_KEY, OPENAI_MODEL
TAVILY_API_KEY, GROQ_KEY, GEMINI_KEY
GATEWAY_URL, ROADMAP_AI_SERVICE_URL
SPRING_BOOT_ADMIN_URL
```

### notification-service
```
MONGODB_URI
REDIS_HOST, REDIS_PORT
INTERNAL_API_KEY, SSE_TIMEOUT_MS
STREAM_NAME, GROUP_NAME, CONSUMER_NAME
GATEWAY_URL, NOTIFICATION_SERVICE_URL
SPRING_BOOT_ADMIN_URL
```

### payment-service
```
MONGO_URI
REDIS_HOST, REDIS_PORT
RABBITMQ_HOST, RABBITMQ_PORT, RABBITMQ_USERNAME, RABBITMQ_PASSWORD
KEYCLOAK_ISSUER_URI, KEYCLOAK_JWK_SET_URI
STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET
STRIPE_SUCCESS_URL, STRIPE_CANCEL_URL
STRIPE_PRO_PRICE_ID, STRIPE_ENTERPRISE_PRICE_ID
INTERNAL_API_KEY
GATEWAY_URL, PAYMENT_SERVICE_URL
SPRING_BOOT_ADMIN_URL
```

### storage-service
```
STORAGE_MONGO_URI
RABBITMQ_HOST, RABBITMQ_PORT, RABBITMQ_USERNAME, RABBITMQ_PASSWORD
STORAGE_MINIO_ENDPOINT, STORAGE_MINIO_ACCESS_KEY, STORAGE_MINIO_SECRET_KEY
STORAGE_MINIO_BUCKET, STORAGE_BASE_URL
STORAGE_GRPC_PORT, ROADMAP_GRPC_ADDRESS
INTERNAL_API_KEY, STORAGE_INTERNAL_API_KEY
GATEWAY_URL, STORAGE_SERVICE_URL, PAYMENT_SERVICE_URL
SPRING_BOOT_ADMIN_URL
```

### hocuspocus-server
```
REDIS_HOST, REDIS_PORT
MONGODB_URL
REDIS_STREAM_KEY
HOCUSPOCUS_PORT
```

### memap-frontend
```
VITE_KEYCLOAK_URL, VITE_KEYCLOAK_REALM, VITE_KEYCLOAK_CLIENT_ID
VITE_MEMAP_HOCUSPOCUS_WS_URL
VITE_API_BASE_URL (or equivalent)
```

### memap-admin-frontend
```
NEXT_PUBLIC_KEYCLOAK_URL, NEXT_PUBLIC_KEYCLOAK_REALM, NEXT_PUBLIC_KEYCLOAK_CLIENT_ID
```
