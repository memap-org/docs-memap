# Authentication - Quick Start Guide

## Using Authentication in Your Components

### Simple: Fetch Data with Automatic Token

```typescript
// src/pages/MyRoadmaps.tsx
import { useEffect, useState } from 'react';
import { RoadmapService } from '../service/roadmap.service';

export function MyRoadmaps() {
  const [roadmaps, setRoadmaps] = useState([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    RoadmapService.getMyRoadmap({ page: 0, size: 20 })
      .then(response => {
        setRoadmaps(response.result.data);
      })
      .catch(error => {
        console.error('Failed to load roadmaps:', error);
      })
      .finally(() => setLoading(false));
  }, []);

  if (loading) return <div>Loading...</div>;
  return (
    <div>
      {roadmaps.map(roadmap => (
        <div key={roadmap.roadmapId}>{roadmap.title}</div>
      ))}
    </div>
  );
}
```

**That's it!** No token management needed. Token is attached automatically.

---

## How the Token Gets There (Behind the Scenes)

```
Your Code                axios-instance.ts                  API
  │                            │                            │
  │ RoadmapService.getMyRoadmap()                           │
  │──────────────────────────────→ check auth               │
  │                            │ Keycloak.authenticated?    │
  │                            │ → YES                      │
  │                            │                            │
  │                            │ refresh if needed          │
  │                            │ (expires in 30s)           │
  │                            │                            │
  │                            │ attach Authorization       │
  │                            │ header with token         │
  │                            │                            │
  │                            │ GET /roadmap/api/roadmap/my-roadmap
  │                            │─────────────────────────→  validate JWT
  │                            │                         ← 200 OK
  │ ← response.result.data ← ← response                  │
```

---

## Common Tasks

### Task 1: Check if User is Logged In

```typescript
import keycloak from '../service/keycloak';

if (keycloak.authenticated) {
  console.log('User:', keycloak.idTokenParsed?.name);
} else {
  console.log('Not logged in');
}
```

### Task 2: Get Current User ID

```typescript
const userId = keycloak.subject;  // User ID from JWT
const userName = keycloak.idTokenParsed?.name;
```

### Task 3: Make Authenticated POST Request

```typescript
// Create a new roadmap
const newRoadmap = await RoadmapService.createRoadmap({
  title: 'My New Roadmap',
  description: 'Learn React',
});

console.log('Created:', newRoadmap.result.roadmapId);
```

### Task 4: Handle Errors

```typescript
try {
  const data = await RoadmapService.getRoadmapById('123');
} catch (error) {
  if (error.response?.status === 404) {
    console.log('Roadmap not found');
  } else if (error.response?.status === 403) {
    console.log('You do not have permission');
  } else {
    console.log('Error:', error.message);
  }
}
```

### Task 5: Check Token Status

```typescript
import keycloak from '../service/keycloak';

// Is token expired?
if (keycloak.isTokenExpired()) {
  console.log('Token expired, will refresh on next request');
}

// When does token expire?
const expiresAt = keycloak.tokenParsed?.exp;
console.log('Expires at:', new Date(expiresAt * 1000));
```

---

## Service Reference

### All Available Services

```typescript
// Roadmap
RoadmapService.getMyRoadmap(params)
RoadmapService.getRoadmapById(id)
RoadmapService.createRoadmap(data)
RoadmapService.deleteRoadmap(id)

// User
UserService.getProfile()
UserService.updateProfile(data)

// Quiz
QuizService.getMyQuizzes(params)
QuizService.createQuiz(data)

// Assignment
AssignmentService.getMyAssignments(params)

// Payment
PaymentService.getSubscription()
```

See [API.md](./API.md) for complete reference.

---

## What the Axios Interceptor Does (Automatically)

### On Every Request ↓

```typescript
// 1. Check: Is user logged in?
if (!keycloak.authenticated) {
  return config;  // Skip token for public endpoints
}

// 2. Refresh: Is token expiring soon?
await keycloak.updateToken(30);  // Refresh if expires in 30s

// 3. Attach: Add token to header
config.headers.Authorization = `Bearer ${keycloak.token}`;

// 4. Send: Make the request
return config;
```

### On Every Response ↓

```typescript
// 1. Success (200-299)?
if (response.status < 300) {
  return response.data;  // ✅ Done
}

// 2. Unauthorized (401)?
if (error.status === 401) {
  // Force refresh & retry once
  await keycloak.updateToken(-1);
  return axiosInstance(originalRequest);
}

// 3. Other error?
return Promise.reject(error);
```

---

## Performance: Token Refresh Batching

When 3 requests happen at once:

**❌ Without batching** (makes 3 refresh calls):
```
Request 1 → Refresh token → Attach token → Call API
Request 2 → Refresh token → Attach token → Call API  (duplicate!)
Request 3 → Refresh token → Attach token → Call API  (duplicate!)
```

**✅ With batching** (makes 1 refresh call):
```
Request 1 ┐
Request 2 ├─→ Refresh token (once) → All share new token → Call APIs
Request 3 ┘
```

**Result**: 3x fewer token refreshes = faster, less load on Keycloak

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "401 Unauthorized" on first request | Interceptor will auto-refresh token |
| Token stays in memory (not disk) | By design (secure), re-login on browser refresh |
| Can't access user info | Use `keycloak.idTokenParsed` for decoded JWT |
| Getting CORS error with Bearer token | API Gateway needs to accept Authorization header |

---

## Testing Token Refresh

Open DevTools → Network tab, make requests:

```typescript
// This will trigger token refresh if expiring soon
RoadmapService.getMyRoadmap({ page: 0, size: 20 });
```

Look for:
- ✅ Request has `Authorization: Bearer ...` header
- ✅ Response status is 200
- ✅ No `Content-Type: multipart/form-data` for auth refresh

---

## Summary

| What | How |
|------|-----|
| **Attach token to requests** | Automatic (request interceptor) |
| **Refresh expired token** | Automatic (every 30s before expiry) |
| **Retry on 401** | Automatic (response interceptor) |
| **Logout user on failure** | Automatic (redirect to login) |
| **Your job** | Call `RoadmapService.getXYZ(...)` and handle response |

**No token management code in components!** ✅

---

## Next: Read Full Docs

- [AUTH_IMPLEMENTATION.md](./AUTH_IMPLEMENTATION.md) — Deep dive
- [AUTH_OPTIMIZATION_SUMMARY.md](./AUTH_OPTIMIZATION_SUMMARY.md) — What changed
- [API.md](./API.md) — Complete API reference
