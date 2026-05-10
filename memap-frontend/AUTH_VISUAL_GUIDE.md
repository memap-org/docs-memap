# Authentication - Visual Implementation Guide

## 1. How Token Flows Through the App

```
┌─────────────────────────────────────────────────────────────────┐
│                   REACT COMPONENT                               │
│                                                                  │
│  function MyRoadmaps() {                                        │
│    const [roadmaps, setRoadmaps] = useState([]);                │
│                                                                  │
│    useEffect(() => {                                            │
│      RoadmapService.getMyRoadmap({ page: 0, size: 20 })        │
│        .then(response => setRoadmaps(response.result.data))     │
│    }, []);                                                       │
│  }                                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ calls
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SERVICE LAYER                                 │
│                                                                  │
│  export const RoadmapService = {                                │
│    getMyRoadmap: (params) =>                                    │
│      axiosInstance.get('/roadmap/api/roadmap/my-roadmap', ....) │
│  }                                                               │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ routes through
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              REQUEST INTERCEPTOR                                │
│                                                                  │
│  1️⃣  if (!keycloak.authenticated) → skip                        │
│  2️⃣  await keycloak.updateToken(30) → refresh if expires soon   │
│  3️⃣  config.headers.Authorization = `Bearer ${token}`           │
│  4️⃣  return config                                              │
│                                                                  │
│  ✅ Result: config with Authorization header                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ sends with header:
                         │ Authorization: Bearer eyJhbGc...
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   API GATEWAY                                   │
│                   (port 8280)                                   │
│                                                                  │
│  GET /roadmap/api/roadmap/my-roadmap                            │
│  Authorization: Bearer eyJhbGc...                               │
│                                                                  │
│  ✅ Validates JWT signature (via Keycloak JWKS)                 │
│  ✅ Extracts user ID from token: jwt.subject                    │
│  ✅ Routes to: http://localhost:8083/api/roadmap/my-roadmap     │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ backend processes
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                 ROADMAP SERVICE                                 │
│                 (port 8083)                                     │
│                                                                  │
│  GET /api/roadmap/my-roadmap                                    │
│  Authorization: Bearer eyJhbGc...                               │
│                                                                  │
│  ✅ Validates token (Spring Security @EnableResourceServer)    │
│  ✅ Loads user roadmaps from MongoDB                            │
│  ✅ Returns: { code: 1000, result: { ... } }                   │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ response with data
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              RESPONSE INTERCEPTOR                               │
│                                                                  │
│  if (status < 300) → return response.data ✅                    │
│  if (status === 401) → refresh & retry ✅                       │
│  else → reject with error ❌                                    │
│                                                                  │
│  ✅ Result: { code: 1000, result: { ... } }                    │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ returns to service
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   SERVICE LAYER                                 │
│                                                                  │
│  Promise resolves with: { code: 1000, result: [...] }           │
│                         ↑                                        │
│                  (response.data auto-extracted)                 │
└────────────────────────┬────────────────────────────────────────┘
                         │
                         │ returns to component
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│                   REACT COMPONENT                               │
│                                                                  │
│  .then(response =>                                              │
│    setRoadmaps(response.result.data)  ✅ Done!                  │
│  )                                                               │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Token Refresh Batching (Performance)

### Scenario: 3 Concurrent Requests

```
Timeline
─────────────────────────────────────────────────────────────────

t=0ms
  Request 1 arrives
  │ if (!refreshPromise) → TRUE
  │ Create refresh promise
  │ refreshPromise = keycloak.updateToken(30)
  ▼

t=5ms
  Request 2 arrives (while Request 1 refreshing)
  │ if (!refreshPromise) → FALSE (already refreshing!)
  │ Reuse existing promise
  │ await refreshPromise (shared)
  ▼

t=10ms
  Request 3 arrives (while Request 1 & 2 refreshing)
  │ if (!refreshPromise) → FALSE
  │ Reuse existing promise
  │ await refreshPromise (shared)
  ▼

t=200ms
  Token refresh completes
  │ refreshPromise.finally() → sets refreshPromise = null
  │ All 3 requests continue with fresh token
  ▼

