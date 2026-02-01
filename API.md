# API Overview

## Service-Specific Documentation

For detailed API documentation of each service, please refer to their respective documentation folders:

- **[Identity Service API](./identity-service/API.md)** - Authentication, user management
- **[Profile Service API](./profile-service/API.md)** - User profiles, avatars, password management
- **[Roadmap Service API](./roadmap-service/API.md)** - Roadmap CRUD, nodes, edges, progress tracking
- **[Roadmap AI Service API](./roadmap-ai-service/API.md)** - AI generation, summaries, chat
- **[File Service API](./file-service/API.md)** - File upload/download, presigned URLs
- **[Learning Service API](./learning-service/API.md)** - Quiz management
- **[Payment Service API](./payment-service/API.md)** - Credits, subscriptions, Stripe payments
- **[Frontend Integration](./memap-frontend/API.md)** - Frontend API integration guide

## Base URLs

| Service            | Base URL              | Context Path | Documentation                              |
| ------------------ | --------------------- | ------------ | ------------------------------------------ |
| Identity Service   | http://localhost:8081 | /identity    | [📖 API Docs](./identity-service/API.md)   |
| Profile Service    | http://localhost:8082 | /profile     | [📖 API Docs](./profile-service/API.md)    |
| Roadmap Service    | http://localhost:8083 | /roadmap     | [📖 API Docs](./roadmap-service/API.md)    |
| Roadmap AI Service | http://localhost:8084 | /ai          | [📖 API Docs](./roadmap-ai-service/API.md) |
| File Service       | http://localhost:8085 | /file        | [📖 API Docs](./file-service/API.md)       |
| Learning Service   | http://localhost:8086 | /learning    | [📖 API Docs](./learning-service/API.md)   |
| Payment Service    | http://localhost:8087 | /payment     | [📖 API Docs](./payment-service/API.md)    |
| Frontend           | http://localhost:5173 | /            | [📖 Docs](./memap-frontend/)               |

## Authentication

### JWT Token Authentication

Most endpoints require authentication via JWT tokens. Tokens are stored in HTTP-only cookies.

**Login to get tokens:**

```http
POST /identity/auth/login
Content-Type: application/json

{
  "username": "user@example.com",
  "password": "password123"
}
```

**Response cookies:**

- `access_token` - JWT access token (10 min expiry)
- `refresh_token` - Refresh token (7 days expiry)

### Authorization Roles

| Role      | Description     |
| --------- | --------------- |
| `STUDENT` | Regular user    |
| `TEACHER` | Content creator |
| `ADMIN`   | Administrator   |

---

## API Quick Reference

Below is a quick reference of key endpoints. For complete documentation with examples, see each service's API documentation.

### Identity Service - [Full API Docs](./identity-service/API.md)

### POST /identity/auth/login

Login with credentials.

**Request:**

```json
{
  "username": "user@example.com",
  "password": "password123"
}
```

### POST /identity/auth/logout

Logout and invalidate tokens.

### POST /identity/auth/refresh

Refresh access token using refresh token.

### POST /identity/auth/introspect

Validate a token.

---

## Profile Service - [Full API Docs](./profile-service/API.md)

### POST /profile/users/register

Register a new user.

**Request:**

```json
{
  "username": "johndoe",
  "email": "john@example.com",
  "password": "SecurePass123!",
  "firstName": "John",
  "lastName": "Doe",
  "gender": "MALE"
}
```

**Response:**

```json
{
  "code": 0,
  "message": null,
  "result": {
    "id": "uuid",
    "userId": "user_123",
    "username": "johndoe",
    "email": "john@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "role": "STUDENT"
  }
}
```

### GET /profile/users/my-profile

Get current user's profile. **Requires authentication.**

### PUT /profile/users

Update user profile. **Requires authentication.**

**Request:**

```json
{
  "firstName": "John",
  "lastName": "Smith",
  "gender": "MALE"
}
```

### POST /profile/users/change-password

Change password. **Requires authentication.**

**Request:**

```json
{
  "oldPassword": "currentPassword",
  "newPassword": "newSecurePassword"
}
```

### POST /profile/users/forgot-password

Initiate password reset.

**Request:**

```json
{
  "email": "user@example.com"
}
```

### POST /profile/users/update-avatar

Upload profile picture. **Requires authentication.**

**Request:** `multipart/form-data` with `file` field.

### GET /profile/users

Search users with pagination.

**Query Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | int | 0 | Page number |
| size | int | 10 | Page size |
| email | string | "" | Email search |
| userId | string | "" | User ID filter |

### POST /profile/users/bulk

Get multiple users by IDs.

**Request:**

```json
{
  "userIds": ["user_1", "user_2", "user_3"]
}
```

---

## Roadmap Service - [Full API Docs](./roadmap-service/API.md)

### POST /roadmap/roadmaps

Create a new roadmap. **Requires authentication.**

**Request:**

```json
{
  "title": "Learn Java",
  "description": "Complete Java learning path",
  "categoryId": "category_uuid",
  "isPublic": true
}
```

### GET /roadmap/roadmaps/{id}

Get roadmap by ID.

### GET /roadmap/roadmaps

List roadmaps with filters.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| page | int | Page number |
| size | int | Page size |
| category | string | Category filter |
| search | string | Search term |
| public | boolean | Public only |

### PUT /roadmap/roadmaps/{id}

