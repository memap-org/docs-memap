# Identity Service API Documentation

## Base URL

```
http://localhost:8081/identity
```

## Authentication

Most endpoints require authentication via JWT tokens stored in HTTP-only cookies.

---

## Endpoints

### User Management

#### Create User

```http
POST /users
Content-Type: application/json

{
  "username": "johndoe",
  "password": "SecurePass123!",
  "email": "john@example.com",
  "roles": ["STUDENT"]
}
```

**Response:**

```json
{
  "userId": "uuid-here",
  "username": "johndoe",
  "email": "john@example.com",
  "roles": ["STUDENT"],
  "createdAt": "2026-01-29T10:00:00Z"
}
```

---

#### Get All Users

```http
GET /users
Authorization: Bearer {token}
```

**Response:**

```json
[
  {
    "userId": "uuid-1",
    "username": "user1",
    "email": "user1@example.com",
    "roles": ["STUDENT"]
  },
  {
    "userId": "uuid-2",
    "username": "user2",
    "email": "user2@example.com",
    "roles": ["TEACHER"]
  }
]
```

---

#### Get User by ID

```http
GET /users/{userId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "userId": "uuid-here",
  "username": "johndoe",
  "email": "john@example.com",
  "roles": ["STUDENT"],
  "createdAt": "2026-01-29T10:00:00Z",
  "updatedAt": "2026-01-29T10:00:00Z"
}
```

---

#### Update User

```http
PUT /users/{userId}
Authorization: Bearer {token}
Content-Type: application/json

{
  "username": "johndoe_updated",
  "email": "john.new@example.com",
  "roles": ["TEACHER"]
}
```

**Response:**

```json
{
  "userId": "uuid-here",
  "username": "johndoe_updated",
  "email": "john.new@example.com",
  "roles": ["TEACHER"],
  "updatedAt": "2026-01-29T11:00:00Z"
}
```

---

#### Delete User

```http
DELETE /users/{userId}
Authorization: Bearer {token}
```

**Response:**

```
Delete success
```

---

## Authentication Endpoints

### Login

```http
POST /auth/login
Content-Type: application/json

{
  "username": "johndoe",
  "password": "SecurePass123!"
}
```

**Response:**

- Sets `access_token` cookie (HTTP-only)
- Sets `refresh_token` cookie (HTTP-only)

```json
{
  "code": 0,
  "message": "Login successful",
  "result": {
    "userId": "uuid-here",
    "username": "johndoe",
    "roles": ["STUDENT"]
  }
}
```

---

### Logout

```http
POST /auth/logout
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 0,
  "message": "Logout successful"
}
```

---

### Refresh Token

```http
POST /auth/refresh
Cookie: refresh_token={token}
```

**Response:**

- Sets new `access_token` cookie

```json
{
  "code": 0,
  "message": "Token refreshed successfully"
}
```

---

### Token Introspection

```http
POST /auth/introspect
Content-Type: application/json

{
  "token": "jwt-token-here"
}
```

**Response:**

```json
{
  "valid": true,
  "userId": "uuid-here",
  "username": "johndoe",
  "roles": ["STUDENT"],
  "expiresAt": "2026-01-29T10:10:00Z"
}
```

---

## Error Responses

### 400 Bad Request

```json
{
  "code": 400,
  "message": "Invalid request parameters",
  "errors": [
    {
      "field": "username",
      "message": "Username is required"
    }
  ]
}
```

### 401 Unauthorized

```json
{
  "code": 401,
  "message": "Invalid credentials"
}
```

### 403 Forbidden

```json
{
  "code": 403,
  "message": "Access denied"
}
```

### 404 Not Found

```json
{
  "code": 404,
  "message": "User not found"
}
```

### 500 Internal Server Error

```json
{
  "code": 500,
  "message": "Internal server error"
}
```

---

## Token Information

| Token Type    | Storage          | Expiry     | Usage                      |
| ------------- | ---------------- | ---------- | -------------------------- |
| Access Token  | HTTP-only Cookie | 10 minutes | API authentication         |
| Refresh Token | HTTP-only Cookie | 7 days     | Generate new access tokens |

---

## Security Headers

All responses include security headers:

- `X-Content-Type-Options: nosniff`
- `X-Frame-Options: DENY`
- `X-XSS-Protection: 1; mode=block`
- `Strict-Transport-Security: max-age=31536000; includeSubDomains`
