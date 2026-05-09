# API Gateway Architecture

## Role in the System

The API Gateway sits between all clients and the backend microservices. Every HTTP request from the frontend or admin panel passes through here before being forwarded to the appropriate downstream service.

```
Clients (browser, mobile)
         │
         │ HTTPS :443 → :8090
         ▼
┌─────────────────────────────────┐
│          API Gateway            │
│  Port: 8090                     │
│                                 │
│  ┌───────────────────────────┐  │
│  │  OAuth2 Resource Server   │  │
│  │  (Keycloak JWT validation)│  │
│  └───────────────────────────┘  │
│                │                │
│  ┌─────────────▼─────────────┐  │
│  │  Spring Cloud Gateway     │  │
│  │  Route matching + forward │  │
│  └───────────────────────────┘  │
└─────────────────────────────────┘
         │
    ┌────┴─────────────────────────────────────┐
    ▼           ▼           ▼           ▼      ▼
 :8082       :8083       :8091       :8087   :8088
Profile    Roadmap   Notification  Payment  Storage
Service    Service    Service      Service  Service
```

## Request Processing Pipeline

```
Incoming Request
      │
      ▼
┌─────────────────────────────────────────────────────┐
│  1. CORS Filter (CorsDeduplicationFilter)            │
│     Deduplicates CORS headers from upstream services │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  2. Security Filter Chain (Order 0)                  │
│     Swagger/API-docs endpoints → permitAll           │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  3. Security Filter Chain (Order 1)                  │
│     PUBLIC_ENDPOINTS → permitAll                     │
│     All others → OAuth2 JWT validation               │
│                                                      │
│  Token Resolution (cookieBearerTokenResolver):       │
│    1. Authorization: Bearer header                   │
│    2. ?token= query parameter (SSE EventSource)      │
│    3. access_token cookie (legacy)                   │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────┐
│  4. Spring Cloud Gateway Router                      │
│     Route matching by path prefix                    │
│     Apply filters (e.g., RewritePath for /roadmap)  │
│     Forward to downstream service                    │
└──────────────────────────┬──────────────────────────┘
                           │
                           ▼
                   Downstream Service
```

## Security Model

The gateway is the single JWT validation point. Downstream services trust that if a request arrived through the gateway, it is authenticated. Services may additionally validate the forwarded token for role extraction.

- **JWT Issuer**: Keycloak (`KEYCLOAK_ISSUER_URI`)
- **Signature validation**: via Keycloak JWKS endpoint (auto-discovered from issuer URI)
- **No session state**: stateless — each request is validated independently

## CORS Strategy

CORS is configured once at the gateway level. Downstream services should not add their own CORS headers (the `CorsDeduplicationFilter` removes duplicates to prevent browser errors).

## Routing Table

| Route ID              | Path Predicate    | Upstream URI          | Path Rewrite                          |
| --------------------- | ----------------- | --------------------- | ------------------------------------- |
| `profile-service`     | `/profile/**`     | `$PROFILE_SERVICE_URI`| none                                  |
| `roadmap-service`     | `/roadmap/**`     | `$ROADMAP_SERVICE_URI`| `/roadmap/{seg}` → `/road-map/{seg}`  |
| `notification-service`| `/notification/**`| `$NOTIFICATION_SERVICE_URI` | none                         |
| `payment-service`     | `/payment/**`     | `$PAYMENT_SERVICE_URI`| none                                  |
| `storage-service`     | `/storage/**`     | `$STORAGE_SERVICE_URI`| none                                  |

## Aggregated Swagger UI

The gateway proxies API doc endpoints from all downstream services and presents them in a single Swagger UI at `/swagger-ui.html`. This avoids the need for developers to access each service directly.
