# AI Service Database Schema

> **Database Type:** MongoDB (Spring Data MongoDB)
> **ORM Framework:** Spring Data MongoDB with `@Document` annotations

## Collections

### roadmap_generation_jobs

Tracks AI-powered roadmap generation jobs.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| jobId | String | Unique job identifier (unique index) |
| userId | String | User ID who requested generation (indexed) |
| scope | String | Generation scope |
| mode | String | Generation mode |
| topic | String | Topic for roadmap generation |
| description | String | Description/context for generation |
| maxNodes | Integer | Maximum number of nodes to generate |
| roadmapCategoryId | String | Category ID for the generated roadmap |
| status | Enum | Job status (indexed) |
| progress | Integer | Generation progress percentage |
| currentStepOrder | Integer | Current pipeline step order |
| result | Map\<String, Object\> | Generation results (populated on COMPLETED) |
| errorCode | String | Error code (populated on FAILED) |
| errorMessage | String | Error message (populated on FAILED) |
| retryCount | Integer | Number of retry attempts |
| lastRetryAt | Instant | Timestamp of last retry |
| createdAt | Instant | Job creation timestamp (indexed) |
| updatedAt | Instant | Last update timestamp |
| completedAt | Instant | Job completion timestamp |
| expireAt | Instant | TTL expiration timestamp (auto-delete after expiry) |

**Indexes:**
- `jobId` on `jobId` (unique)
- `userId` on `userId`
- `status` on `status`
- `createdAt` on `createdAt`
- `user_created_idx` on `userId, createdAt` (composite)
- `user_active_job_idx` on `userId, status` (composite)
- `expireAt` on `expireAt` (TTL index, expireAfterSeconds = 0)

---

### job_steps

Individual steps within a roadmap generation job pipeline.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key |
| jobId | String | Parent job ID (indexed) |
| stepOrder | int | Order of the step in the pipeline |
| stepType | Enum | Type of step |
| status | Enum | Step status |
| input | Map\<String, Object\> | Step input data |
| output | Map\<String, Object\> | Step output data |
| acknowledged | boolean | Whether step has been acknowledged, default false |
| errorCode | String | Error code on failure |
| errorMessage | String | Error message on failure |
| startedAt | Instant | Step start timestamp |
| completedAt | Instant | Step completion timestamp |
| createdAt | Instant | Step creation timestamp |
| updatedAt | Instant | Last update timestamp |

**Indexes:**
- `jobId` on `jobId`
- `job_step_order_idx` on `jobId, stepOrder` (composite)

---

### User

Chat/user entity for the AI service. Extends `BaseEntity`.

| Field | Type | Description |
|-------|------|-------------|
| id | String (ObjectId) | Primary key (from BaseEntity) |
| createdAt | Instant | Creation timestamp (from BaseEntity) |
| updatedAt | Instant | Update timestamp (from BaseEntity) |
| externalUserId | String | External system user ID |
| username | String | Username |
| email | String | User email address |
