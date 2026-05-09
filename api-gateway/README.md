# API Gateway

## Overview

The API Gateway is the single entry point for all client requests in the MeMap platform. It handles routing, authentication (OAuth2/JWT via Keycloak), CORS, and exposes an aggregated Swagger UI for all downstream services.

## Port

| Protocol | Port |
| -------- | ---- |
| HTTP     | 8090 |

## Technology Stack

- **Spring Boot** + **Spring Cloud Gateway** (WebMVC mode)
- **Spring Security OAuth2 Resource Server** — JWT validation via Keycloak JWKS
- **SpringDoc** — Aggregated Swagger UI

## Routes

| Path prefix       | Downstream service    | Rewrite rule                              |
| ----------------- | --------------------- | ----------------------------------------- |
| `/profile/**`     | Profile Service :8082 | none                                      |
| `/roadmap/**`     | Roadmap Service :8083 | `/roadmap/{seg}` → `/road-map/{seg}`      |
| `/notification/**`| Notification Svc :8091| none                                      |
| `/payment/**`     | Payment Service :8087 | none                                      |
| `/storage/**`     | Storage Service :8088 | none                                      |

## Authentication

The gateway validates JWT tokens issued by Keycloak. Token resolution order:

1. `Authorization: Bearer <token>` header
2. `?token=<token>` query parameter (for SSE `EventSource` connections)
3. `access_token` HTTP-only cookie (legacy / direct access)

### Public Endpoints (no auth required)

```
POST /profile/authentication/login
POST /profile/authentication/refresh-token
POST /profile/users/register
POST /profile/users/forgot-password
POST /profile/invitations/accept
GET  /profile/invitations/validate
GET  /storage/file/*/access
```

## Swagger UI

Aggregated Swagger UI is available at:

```
GET /swagger-ui.html
```

Individual service API docs are proxied:

| Service              | Docs URL                   |
| -------------------- | -------------------------- |
| Profile Service      | `/profile/api-docs`        |
| Roadmap Service      | `/roadmap/api-docs`        |
| Notification Service | `/notification/v3/api-docs`|
| Payment Service      | `/payment/api-docs`        |
| Storage Service      | `/storage/api-docs`        |

## Environment Variables

| Variable                  | Default                      | Description                        |
| ------------------------- | ---------------------------- | ---------------------------------- |
| `KEYCLOAK_ISSUER_URI`     | (required)                   | Keycloak realm issuer URI for JWT  |
| `PROFILE_SERVICE_URI`     | `http://localhost:8082`      | Profile service base URL           |
| `ROADMAP_SERVICE_URI`     | `http://localhost:8083`      | Roadmap service base URL           |
| `NOTIFICATION_SERVICE_URI`| `http://localhost:8091`      | Notification service base URL      |
| `PAYMENT_SERVICE_URI`     | `http://localhost:8087`      | Payment service base URL           |
| `STORAGE_SERVICE_URI`     | `http://localhost:8088`      | Storage service base URL           |

## CORS

Allowed origins (configured in `SecurityConfig`):

- `http://localhost:3000`, `http://localhost:5173`, `http://localhost:4000`
- `https://roadmap.memap.id.vn` (main frontend)
- `https://admin.memap.id.vn` (admin frontend)
- `https://api.memap.id.vn` (production API)
- `https://prememapp.vercel.app`, `https://memapp.vercel.app`

## Running Locally

```bash
cd api-gateway
./mvnw spring-boot:run
```

Or via Docker Compose from the root:

```bash
docker-compose up api-gateway
```