t=210ms
  All 3 requests complete
  │ Request 1: ✅ Roadmap data
  │ Request 2: ✅ Quiz data
  │ Request 3: ✅ User data
  ▼

Result:
──────
❌ Without batching: 3 token refreshes (3 × 200ms = 600ms overhead)
✅ With batching: 1 token refresh (1 × 200ms = 200ms overhead)

Savings: 2 × 200ms = 400ms per batch!
```

---

## 3. Error Handling Flow

```
API Call Error

  │
  ├─→ Status: 401 (Unauthorized)
  │     │
  │     ├─→ _retry flag exists? NO
  │     │     │
  │     │     ├─→ Set _retry = true
  │     │     ├─→ Force token refresh: keycloak.updateToken(-1)
  │     │     ├─→ Update Authorization header
  │     │     ├─→ Retry original request
  │     │     └─→ Respond with retry result
  │     │
  │     └─→ _retry flag exists? YES
  │           └─→ Reject error (prevent infinite loop)
  │
  ├─→ Status: 404 (Not Found)
  │     └─→ Reject error immediately
  │
  ├─→ Status: 403 (Forbidden)
  │     └─→ Reject error immediately
  │
  └─→ Status: 5xx (Server Error)
        └─→ Reject error immediately
```

---

## 4. Token Lifecycle

```
                    Token Lifecycle
        ┌───────────────────────────────────┐
        │                                   │
        │  Token issued by Keycloak         │
        │  Valid for: 5 minutes             │
        │                                   │
        └────────────┬──────────────────────┘
                     │
                     ▼
        ┌─────────────────────────────────────────┐
        │ t=0s to t=240s: Token Valid             │
        │                                         │
        │ At t=220s (≤30s until expiry):          │
        │ ┌─────────────────────────────────────┐ │
        │ │ Request arrives                     │ │
        │ │ Request Interceptor: Refresh Token  │ │
        │ │ Keycloak.updateToken(30) → true    │ │
        │ │ Continue with fresh token ✅        │ │
        │ └─────────────────────────────────────┘ │
        │                                         │
        └────────────┬──────────────────────────────┘
                     │
                     ▼
        ┌──────────────────────────────┐
        │ t=240s: Old Token Expired    │
        │                              │
        │ Request arrives:             │
        │ ┌──────────────────────────┐ │
        │ │ Attach old token        │ │
        │ │ API returns: 401        │ │
        │ │ Response Interceptor:   │ │
        │ │ Force refresh (-1)      │ │
        │ │ Get new token ✅        │ │
        │ │ Retry request ✅        │ │
        │ └──────────────────────────┘ │
        │                              │
        └──────────────────────────────┘
```

---

## 5. Authentication Decision Tree

```
                      User Action
                         │
                         ▼
                   ┌──────────────┐
                   │ User clicks  │
                   │ an API call  │
                   └──────┬───────┘
                          │
                          ▼
                  ┌────────────────────┐
                  │ Keycloak knows     │
                  │ about this user?   │
                  └─────┬──────────┬───┘
                        │          │
                     YES│          │NO
                        │          ▼
                        │    ┌──────────────────┐
                        │    │ Redirect to      │
                        │    │ Keycloak login   │
                        │    │ page             │
                        │    └──────────────────┘
                        │
                        ▼
        ┌───────────────────────────────────┐
        │ Does token need refresh?          │
        │ (expires in ≤30s)                 │
        └─────┬─────────────────────────┬───┘
              │                         │
           YES│                         │NO
              │                         │
              ▼                         │
        ┌──────────────────┐           │
        │ Call Keycloak:   │           │
        │ updateToken(30)  │           │
        └─────┬────────────┘           │
              │                         │
              ▼                         ▼
        ┌──────────────────────────────────┐
        │ Attach new token to header:      │
        │ Authorization: Bearer {...}      │
        │                                  │
        │ Send HTTP request                │
        └─────────────┬────────────────────┘
                      │
              ┌───────┴────────┐
              │                │
           ✅ │                │ ❌
        Status │                │ Status
        200-299│                │ 401
              │                │
              ▼                ▼
        Return data    Force refresh (-1)
        to component   + Retry once
                       (handled automatically)
