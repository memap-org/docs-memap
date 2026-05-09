# Profile Service API Documentation

> **Branch:** `main` | **Port:** 8082 | **Context path:** `/profile`

---

## Base URLs

```
Via API Gateway:  http://localhost:8090/profile/...
Direct:           http://localhost:8082/profile/...
Swagger UI:       http://localhost:8082/profile/swagger-ui.html
```

---

## Authentication

All endpoints require a valid Keycloak JWT (passed as Bearer token or HTTP-only cookie) **except** `/users/register` and invitation validation/acceptance endpoints.

---

## REST Endpoints

### User Management — `UserController`

Base: `/users`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/users/register` | Public | Register a new user (creates Keycloak identity + profile) |
| GET | `/users/my-profile` | JWT | Get the current user's profile |
| PUT | `/users` | JWT | Update current user profile |
| PATCH | `/users/remove/{id}` | JWT | Soft-remove a user |
| POST | `/users/change-password` | JWT | Change password (calls Keycloak via Feign; evicts Redis token cache) |
| POST | `/users/forgot-password` | Public | Trigger forgot-password email flow |
| POST | `/users/update-avatar` | JWT | Upload and set avatar (`multipart/form-data`) |
| GET | `/users` | JWT | Search users by email/userId (paginated) |
| POST | `/users/bulk` | JWT | Get user info by list of IDs |

**Key flows (GitNexus):**
- `register` → provisions user in Keycloak via `TokenExchangeService.createUser()` → client token exchanged/cached in Redis
- `changePassword` → validates old password via Keycloak token exchange → updates → evicts Redis token cache
- `updateAvatar` → syncs current user from JWT token, resolves email, uploads avatar file

**Search query params for `GET /users`:**
```
email: string (partial match)
userId: string
page: int (0-based)
size: int (default 10)
```

### Invitation Management — `InvitationController`

Base: `/invitations`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/invitations` | JWT | Create an invitation (sends email to invitee) |
| GET | `/invitations` | JWT | List invitations sent by current user (paginated) |
| PATCH | `/invitations/{id}/revoke` | JWT | Revoke a pending invitation |
| POST | `/invitations/{id}/resend` | JWT | Resend invitation email |
| GET | `/invitations/validate` | Public | Validate an invitation token (returns token info) |
| POST | `/invitations/accept` | Public | Accept invitation with email+password |
| POST | `/invitations/accept-oauth` | Public | Accept invitation via OAuth (Google SSO) |

**Key flows (GitNexus):**
- `createInvitation` → `provisionDisabledIdentity` in Keycloak → disabled user created → email sent
- `acceptInvitation` → `enableInvitedIdentity` → `enableUser` + `updateUser` in Keycloak → token exchange
- `acceptInvitationOAuth` → same enable flow but via OAuth credentials

### Email — `EmailController`

Base: `/email`

| Method | Path | Auth | Description |
|---|---|---|---|
| POST | `/email/test` | JWT | Test email send (triggers notification-serice via HTTP) |

**Flow (GitNexus trace `SendEmail → Notification`):**
`sendEmail` → `EmailService.sendHtmlEmail()` (Spring Mail) + `SendNotificationService.send()` in notification-serice

---

## gRPC Server (embedded port)

Consumed by `roadmap-service`.

**Proto:** `src/main/proto/profile/profile_lookup.proto`

| RPC | Request | Response |
|---|---|---|
| `GetUserProfile` | `{user_id}` | `{user_id, display_name, email, avatar_url, found, role}` |
| `GetUserProfiles` | `{user_ids: []}` | `map<user_id, {display_name, email, avatar_url, role}>` |

Roles: `STUDENT`, `TEACHER`, `ADMIN`

---

## Request / Response Examples

### Register User

```http
POST /profile/users/register
Content-Type: application/json

{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "gender": "MALE",
  "dateOfBirth": "1995-05-15",
  "phoneNumber": "+1234567890"
}
```

```json
{
  "code": 1000,
  "result": {
    "userId": "keycloak-uuid",
    "username": "johndoe",
    "email": "john@example.com",
    "firstName": "John",
    "lastName": "Doe"
  }
}
```

### Create Invitation

```http
POST /profile/invitations
Authorization: Bearer {token}
Content-Type: application/json

{
  "email": "invitee@example.com",
  "role": "STUDENT"
}
```

---

## Key Implementation Notes

- **Keycloak integration:** `IdentityClient` (OpenFeign) — all user CRUD and token operations go through Keycloak Admin REST API
- **Token cache:** Keycloak client-credentials token is cached in Redis to avoid repeated token requests; evicted on password change
- **User sync:** `UserService.syncCurrentUserFromToken()` — on first request, syncs Keycloak token claims into local MySQL profile record
- **MySQL schema:** Managed by Flyway (`src/main/resources/db/migration/`)
- **Postman collection:** `Profile-Service-API.postman_collection.json` in service root
