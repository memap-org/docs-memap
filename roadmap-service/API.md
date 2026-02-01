# Roadmap Service API Documentation

## Base URL

```
http://localhost:8083/road-map
```

## Authentication

All endpoints require authentication via JWT tokens issued by Keycloak. The token must be included in the Authorization header:

```
Authorization: Bearer {token}
```

---

## Endpoints

### Roadmap Management

#### Create Roadmap

```http
POST /api/roadmap
Content-Type: application/json
Authorization: Bearer {token}

{
  "title": "Java Backend Developer Roadmap",
  "description": "Complete learning path for becoming a Java backend developer",
  "scope": "PRIVATE",
  "categoryId": "cat-123",
  "nodes": [
    {
      "nodeId": "node-1",
      "label": "Java Basics",
      "type": "default",
      "position": { "x": 100, "y": 100 },
      "data": {
        "description": "Learn Java fundamentals",
        "resources": []
      }
    }
  ],
  "edges": [
    {
      "edgeId": "edge-1",
      "source": "node-1",
      "target": "node-2",
      "type": "smoothstep"
    }
  ]
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Success",
  "result": {
    "roadmapId": "roadmap-123",
    "title": "Java Backend Developer Roadmap",
    "createdAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Get Roadmap by ID

```http
GET /api/roadmap/{roadmapId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "roadmapId": "roadmap-123",
    "title": "Java Backend Developer Roadmap",
    "description": "Complete learning path",
    "scope": "PRIVATE",
    "ownerId": "user-123",
    "categoryId": "cat-123",
    "imageUrl": "https://api.memap.id.vn/files/roadmap-123.jpg",
    "nodes": [...],
    "edges": [...],
    "createdAt": "2026-01-29T10:00:00Z",
    "updatedAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Get My Roadmaps

```http
GET /api/roadmap/my-roadmap?page=1&size=10&scope=ALL&search=java
Authorization: Bearer {token}
```

**Query Parameters:**

- `page` (optional, default: 1): Page number
- `size` (optional, default: 10): Page size
- `scope` (optional, default: ALL): Filter by scope (PRIVATE, GROUP, PUBLIC, ALL)
- `search` (optional): Search by title

**Response:**

```json
{
  "code": 1000,
  "result": {
    "currentPage": 1,
    "totalPages": 5,
    "pageSize": 10,
    "totalElements": 47,
    "data": [
      {
        "roadmapId": "roadmap-123",
        "title": "Java Backend Developer",
        "description": "Complete learning path",
        "scope": "PRIVATE",
        "imageUrl": "https://...",
        "createdAt": "2026-01-29T10:00:00Z"
      }
    ]
  }
}
```

---

#### Get Accessible Roadmaps

Get all roadmaps that the user owns or has been granted access to.

```http
GET /api/roadmap/accessible?page=1&size=10&search=java
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "currentPage": 1,
    "totalPages": 3,
    "pageSize": 10,
    "totalElements": 25,
    "data": [...]
  }
}
```

---

#### Update Roadmap Information

```http
PATCH /api/roadmap/info/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "title": "Updated Title",
  "description": "Updated description",
  "categoryId": "cat-456"
}
```

**Response:**

```json
{
  "code": 1000,
  "result": true
}
```

---

#### Delete Roadmap

Soft delete a roadmap (only owner can delete).

```http
DELETE /api/roadmap/{roadmapId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Roadmap have been deleted",
  "result": true
}
```

---

### Node Management

#### Update Node

```http
PATCH /api/roadmap/node/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "nodeId": "node-1",
  "label": "Advanced Java",
  "type": "default",
  "position": { "x": 200, "y": 150 },
  "data": {
    "description": "Learn advanced Java concepts",
    "resources": [
      {
        "title": "Java Documentation",
        "url": "https://docs.oracle.com/javase"
      }
    ]
  }
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Node node-1 have been updated",
  "result": true
}
```

---

#### Update Node Status

```http
PATCH /api/roadmap/node/status/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "nodeId": "node-1",
  "status": "DONE"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Node node-1 have been updated",
  "result": true
}
```

---

### Edge Management

#### Update Edge

```http
PATCH /api/roadmap/edge/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "edgeId": "edge-1",
  "source": "node-1",
  "target": "node-2",
  "type": "smoothstep",
  "label": "Next Step"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Edge edge-1 have been updated",
  "result": true
}
```

---

### Progress Tracking

#### Get Roadmap Progress

Get tracking progress for the current user.

```http
GET /api/roadmap-tracking-progress/{roadmapId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "roadmapId": "roadmap-123",
    "userId": "user-123",
    "totalNodes": 10,
    "completedNodes": 5,
    "inProgressNodes": 2,
    "pendingNodes": 3,
    "progressPercentage": 50.0,
    "nodeTracking": [
      {
        "nodeId": "node-1",
        "status": "DONE",
        "updatedAt": "2026-01-29T10:00:00Z"
      },
      {
        "nodeId": "node-2",
        "status": "IN_PROGRESS",
        "updatedAt": "2026-01-29T11:00:00Z"
      }
    ]
  }
}
```

---

#### Get Node Status

```http
GET /api/roadmap-tracking-progress/{roadmapId}/{nodeId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "nodeId": "node-1",
    "status": "DONE",
    "updatedAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Update Node Tracking

```http
PATCH /api/roadmap-tracking-progress/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "nodeId": "node-1",
  "status": "DONE"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Tracking status updated",
  "result": true
}
```

---

#### Reset Progress

Set all nodes to PENDING status.

```http
POST /api/roadmap-tracking-progress/reset/{roadmapId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Progress has been reset",
  "result": true
}
```

---

#### Mark All Done

Set all nodes to DONE status.

```http
POST /api/roadmap-tracking-progress/done/{roadmapId}
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "message": "All nodes marked as DONE",
  "result": true
}
```

---

### Categories

#### Get All Categories

```http
GET /api/roadmap-category
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Get all roadmap categories successfully",
  "result": [
    {
      "categoryId": "cat-1",
      "name": "Programming",
      "description": "Programming languages and frameworks",
      "icon": "code"
    },
    {
      "categoryId": "cat-2",
      "name": "Data Science",
      "description": "Data analysis and machine learning",
      "icon": "chart"
    }
  ]
}
```

---

#### Create Category

```http
POST /api/roadmap-category
Content-Type: application/json
Authorization: Bearer {token}

