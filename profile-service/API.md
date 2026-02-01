# Profile Service API Documentation

## Base URL

```
http://localhost:8082/profile
```

## Authentication

All endpoints require JWT authentication (stored in HTTP-only cookies) except for registration.

---

## Endpoints

### User Registration and Profile Management

#### Register User

```http
POST /users/register
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

**Response:**

```json
{
  "code": 0,
  "message": null,
  "result": {
    "userId": "uuid-here",
    "username": "johndoe",
    "email": "john@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "gender": "MALE",
    "dateOfBirth": "1995-05-15",
    "phoneNumber": "+1234567890",
    "avatarUrl": null,
    "createdAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Get My Profile

```http
GET /users/my-profile
Authorization: Bearer {token}
```

**Roles Required**: `STUDENT`, `TEACHER`, `ADMIN`

**Response:**

```json
{
  "code": 0,
  "message": null,
  "result": {
    "userId": "uuid-here",
    "username": "johndoe",
    "email": "john@example.com",
    "firstName": "John",
    "lastName": "Doe",
    "gender": "MALE",
    "dateOfBirth": "1995-05-15",
    "phoneNumber": "+1234567890",
    "avatarUrl": "http://localhost:8085/file/avatars/avatar-uuid.jpg",
    "bio": "Learning enthusiast",
    "createdAt": "2026-01-29T10:00:00Z",
    "updatedAt": "2026-01-29T11:00:00Z"
  }
}
```

---

#### Update Profile

```http
PUT /users
Authorization: Bearer {token}
Content-Type: application/json

{
  "firstName": "John",
  "lastName": "Doe Updated",
  "phoneNumber": "+1234567890",
  "dateOfBirth": "1995-05-15",
  "bio": "Passionate learner and teacher",
  "gender": "MALE"
}
```

**Roles Required**: `STUDENT`, `TEACHER`, `ADMIN`

**Response:**

```json
{
  "code": 0,
  "message": null,
  "result": {
    "userId": "uuid-here",
    "username": "johndoe",
    "firstName": "John",
    "lastName": "Doe Updated",
    "bio": "Passionate learner and teacher",
    "updatedAt": "2026-01-29T12:00:00Z"
  }
}
```

---

#### Remove User (Soft Delete)

```http
PATCH /users/remove/{userId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 0,
  "message": "Remove success",
  "result": null
}
```

---

### Password Management

#### Change Password

```http
POST /users/change-password
Authorization: Bearer {token}
Content-Type: application/json

{
  "oldPassword": "OldPass123!",
  "newPassword": "NewPass456!"
}
```

**Roles Required**: `STUDENT`, `TEACHER`, `ADMIN`

**Response:**

```json
{
  "code": 0,
  "message": "Success",
  "result": null
}
```

---

#### Forgot Password

```http
POST /users/forgot-password
Content-Type: application/json

{
  "email": "john@example.com"
}
```

**Response:**

```json
{
  "code": 0,
  "message": "Success",
  "result": null
}
```

**Note**: This endpoint sends a password reset email to the user.

---

### Avatar Management

#### Update Avatar

```http
POST /users/update-avatar
Authorization: Bearer {token}
Content-Type: multipart/form-data

file: [binary file data]
```

**Supported file types**: JPG, JPEG, PNG, GIF
**Max file size**: 5MB

**Response:**

```json
{
  "code": 0,
  "message": "Success",
  "result": "http://localhost:8085/file/avatars/avatar-uuid.jpg"
}
```

---

### User Search and Bulk Operations

#### Search Users by Email

```http
GET /users?page=0&size=10&email=john&userId=
Authorization: Bearer {token}
```

**Query Parameters**:

- `page` (default: 0) - Page number
- `size` (default: 10) - Page size
- `email` (optional) - Email search query
- `userId` (optional) - User ID filter

**Response:**

```json
{
  "code": 0,
  "message": "User find by email: john",
  "result": {
    "currentPage": 0,
    "totalPages": 1,
    "pageSize": 10,
    "totalElements": 2,
    "data": [
      {
        "userId": "uuid-1",
        "email": "john@example.com",
        "username": "johndoe",
        "avatarUrl": "http://localhost:8085/file/avatars/avatar-uuid.jpg"
      },
      {
        "userId": "uuid-2",
        "email": "johnny@example.com",
        "username": "johnny",
        "avatarUrl": null
      }
    ]
  }
}
```

---

#### Get Users in Bulk

```http
POST /users/bulk
Authorization: Bearer {token}
Content-Type: application/json

{
  "userIds": [
    "uuid-1",
    "uuid-2",
    "uuid-3"
  ]
}
```

**Response:**

```json
{
  "code": 0,
  "message": "Get user list successfully",
  "result": [
    {
      "userId": "uuid-1",
      "email": "john@example.com",
      "username": "johndoe",
      "avatarUrl": "http://localhost:8085/file/avatars/avatar-1.jpg"
    },
    {
      "userId": "uuid-2",
      "email": "jane@example.com",
      "username": "janedoe",
      "avatarUrl": "http://localhost:8085/file/avatars/avatar-2.jpg"
    },
    {
      "userId": "uuid-3",
      "email": "bob@example.com",
      "username": "bobsmith",
      "avatarUrl": null
    }
  ]
}
```

---

## Error Responses

### 400 Bad Request

```json
{
  "code": 1001,
  "message": "Invalid request parameters",
  "result": null
}
```

### 401 Unauthorized

```json
{
  "code": 1002,
  "message": "Unauthenticated",
  "result": null
}
```

### 403 Forbidden

```json
{
  "code": 1003,
  "message": "Access denied",
  "result": null
}
```

### 404 Not Found

```json
{
  "code": 1004,
  "message": "User not found",
  "result": null
}
```

### 409 Conflict

```json
{
  "code": 1005,
  "message": "Email already exists",
  "result": null
}
```

### 500 Internal Server Error

```json
{
  "code": 9999,
  "message": "Internal server error",
  "result": null
}
```

---

## Data Models

### UserRequest (Registration)

```typescript
{
  username: string;      // Required, 3-50 characters
  email: string;         // Required, valid email format
  password: string;      // Required, min 8 characters
  firstName: string;     // Required
  lastName: string;      // Required
  gender: "MALE" | "FEMALE" | "OTHER";  // Required
  dateOfBirth?: string;  // Optional, format: YYYY-MM-DD
  phoneNumber?: string;  // Optional
}
```

### UpdateProfileRequest

```typescript
{
  firstName?: string;
  lastName?: string;
  phoneNumber?: string;
  dateOfBirth?: string;  // Format: YYYY-MM-DD
  bio?: string;          // Max 500 characters
  gender?: "MALE" | "FEMALE" | "OTHER";
}
```

### UserResponse

```typescript
{
  userId: string;
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  gender: string;
  dateOfBirth?: string;
  phoneNumber?: string;
  avatarUrl?: string;
  bio?: string;
  createdAt: string;     // ISO 8601 format
  updatedAt: string;     // ISO 8601 format
}
```

---

## Integration with Other Services

### Identity Service

- Validates JWT tokens
- Manages user credentials
- Handles OAuth2 authentication

### File Service

- Stores and retrieves avatar images
- Provides presigned URLs for avatar access

### RabbitMQ Events

- `user.registered` - Published when a new user registers
- `user.updated` - Published when a user profile is updated
- `email.sent` - Consumed for email notifications
