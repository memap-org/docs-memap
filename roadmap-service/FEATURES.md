# Roadmap Service — Core Features

**Service**: `roadmap-service`  
**Port**: `8083` | **Context path**: `/road-map` | **gRPC**: `9083`  
**Database**: MongoDB | **Cache**: Redis + Caffeine

---

## 1. Roadmap Management

### Data Model

| Field         | Type              | Description                      |
| ------------- | ----------------- | -------------------------------- |
| `id`          | String (ObjectId) | Unique roadmap identifier        |
| `name`        | String            | Roadmap title                    |
| `description` | String            | Optional description             |
| `scope`       | Enum              | `PRIVATE`, `GROUP`, `PUBLIC`     |
| `ownerId`     | String            | Keycloak user ID of creator      |
| `categoryId`  | String            | Optional category reference      |
| `thumbnailId` | String            | Optional storage-service file ID |
| `createdAt`   | DateTime          | Auto-set on creation             |
| `updatedAt`   | DateTime          | Auto-updated on change           |

### Scope Rules

| Scope     | Visibility                        | Edit                               |
| --------- | --------------------------------- | ---------------------------------- |
| `PRIVATE` | Owner only                        | Owner only                         |
| `GROUP`   | Owner + granted users             | Owner + users with EDIT permission |
| `PUBLIC`  | Anyone (unauthenticated included) | Owner only                         |

### Key Endpoints

```
POST   /road-map/api/roadmap          Create roadmap
GET    /road-map/api/roadmap/{id}     Get roadmap by ID
PUT    /road-map/api/roadmap/{id}     Update roadmap metadata
DELETE /road-map/api/roadmap/{id}     Delete roadmap (owner only)
GET    /road-map/api/roadmap/my       List roadmaps owned by current user
GET    /road-map/api/roadmap/public   List public roadmaps (paginated)
GET    /road-map/api/roadmap/shared   List roadmaps shared with current user
```

---

## 2. Node & Edge Management

Roadmaps are modeled as directed graphs. **Nodes** represent learning topics; **Edges** represent relationships/order between nodes.

### Node Fields

| Field                     | Type               | Description                            |
| ------------------------- | ------------------ | -------------------------------------- |
| `id`                      | String             | Unique node ID                         |
| `roadmapId`               | String             | Parent roadmap                         |
| `title`                   | String             | Node label/heading                     |
| `content`                 | String (rich text) | Node body (TipTap JSON or HTML)        |
| `positionX` / `positionY` | Float              | Canvas coordinates (XYFlow)            |
| `type`                    | String             | Node type (default, custom)            |
| `fileIds`                 | List\<String\>     | Attached file IDs from storage-service |

### Edge Fields

| Field          | Type   | Description         |
| -------------- | ------ | ------------------- |
| `id`           | String | Unique edge ID      |
| `roadmapId`    | String | Parent roadmap      |
| `sourceNodeId` | String | Source node         |
| `targetNodeId` | String | Target node         |
| `label`        | String | Optional edge label |

### Key Endpoints

```
POST   /road-map/api/nodes                     Create node
GET    /road-map/api/nodes/{roadmapId}         List all nodes in roadmap
PUT    /road-map/api/nodes/{nodeId}            Update node
DELETE /road-map/api/nodes/{nodeId}            Delete node

POST   /road-map/api/edges                     Create edge
GET    /road-map/api/edges/{roadmapId}         List all edges in roadmap
DELETE /road-map/api/edges/{edgeId}            Delete edge
```

---

## 3. Access Control & Permissions

### Permission Levels

| Level | Code   | Description                       |
| ----- | ------ | --------------------------------- |
| View  | `VIEW` | Can read roadmap content          |
| Edit  | `EDIT` | Can modify nodes, edges, metadata |

### Permission Operations

