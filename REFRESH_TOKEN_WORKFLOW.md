# Refresh Token Workflow

This document describes the current refresh token flow implemented in the codebase today.

## Scope

The refresh flow currently spans these components:

- `memap-frontend`: detects expired sessions and triggers refresh.
- `api-gateway`: accepts the refresh endpoint as a public route and reads `access_token` from cookies for protected routes.
- `profile-service`: validates the refresh token, exchanges it with the identity provider, rotates stored tokens, and returns new cookies.
- External identity provider: Keycloak-compatible OpenID Connect token endpoint used by `profile-service`.

## Current Entry Points

- Login: `POST /profile/authentication/login`
- Refresh: `POST /profile/authentication/refresh-token`
- Logout: `POST /profile/authentication/logout`

All of these requests are sent by the browser to the API gateway. The gateway forwards `/profile/**` traffic to `profile-service`.

## Cookies Used

`profile-service` issues two cookies:

- `access_token`: `maxAgeAccessToken = 600` seconds (10 minutes)
- `refresh_token`: `maxAgeRefreshToken = 604800` seconds (7 days)

Cookie settings come from `profile-service/src/main/resources/application.yaml`:

- `HttpOnly = true`
- `Secure = true`
- `SameSite = None`
- `Path = /`

Because the frontend uses `withCredentials: true`, the browser includes these cookies on gateway requests.

## End-to-End Flow

```text
Browser
  -> POST /profile/authentication/login
  -> API Gateway
  -> profile-service
  -> Keycloak token endpoint
  -> profile-service stores access_token + refresh_token pair
  -> profile-service returns Set-Cookie(access_token, refresh_token)

Later protected request
  -> Browser sends cookies
  -> API Gateway reads access_token cookie
  -> Downstream service responds 401 when token is expired/invalid
  -> Frontend interceptor calls POST /profile/authentication/refresh-token
  -> profile-service validates refresh_token cookie against DB
  -> profile-service exchanges refresh token with Keycloak
  -> profile-service deletes old stored refresh token
  -> profile-service stores new access_token + refresh_token pair
  -> profile-service returns new Set-Cookie(access_token, refresh_token)
  -> Frontend retries the original request
```

## Detailed Workflow

### 1. Login creates the initial session

The frontend calls `authenticateService.login()`.

`profile-service` then:

1. Calls `TokenExchangeService.getTokenForUser(...)`.
2. Exchanges username/password with the identity provider token endpoint.
3. Receives `access_token` and `refresh_token`.
4. Saves both values in the `TokenModel` table.
5. Returns both tokens as secure cookies.

Important implementation detail:

- The database stores both the access token and refresh token together.
- Refresh validation is therefore stateful, not purely JWT-based.

## 2. Protected requests use the access token cookie

For normal authenticated API calls:

1. The browser sends `access_token` and `refresh_token` cookies.
2. `api-gateway` checks the `Authorization` header first.
3. If no bearer header exists, it falls back to the `access_token` cookie.
4. Public endpoints such as `/profile/authentication/login` and `/profile/authentication/refresh-token` bypass access-token extraction.

This means the current browser session is cookie-based even though services still validate JWTs.

## 3. Frontend decides when to refresh

The refresh trigger is currently implemented in `memap-frontend/src/service/axios-instance.ts`.

When a response comes back as HTTP `401`:

1. The interceptor checks the backend error `code`.
2. Only `1003`, `2003`, and `2004` enter the refresh path.
3. If a refresh is already in progress, the request is queued.
4. Otherwise, the frontend calls `authenticateService.refreshToken()`.

This is important: the gateway does not automatically refresh expired access tokens in the current implementation. The browser client is responsible for initiating refresh.

## 4. Refresh request path

The frontend sends:

- `POST /profile/authentication/refresh-token`
- no JSON payload
- cookies included via `withCredentials: true`

The refresh token is taken from the `refresh_token` cookie inside `profile-service`, not from the request body.

## 5. profile-service validates and rotates the refresh token

`AuthenticationService.getNewAccessToken(...)` performs the refresh flow:

1. Read `refresh_token` from the incoming cookies.
2. If the cookie is missing, throw `UNAUTHENTICATED`.
3. Check that the refresh token exists in `TokenRepository`.
4. Call `TokenExchangeService.getNewAccessTokenForUser(refreshToken)`.

`TokenExchangeService.getNewAccessTokenForUser(...)` then:

1. Calls the identity provider token endpoint with the refresh token grant.
2. Receives a new `access_token` and a new `refresh_token`.
3. Deletes the old refresh token from local persistence.
4. Saves the new access/refresh token pair.
5. Returns the new token pair to `AuthenticationService`.

Finally, `AuthenticationService`:

1. Creates new `access_token` and `refresh_token` cookies.
2. Returns them through `Set-Cookie` headers.
3. Sends an empty response body with HTTP `200`.

This is refresh token rotation. After a successful refresh, the old stored refresh token is no longer valid in local persistence.

## 6. Frontend resumes blocked requests

If refresh succeeds:

1. The frontend marks refresh as completed.
2. Queued requests are released.
3. The original failed request is retried.

If refresh fails:

1. The frontend removes `PROFILE` from local storage.
2. The browser is redirected to `/auth`.
3. The original request stays failed.

## 7. Logout invalidates both local and remote session state

Logout is related to the same token lifecycle:

1. `profile-service` reads `access_token` from cookies.
2. It looks up the associated refresh token from local persistence.
3. It calls the identity provider logout endpoint using the refresh token.
4. On success, it deletes the stored refresh token.
5. It blacklists the access token in Redis.
6. It deletes both cookies in the response.

This means logout clears:

- browser cookies
- local refresh-token persistence
- access-token reuse through Redis blacklist
- remote identity provider refresh session

## Failure Cases

### Missing refresh cookie

If `refresh_token` is missing, `profile-service` throws `UNAUTHENTICATED` immediately.

### Refresh token not found in persistence

If the cookie exists but the token is not found in `TokenRepository`, refresh is rejected with `UNAUTHENTICATED`.

### Identity provider rejects refresh

If the identity provider rejects the refresh token, the refresh request fails and the frontend redirects the user back to the auth screen.

### Concurrent expired requests

The frontend uses an in-memory queue to avoid firing multiple refresh requests from the same browser tab at the same time.

## Current Architecture Note

There is a pending OpenSpec proposal under `openspec/changes/refactor-centralize-refresh-at-gateway/` that describes moving refresh handling into `api-gateway`.

That proposal is not the current behavior.

As of the code in this repository today:

- refresh is initiated by the frontend interceptor
- refresh is executed by `profile-service`
- token rotation is persisted in `profile-service`
- `api-gateway` only permits the refresh endpoint and reads the access token cookie

## Code Locations

- `memap-frontend/src/service/axios-instance.ts`
- `memap-frontend/src/service/authenticate.service.ts`
- `memap-frontend/src/handlers/token-refresh-handler.ts`
- `api-gateway/src/main/java/com/memap/apigateway/config/security/SecurityConfig.java`
- `profile-service/src/main/java/com/techmap/profileservice/controller/identity/AuthenticationController.java`
- `profile-service/src/main/java/com/techmap/profileservice/service/AuthenticationService.java`
- `profile-service/src/main/java/com/techmap/profileservice/service/identity/TokenExchangeService.java`
- `profile-service/src/main/java/com/techmap/profileservice/service/TokenService.java`
- `profile-service/src/main/java/com/techmap/profileservice/repository/TokenRepository.java`
- `profile-service/src/main/java/com/techmap/profileservice/entity/TokenModel.java`
- `profile-service/src/main/resources/application.yaml`
