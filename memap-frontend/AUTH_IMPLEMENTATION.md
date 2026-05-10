# MeMap Frontend Authentication Implementation

## Overview

The frontend uses **Keycloak** for authentication with an optimized **axios-based interceptor pattern** for token management. This document describes the implementation, how it works, and best practices.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     React Component                             │
└────────────────────────┬────────────────────────────────────────┘
                         │ uses
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│            Service Layer (roadmap.service.ts, etc.)             │
│  - Calls: axiosInstance.get/post/patch/delete(url, data)       │
└────────────────────────┬────────────────────────────────────────┘
                         │ routes through
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│         Axios Instance (src/service/axios-instance.ts)          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Request Interceptor                                      │  │
│  │  1. Check if authenticated with Keycloak                │  │
│  │  2. Refresh token if expires in 30s                     │  │
│  │  3. Attach: Authorization: Bearer {token}               │  │
│  │  4. Send request                                         │  │
│  └──────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Response Interceptor                                     │  │
│  │  1. Success (200-299): Return response.data              │  │
│  │  2. 401 Unauthorized: Force refresh & retry              │  │
│  │  3. Other errors: Reject with error                      │  │
│  └──────────────────────────────────────────────────────────┘  │
└────────────────────────┬────────────────────────────────────────┘
                         │ sends with token
                         ▼