```
POST   /road-map/api/permissions/invite        Invite user (by email or ID)
GET    /road-map/api/permissions/{roadmapId}   List all permissions for roadmap
PUT    /road-map/api/permissions/{id}          Update permission level
DELETE /road-map/api/permissions/{id}          Revoke permission
```

**Invite flow:**

1. Owner calls invite endpoint with `userId` or `email` + `permissionLevel`
2. `roadmap-service` calls `ProfileLookupService` (gRPC) to resolve email → userId if needed
3. Permission record stored in MongoDB
4. Optional: notification sent via `notification-serice`

---

## 4. Progress Tracking

Users track their learning progress at the node level.

### Progress Statuses

| Status        | Meaning               |
| ------------- | --------------------- |
| `PENDING`     | Not started (default) |
| `IN_PROGRESS` | Currently working on  |
| `DONE`        | Completed             |
| `SKIPPED`     | Intentionally skipped |

### Progress Endpoints

```
PUT    /road-map/api/progress/{nodeId}              Update node progress status
GET    /road-map/api/progress/{roadmapId}           Get all progress for a roadmap
GET    /road-map/api/progress/{roadmapId}/summary   Get completion summary (% done)
```

Progress is stored per-user per-node — multiple users can track the same public roadmap independently.

---

## 5. Categories

Roadmaps can be organized into categories for discovery.

```
GET    /road-map/api/categories          List all categories
POST   /road-map/api/categories          Create category (ADMIN/TEACHER)
DELETE /road-map/api/categories/{id}     Delete category (ADMIN)
```

---

## 6. File Resources on Nodes

Nodes can have attached files (PDFs, images, etc.) stored in `storage-service`.

**Flow:**

1. Client uploads file via `POST /storage/...` → gets `fileId`
2. Client attaches `fileId` to node via `PUT /road-map/api/nodes/{nodeId}` with `fileIds` list
3. `roadmap-service` calls `FileLookupService` (gRPC) to validate file exists before storing reference

---

## 7. Real-Time Collaboration

Collaborative editing is handled by the **`hocuspocus-server`** (Node.js), not by `roadmap-service` directly.

**Architecture:**

```
Frontend ←→ hocuspocus-server:1234 (WebSocket / Yjs CRDT)
                  ↕
            MongoDB (document state persistence)
            Redis   (pub/sub for multi-instance sync)
```

- Each roadmap has a Yjs document identified by `roadmapId`
- All connected clients receive real-time diffs
- Changes are persisted to MongoDB on server
- `roadmap-service` REST API is the source of truth for roadmap metadata; Hocuspocus handles the live editing layer for node content

---

## 8. gRPC Lookup Service

`roadmap-service` exposes gRPC on port `9083` for other services:

```protobuf
service RoadmapLookupService {
  rpc GetRoadmapName(GetRoadmapNameRequest) returns (RoadmapNameResponse);
  rpc GetRoadmapNames(GetRoadmapNamesRequest) returns (RoadmapNamesResponse);
  rpc GetRoadmapIdsByUser(GetRoadmapIdsByUserRequest) returns (GetRoadmapIdsByUserResponse);
  rpc ValidateRoadmapStorageContext(ValidateRoadmapStorageContextRequest) returns (ValidateRoadmapStorageContextResponse);
}
```

Used by:

- `learning-serice` — fetches roadmap names to enrich quiz data
- `roadmap-ai-service` — validates storage context before AI operations

---

## 9. Caching Strategy

| Layer            | Technology | TTL / Scope                  |
| ---------------- | ---------- | ---------------------------- |
| L1 (in-process)  | Caffeine   | Short TTL, local to instance |
| L2 (distributed) | Redis      | Shared across instances      |

Cache keys typically include `roadmapId` + `userId` for per-user data, or just `roadmapId` for public metadata.

Cache is evicted on write operations (cache-aside pattern with `@CacheEvict`).

---

## 10. Swagger Access

`http://localhost:8083/road-map/swagger-ui.html`
