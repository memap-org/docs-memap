# Profile Service Database Schema

> **Database Type:** PostgreSQL (JPA/Hibernate)
> **ORM Framework:** Jakarta Persistence (JPA) with Hibernate

## Tables

### user

| Field | Type | Description |
|-------|------|-------------|
| id | String (UUID) | Primary key, auto-generated UUID |
| userId | String | Unique user identifier (unique, not null) |
| username | String | Username for login (unique, not null) |
| password | String | Transient field, not persisted to database |
| email | String | User email address (unique, not null) |
| firstName | String | User's first name (nullable) |
| lastName | String | User's last name (nullable) |
| gender | Enum (MALE, FEMALE, OTHER) | User's gender (nullable) |
| avatar | String | URL/path to user's avatar image (nullable) |
| status | Enum (ACTIVE, REMOVED, BANED) | User account status (nullable) |
| role | Enum (STUDENT, TEACHER, ADMIN) | User role (not null) |
| joinAt | LocalDateTime | Date/time when user joined (nullable) |

**Indexes:**
- `idx_user_userId` on `userId`
- `idx_user_email` on `email`
- `idx_user_status` on `status`
- `idx_user_email_status` on `email, status` (composite)

---

### invitation

| Field | Type | Description |
|-------|------|-------------|
| id | String (UUID) | Primary key, auto-generated UUID |
| email | String | Email address of the invited user (nullable) |
| role | Enum (STUDENT, TEACHER, ADMIN) | Role to assign upon acceptance (not null) |
| status | Enum (PENDING, ACCEPTED, REVOKED, EXPIRED) | Invitation status (nullable) |
| tokenHash | String | Hashed invitation token (nullable) |
| expiresAt | LocalDateTime | Expiration date/time of the invitation (nullable) |
| acceptedAt | LocalDateTime | Date/time when invitation was accepted (nullable) |
| revokedAt | LocalDateTime | Date/time when invitation was revoked (nullable) |
| revokedBy | String | User ID who revoked the invitation (nullable) |
| createdBy | String | User ID who created the invitation (nullable) |
| invitedUserId | String | User ID of the invited user after acceptance (nullable) |
| createdAt | LocalDateTime | Auto-set on creation (JPA auditing) |
| updatedAt | LocalDateTime | Auto-set on update (JPA auditing) |

**Indexes:**
- `idx_email` on `email`