```

---

## 6. Code Organization

```
src/
├── service/
│   ├── axios-instance.ts          ← Token attachment & refresh
│   ├── keycloak.ts                ← Keycloak init
│   ├── authenticate.service.ts    ← Login/logout endpoints
│   ├── roadmap.service.ts         ← Roadmap API methods
│   ├── user.service.ts            ← User API methods
│   └── [other].service.ts         ← Other API methods
│
├── components/
│   └── MyComponent.tsx            ← Uses RoadmapService.getXYZ()
│
├── pages/
│   └── MyPage.tsx                 ← Uses RoadmapService.getXYZ()
│
└── types/
    └── dto/
        ├── api-response.dto.ts    ← API response types
        └── [domain].dto.ts        ← Domain-specific types
```

**Flow**: Component → Service → axios-instance → API Gateway → Backend

---

## 7. What Actually Gets Sent

### Request to Backend

```http
GET /roadmap/api/roadmap/my-roadmap?page=0&size=20 HTTP/1.1
Host: localhost:8280
Authorization: Bearer eyJhbGciOiJSUzI1NiIsInR5cC...
Content-Type: application/json
```

**Note**: Token was attached by Request Interceptor automatically!

### Response from Backend

```json
{
  "code": 1000,
  "result": {
    "data": [
      { "roadmapId": "123", "title": "Learn React" },
      { "roadmapId": "456", "title": "Learn TypeScript" }
    ],
    "currentPage": 0,
    "totalPages": 1,
    "pageSize": 20,
    "totalElements": 2
  }
}
```

**Note**: Component receives only `response.result` (response.data is auto-extracted)!

---

## 8. Common Scenarios

### Scenario 1: Happy Path (Token Fresh)

```
Component calls: RoadmapService.getMyRoadmap()
  ↓
Request Interceptor:
  keycloak.authenticated? YES
  keycloak.isTokenExpired(30)? NO (fresh)
  → Just attach token
  ↓
API returns: 200 OK
  ↓
Response Interceptor:
  status < 300? YES
  → return response.data
  ↓
Component: receives data ✅
```

### Scenario 2: Token Expiring Soon

```
Component calls: RoadmapService.getMyRoadmap()
  ↓
Request Interceptor:
  keycloak.authenticated? YES
  keycloak.isTokenExpired(30)? YES (≤30s)
  → Refresh token
  → Attach new token
  ↓
API returns: 200 OK
  ↓
Response Interceptor:
  status < 300? YES
  → return response.data
  ↓
Component: receives data ✅
```

### Scenario 3: Token Expired (401)

```
Component calls: RoadmapService.getMyRoadmap()
  ↓
Request Interceptor:
  keycloak.authenticated? YES
  keycloak.isTokenExpired(30)? YES
  → Refresh token
  → Attach new token
  ↓
API returns: 200 OK (with refreshed token)
  ↓
Response Interceptor:
  status < 300? YES
  → return response.data
  ↓
Component: receives data ✅

---

BUT if somehow 401 still occurs:

Request Interceptor:
  → Attach token
  ↓
API returns: 401
  ↓
Response Interceptor:
  status === 401 AND !_retry? YES
  → Force refresh (-1)
  → Retry original request
  ↓
API returns: 200 OK
  ↓
Response Interceptor:
  status < 300? YES
  → return response.data
  ↓
Component: receives data ✅
```

---

## Summary Checklist

- ✅ Tokens are attached automatically (no manual work)
- ✅ Tokens refresh proactively (30s before expiry)
- ✅ 401 errors are auto-retried (with fresh token)
- ✅ Concurrent requests batch refresh (efficient)
- ✅ Errors are properly rejected (with details)
- ✅ Memory is cleaned up (no leaks)
- ✅ Code is type-safe (TypeScript)
- ✅ Code is well-documented (3 guides)

**Everything is automatic. Just call services! 🚀**