{
  "name": "DevOps",
  "description": "DevOps practices and tools",
  "icon": "server"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Create roadmap category",
  "result": true
}
```

---

### Access Permissions

#### Get Roadmap Permissions

Get list of users who have access to the roadmap.

```http
GET /api/roadmap/{roadmapId}/permissions
Authorization: Bearer {token}
```

**Response:**

```json
{
  "code": 1000,
  "result": [
    {
      "userId": "user-123",
      "permission": "OWNER",
      "canEdit": true,
      "canView": true
    },
    {
      "userId": "user-456",
      "permission": "EDITOR",
      "canEdit": true,
      "canView": true
    },
    {
      "userId": "user-789",
      "permission": "VIEWER",
      "canEdit": false,
      "canView": true
    }
  ]
}
```

---

#### Update Roadmap Scope

```http
PATCH /api/roadmap/scope/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "scope": "GROUP"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Update status of roadmap successfully",
  "result": true
}
```

---

#### Add User Permission

```http
POST /api/roadmap/scope/user/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "userId": "user-456",
  "permission": "EDITOR",
  "canEdit": true,
  "canView": true
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Add user permission success",
  "result": {
    "roadmapId": "roadmap-123",
    "userId": "user-456",
    "permission": "EDITOR",
    "addedAt": "2026-01-29T10:00:00Z"
  }
}
```

---

#### Remove User Permission

```http
DELETE /api/roadmap/scope/user/{roadmapId}
Content-Type: application/json
Authorization: Bearer {token}

{
  "userId": "user-456"
}
```

**Response:**

```json
{
  "code": 1000,
  "message": "Remove user permission success",
  "result": {
    "roadmapId": "roadmap-123",
    "userId": "user-456",
    "removed": true
  }
}
```

---

### File Management

#### Upload Roadmap Image

```http
PATCH /api/roadmap/image/{roadmapId}
Content-Type: multipart/form-data
Authorization: Bearer {token}

file: (binary)
```

**Response:**

```json
{
  "code": 1000,
  "message": "Update roadmap image successfully",
  "result": {
    "fileName": "roadmap-123.jpg",
    "fileUrl": "https://api.memap.id.vn/files/roadmap-123.jpg",
    "contentType": "image/jpeg",
    "size": 102400
  }
}
```

---

#### Upload Media File

```http
POST /api/roadmap/media/upload/{roadmapId}
Content-Type: multipart/form-data
Authorization: Bearer {token}

file: (binary)
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "fileName": "document.pdf",
    "fileUrl": "https://api.memap.id.vn/files/roadmap-123/document.pdf",
    "contentType": "application/pdf",
    "size": 512000
  }
}
```

---

#### Download Media File

```http
GET /api/roadmap/media/download/{roadmapId}/{fileName}
Authorization: Bearer {token}
```

**Response:**

- Binary file content with appropriate Content-Type header

---

#### Upload File Resource

```http
POST /api/roadmap/resource/upload/file/{roadmapId}
Content-Type: multipart/form-data
Authorization: Bearer {token}

file: (binary)
```

**Response:**

```json
{
  "code": 1000,
  "result": {
    "fileName": "resource.zip",
    "fileUrl": "https://...",
    "contentType": "application/zip",
    "size": 1024000
  }
}
```

---

## Status Codes

- `1000`: Success
- `1001`: Validation error
- `1002`: Resource not found
- `1003`: Unauthorized
- `1004`: Forbidden
- `1005`: Internal server error

## Error Response Format

```json
{
  "code": 1003,
  "message": "You don't have permission to access this roadmap"
}
```

## Node Status Values

- `PENDING`: Not started
- `IN_PROGRESS`: Currently working on
- `DONE`: Completed
- `SKIPPED`: Skipped this node

## Roadmap Scope Values

- `PRIVATE`: Only owner can view and edit
- `GROUP`: Owner + granted users can view/edit
- `PUBLIC`: Anyone can view, only owner can edit

## Permission Types

- `OWNER`: Full control (view, edit, delete, manage permissions)
- `EDITOR`: Can view and edit roadmap
- `VIEWER`: Can only view roadmap
