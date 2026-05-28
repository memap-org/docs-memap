# Notification Service Database Schema

> **Database Type:** MongoDB (Spring Data MongoDB)
> **ORM Framework:** Spring Data MongoDB with `@Document` annotations

## Collections

### notifications

User notifications. Extends `MongoAuditableDocument` (createdAt, updatedAt).

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| createdAt | Instant | Auto-set on creation (from MongoAuditableDocument) |
| updatedAt | Instant | Auto-set on update (from MongoAuditableDocument) |
| title | String | Notification title |
| content | String | Notification body content |
| sender | String | Sender identifier |
| receiver | String | Receiver user ID (indexed) |
| channel | Enum (SYSTEM, EMAIL) | Delivery channel |
| read | boolean | Whether notification has been read |
| readAt | Instant | Timestamp when notification was read |
| referenceId | String | Reference to related entity ID |
| referenceType | String | Type of the referenced entity |
| streamMessageId | String | SSE stream message ID (unique, sparse index) |

**Indexes:**
- `receiver` on `receiver`
- `streamMessageId` on `streamMessageId` (unique, sparse)

---

### notification_templates

Templates for generating notifications. Extends `MongoAuditableDocument`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| createdAt | Instant | Auto-set on creation (from MongoAuditableDocument) |
| updatedAt | Instant | Auto-set on update (from MongoAuditableDocument) |
| template_key | String | Unique template identifier key (unique index) |
| title | String | Template title |
| content | String | Template content with placeholders |
| channel | Enum (SYSTEM, EMAIL) | Default delivery channel |
| notificationType | Enum (SYSTEM, ROADMAP, QUIZ, PAYMENT, PROFILE) | Notification category |
| deleted | boolean | Soft delete flag |
| deletedAt | Instant | Soft delete timestamp |
| deletedBy | String | User who deleted the template |

**Indexes:**
- `template_key` on `template_key` (unique)

## Enums

### Channel

| Value | Description |
|-------|-------------|
| SYSTEM | In-app system notification |
| EMAIL | Email notification |

### NotificationType

| Value | Description |
|-------|-------------|
| SYSTEM | System-level notifications |
| ROADMAP | Roadmap-related notifications |
| QUIZ | Quiz-related notifications |
| PAYMENT | Payment-related notifications |
| PROFILE | Profile-related notifications |
