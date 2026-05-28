# Storage Service Database Schema

> **Database Type:** MongoDB (Spring Data MongoDB)
> **ORM Framework:** Spring Data MongoDB with `@Document` annotations

## Collections

### file_metadata

Stores metadata for uploaded files.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| name | String | Stored file name |
| originalName | String | Original uploaded file name |
| contentType | String | MIME type of the file |
| size | Long | File size in bytes |
| md5Checksum | String | MD5 checksum for integrity verification |
| storagePath | String | Path where the file is stored |
| ownerId | String | User ID who owns the file (indexed) |
| roadmapId | String | Associated roadmap ID (indexed) |
| roadmapOwnerId | String | Owner of the associated roadmap (indexed) |
| roadmapAssetType | String | Type of roadmap asset (enum reference) |
| createdAt | LocalDateTime | Auto-set on creation |
| updatedAt | LocalDateTime | Auto-set on update |

**Indexes:**
- `ownerId` on `ownerId`
- `roadmapId` on `roadmapId`
- `roadmapOwnerId` on `roadmapOwnerId`
- `idx_roadmapOwnerId_roadmapId` on `roadmapOwnerId, roadmapId` (composite)

---

### roadmap_storage

Tracks storage quotas and usage per roadmap.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| roadmapId | String | Roadmap ID (unique index) |
| ownerId | String | Roadmap owner user ID |
| usedStorage | Long | Bytes currently used, default 0 |
| maxStorage | Long | Storage quota limit in bytes |
| fileCount | Integer | Number of files, default 0 |
| createdAt | LocalDateTime | Auto-set on creation |
| updatedAt | LocalDateTime | Auto-set on update |

**Indexes:**
- `roadmapId` on `roadmapId` (unique)

## Enums

### RoadmapAssetType

| Value | Description |
|-------|-------------|
| ROADMAP_IMAGE | Roadmap cover/thumbnail image |
| ROADMAP_RESOURCE | Resource attached to roadmap |
| NODE_RESOURCE | Resource attached to a node |
| NODE_INLINE_MEDIA | Inline media embedded in node content |