┌─────────────────────────────────────────────────────────────────┐
│              API Gateway (port 8280)                            │
│              ↓                                                  │
│  Validates JWT token (via Keycloak JWKS URI)                   │
│              ↓                                                  │
│  Routes to service: /roadmap/**, /profile/**, etc.             │
└─────────────────────────────────────────────────────────────────┘
```

## Key Components

### 1. Keycloak Configuration

**File**: `src/service/keycloak.ts`

Keycloak is initialized with OAuth2 implicit flow for browser-based apps:
- Realm: configured via environment
- Client ID: public client
- Auto-login: disabled (explicit login required)

### 2. Axios Instance

**File**: `src/service/axios-instance.ts`

#### Request Interceptor
```typescript
// When: Before every HTTP request
// What:
//   1. Check if user is authenticated with Keycloak
//   2. Ensure token is fresh (refresh if expires in 30s)
//   3. Attach: Authorization: Bearer {token}
//   4. Handle concurrent requests (batches refresh)
```

**Performance optimizations**:
- ✅ **Batched token refresh**: Concurrent requests share single refresh promise
- ✅ **Lazy refresh**: Only refreshes if token expires soon
- ✅ **Early exit**: Skips overhead for unauthenticated requests

#### Response Interceptor
```typescript
// When: After every HTTP response
// What:
//   1. Success (200-299): Unwrap response.data (auto-simplification)
//   2. 401 Unauthorized: Force refresh & retry (once)
//   3. Other errors: Reject with error details
```

**Features**:
- ✅ **One-time retry**: Prevents infinite loops with `_retry` flag
- ✅ **Force refresh**: `-1` minValidity forces new token
- ✅ **Graceful logout**: Redirects to login on repeated failures

### 3. Service Layer

All services use the same pattern:

```typescript
// src/service/roadmap.service.ts
export const RoadmapService = {
  getMyRoadmap: (params) => 
    axiosInstance.get('/roadmap/api/roadmap/my-roadmap', { params }),
  
  createRoadmap: (data) => 
    axiosInstance.post('/roadmap/api/roadmap', data),
};
```

**Token attachment happens automatically** — services don't need to handle it.

## Authentication Flow

### 1. Initial Login

```
User clicks Login
  ↓
Redirect to Keycloak login page (browser)
  ↓
User enters credentials
  ↓
Keycloak redirects back with authorization code
  ↓
Frontend exchanges code for tokens (client-side)
  ↓
Store tokens in Keycloak (memory by default, can persist)
  ↓
User redirected to app home
```

### 2. During Usage (Automatic Token Refresh)

```
Component calls: RoadmapService.getMyRoadmap(...)
  ↓
Request Interceptor:
  - Check: keycloak.authenticated? → YES
  - Refresh token if expires in 30s
  - Attach: Authorization: Bearer {token}
  ↓
API Gateway validates JWT signature
  ↓
Request succeeds or fails
  ↓
Response Interceptor:
  - If 401: Force refresh & retry once
  - Otherwise: Return data or error
  ↓
Service promise resolves/rejects
```

### 3. Token Expiration Handling

| Scenario | Action |
|----------|--------|
| Token expires in 30s | Refresh on next request (request interceptor) |
| Token expired, request made | Force refresh & retry (response interceptor) |
| Refresh fails (expired refresh token) | Redirect to login (isRedirecting flag) |

## Best Practices

### ✅ Do

1. **Always use axiosInstance** for API calls
   ```typescript
   // Good
   const response = await axiosInstance.get('/roadmap/123');
   ```

2. **Use services for API grouping**
   ```typescript
   // Good
   RoadmapService.getRoadmapById(id);
   ```

3. **Handle errors in components**
   ```typescript
   // Good
   try {
     const data = await RoadmapService.getMyRoadmap(...);
   } catch (error) {
     showErrorToast(error.message);
   }
   ```

4. **Check authentication before sensitive operations**
   ```typescript
   // Good
   if (keycloak.authenticated) {
     // Call API
   }
   ```

### ❌ Don't

1. **Don't create custom axios instances**
   ```typescript
   // Bad - bypasses token attachment
   const customAxios = axios.create({ ... });
   ```

2. **Don't manually attach tokens**
   ```typescript
   // Bad - redundant, interceptor already does this
   headers: { Authorization: `Bearer ${keycloak.token}` }
   ```

3. **Don't call Keycloak directly from components**
   ```typescript
   // Bad - couples component to Keycloak API
   keycloak.updateToken(30);
   ```

4. **Don't store tokens in localStorage** (Keycloak handles it)
   ```typescript
   // Bad - XSS vulnerability
   localStorage.setItem('token', keycloak.token);
   ```

## Troubleshooting

### Issue: Token always expired
**Solution**: Check Keycloak server time synchronization with client

### Issue: Infinite redirect loop
**Solution**: `isRedirecting` flag prevents this — check Keycloak realm configuration

### Issue: CORS errors with Bearer token
**Solution**: Verify API Gateway accepts Authorization header
- Check: `Access-Control-Allow-Headers: Authorization`

### Issue: 401 on first request after login
**Solution**: Token refresh in request interceptor handles this automatically

## Type Definitions

### ApiResponse<T>
```typescript
interface ApiResponse<T> {
  code: number;        // 1000 = success, 4xxx/3xxx = error
  message?: string;
  result: T;
}
```

### PageResponse<T>
```typescript
interface PageResponse<T> {
  data: T[];
  currentPage: number;
  totalPages: number;
  pageSize: number;
  totalElements: number;
}
```

## Performance Metrics

| Metric | Value | Note |
|--------|-------|------|
| Token refresh batching | 1 refresh for N concurrent requests | O(1) instead of O(N) |
| Token refresh cycle | 30s before expiry | Proactive, prevents 401s |
| 401 retry attempts | 1 | Prevents infinite loops |
| Response unwrapping | Automatic | Extracts `response.data` |

## Environment Variables

Set in `.env`:
```bash
VITE_MEMAP_BACKEND_API=http://localhost:8280
VITE_KEYCLOAK_URL=http://localhost:8080
VITE_KEYCLOAK_REALM=memap
VITE_KEYCLOAK_CLIENT_ID=memap-frontend
```

## Related Files

| File | Purpose |
|------|---------|
| `src/service/axios-instance.ts` | Core interceptor logic |
| `src/service/keycloak.ts` | Keycloak initialization |
| `src/service/*.service.ts` | API service methods |
| `src/constants/api-config.ts` | API configuration |
| `docs/memap-frontend/API.md` | API integration guide |

## Summary

The auth implementation is **optimized for performance and security**:
- ✅ Keycloak handles token lifecycle (issue, refresh, revoke)
- ✅ Axios interceptors transparently manage tokens on every request
- ✅ Token refresh is batched for concurrent requests
- ✅ 401 responses are automatically retried with fresh token
- ✅ Components don't need to handle token management

**Services + axiosInstance = automatic, performant authentication for all API calls.**