Update roadmap. **Requires authentication + ownership.**

### DELETE /roadmap/roadmaps/{id}

Delete roadmap. **Requires authentication + ownership.**

### POST /roadmap/roadmaps/{id}/nodes

Add node to roadmap.

**Request:**

```json
{
  "title": "Introduction to Java",
  "description": "Getting started with Java",
  "position": { "x": 100, "y": 200 },
  "parentId": null
}
```

### PUT /roadmap/roadmaps/{roadmapId}/nodes/{nodeId}

Update node.

### DELETE /roadmap/roadmaps/{roadmapId}/nodes/{nodeId}

Delete node.

### POST /roadmap/roadmaps/{id}/permissions

Add user permission.

**Request:**

```json
{
  "userId": "user_123",
  "permission": "EDITOR"
}
```

---

## Roadmap AI Service API

### POST /ai/generate

Generate roadmap using AI.

**Request:**

```json
{
  "topic": "Machine Learning",
  "level": "beginner",
  "duration": "3 months"
}
```

**Response:**

```json
{
  "title": "Machine Learning Roadmap",
  "description": "...",
  "nodes": [
    {
      "title": "Python Basics",
      "description": "...",
      "resources": [...]
    }
  ]
}
```

### POST /ai/chat

Chat with AI about roadmap.

**Request:**

```json
{
  "roadmapId": "roadmap_uuid",
  "message": "What should I learn first?"
}
```

### POST /ai/node/summary

Get AI summary for node.

**Request:**

```json
{
  "nodeId": "node_uuid",
  "content": "Node content to summarize"
}
```

### POST /ai/node/questions

Generate quiz questions for node.

---

## File Service - [Full API Docs](./file-service/API.md)

### POST /file/upload

Upload a file.

**Request:** `multipart/form-data`
| Field | Type | Description |
|-------|------|-------------|
| file | File | File to upload |
| folder | string | Target folder (optional) |

**Response:**

```json
{
  "id": "file_uuid",
  "filename": "document.pdf",
  "url": "https://storage.example.com/files/document.pdf",
  "size": 1024000,
  "contentType": "application/pdf"
}
```

### GET /file/download/{id}

Download file by ID.

### GET /file/{id}

Get file metadata.

### DELETE /file/{id}

Delete file. **Requires authentication.**

---

## Payment Service - [Full API Docs](./payment-service/API.md)

### GET /payment/credits/balance

Get current user's credit balance and account info. **Requires authentication.**

**Response:**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "accountId": "uuid",
    "balance": 450,
    "lifetimeEarned": 1000,
    "lifetimeSpent": 550
  }
}
```

### POST /payment/credits/check

Check if user has enough credits for a feature. **Requires authentication.**

**Request:**

```json
{
  "featureType": "AI_ROADMAP_GENERATION",
  "quantity": 1
}
```

### POST /payment/credits/consume

Consume credits for feature usage. **Internal API - service-to-service.**

**Request:**

```json
{
  "userId": "user_123",
  "featureType": "AI_ROADMAP_GENERATION",
  "quantity": 1,
  "description": "Generated roadmap"
}
```

### GET /payment/packages

Get available credit packages for purchase.

### POST /payment/packages/purchase

Create Stripe checkout session for credit purchase. **Requires authentication.**

**Request:**

```json
{
  "packageId": "pkg_basic",
  "successUrl": "https://example.com/success",
  "cancelUrl": "https://example.com/cancel"
}
```

### GET /payment/subscriptions/plans

Get available subscription plans.

### GET /payment/subscriptions/current

Get user's current subscription. **Requires authentication.**

### POST /payment/subscriptions/subscribe

Subscribe to a plan via Stripe checkout. **Requires authentication.**

### POST /payment/subscriptions/cancel

Cancel subscription at period end. **Requires authentication.**

### GET /payment/history

Get payment history. **Requires authentication.**

### POST /payment/webhooks/stripe

Stripe webhook endpoint for payment events.

---

## Standard Response Format

### Success Response

```json
{
  "code": 0,
  "message": "Success",
  "result": { ... }
}
```

### Error Response

```json
{
  "code": 2006,
  "message": "User not found",
  "result": null
}
```

### Paginated Response

```json
{
  "code": 0,
  "message": null,
  "result": {
    "currentPage": 0,
    "totalPages": 10,
    "pageSize": 20,
    "totalElements": 195,
    "data": [ ... ]
  }
}
```

---

## Error Codes

| Code Range | Service  | Description             |
| ---------- | -------- | ----------------------- |
| 1xxx       | Roadmap  | Roadmap-related errors  |
| 2xxx       | Profile  | User/profile errors     |
| 3xxx       | Learning | Learning-related errors |
| 7xxx       | Payment  | Payment/credit errors   |
| 9999       | All      | Uncategorized error     |

See [ERROR_CODES.md](ERROR_CODES.md) for complete reference.

---

## Interactive Documentation

Access Swagger UI for each service:

- Identity: http://localhost:8081/identity/swagger-ui.html
- Profile: http://localhost:8082/profile/swagger-ui.html
- Roadmap: http://localhost:8083/roadmap/swagger-ui.html
- AI: http://localhost:8084/ai/swagger-ui.html
- File: http://localhost:8085/file/swagger-ui.html
- Payment: http://localhost:8087/payment/api-docs
