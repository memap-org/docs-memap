# MeMaP — Codebase Audit Report

**Date:** 2026-05-09
**Scope:** All Java/Spring Boot services (`profile-service`, `roadmap-service`, `notification-serice`, `payment-service`, `storage-service`, `api-gateway`, `roadmap-ai-service`)
**Method:** Static inspection (file reads + targeted greps); GitNexus index used for repo overview.
**Note:** GitNexus FTS/embeddings were unavailable during this run (read-only DB warnings, 0 embeddings). Findings come from direct source review.

---

## How to read this report

Each finding has two halves, kept separate as requested:

- **Bug / Issue** — what is wrong and where.
- **Fix proposal** — concrete, minimal change to address it.

Severity legend:

| Level | Meaning |
|---|---|
| **CRITICAL** | Correctness / money / data loss — fix before next release. |
| **HIGH** | Real performance or correctness impact in production. |
| **MEDIUM** | Latent risk, scaling problem, or convention drift. |
| **LOW** | Cleanup / hardening. |

---

## Summary table

| # | Severity | Service | Title |
|---|---|---|---|
| 1 | CRITICAL | payment-service | Stripe webhook handlers are unimplemented stubs but events are marked processed |
| 2 | HIGH | roadmap-service | `ProfileSyncService.syncRoleForUser` scans the entire `RoadMap` collection per user |
| 3 | HIGH | roadmap-service | `ProfileSyncStartupConfig` runs full collection scan + per-doc saves on every boot |
| 4 | HIGH | roadmap-service | `QuizRepository` paginates and filters in JVM after `findAll()`-style fetch |
| 5 | HIGH | roadmap-service | `ProfileSyncService.fetchUserRole` swallows gRPC errors as "no role" |
| 6 | MEDIUM | notification-serice | `markAllAsRead` reads-then-saves N docs instead of one bulk update |
| 7 | MEDIUM | all Java services | 114 `LocalDateTime` usages violate the "always `Instant`" convention |
| 8 | MEDIUM | api-gateway | DEBUG logging for `gateway` and `web` packages baked into `application.yml` |
| 9 | MEDIUM | profile-service | `AsyncConfig` uses `CallerRunsPolicy` — overflow blocks HTTP request threads |
| 10 | MEDIUM | api-gateway | Swagger filter chain `permitAll()` exposes API docs in all profiles |
| 11 | MEDIUM | payment-service | Default `RABBITMQ_PASSWORD:admin` hard-coded in `application.yml` |
| 12 | LOW | notification-serice | Service directory is misspelled (`notification-serice`) |
| 13 | LOW | roadmap-service | `RoadmapServiceApplication` has `@EnableAsync` but no `TaskExecutor` bean — uses default `SimpleAsyncTaskExecutor` (unbounded threads) |
| 14 | LOW | storage-service | `InternalFileController.deleteInternal` is not behind `@PreAuthorize` — relies entirely on the `InternalApiKeyFilter` |

---

## 1. CRITICAL — Stripe webhook handlers are unimplemented but acknowledged

**File:** `payment-service/src/main/java/com/techmap/payment/application/payment/service/StripeWebhookDispatchService.java`

### Bug
All seven handlers (`handleCheckoutSessionCompleted`, `handlePaymentIntentSucceeded`, `handleSubscriptionCreated`, `handleSubscriptionUpdated`, `handleSubscriptionDeleted`, `handleInvoicePaymentSucceeded`, `handleInvoicePaymentFailed`) only log `NOT_IMPLEMENTED handler invoked: …` and return. After the switch, the dispatcher unconditionally calls `processedEventPort.save(...)` and the controller returns `200 OK` to Stripe.

Consequence: every payment, subscription, and invoice event from Stripe is silently dropped **and** recorded as "already processed", so Stripe will not retry. Credits/subscriptions will never be granted, refunded, or expired by the webhook. This is a money-handling correctness bug — the worst kind.

### Fix proposal
1. Either implement each handler (call into the appropriate `application/credit` and `application/subscription` use-cases), **or** fail loudly until they are: throw an `AppException(NOT_IMPLEMENTED)` from each stub so the controller returns a non-2xx response and Stripe will retry. **Do not** call `processedEventPort.save` for events that were not actually processed.
2. Move `processedEventPort.save(...)` into each handler's success path (after side-effects commit), so a failed handler keeps the event eligible for retry.
3. Wrap dispatch in `@Transactional` (Mongo multi-doc tx, since we save credit/subscription + processed-event) **or** apply the standard inbox/outbox pattern: insert the processed-event row first inside the same transaction as the side-effect, idempotent on `eventId` unique index.
4. Add an integration test per event type that asserts (a) the side-effect fires once, (b) the second delivery is a no-op, (c) a handler exception leaves `ProcessedStripeEvent` empty.

---

## 2. HIGH — `syncRoleForUser` does a full RoadMap scan per user

**File:** `roadmap-service/src/main/java/com/techmap/roadmap/service/impl/ProfileSyncService.java:118`

### Bug
```java
List<RoadMap> allRoadmaps = roadMapRepository.findAll();
for (RoadMap roadmap : allRoadmaps) { … filter by userId … }
```
There is already a query method `findByPermissions_UserIdAndIsDeletedFalse(userId)` (used in `QuizServiceImpl:173`). Each call to `syncRoleForUser` loads the entire roadmap collection into memory and iterates it.

### Fix proposal
Replace the `findAll()` with the targeted query:
```java
List<RoadMap> roadmaps = roadMapRepository.findByPermissions_UserIdAndIsDeletedFalse(userId);
```
Same for `syncRolesForAllRoadmaps` if a "permissions with null role" Mongo query can be written (`{ "permissions.role": null }`). Otherwise leave it but call it from a scheduled batch only — not on startup (see #3).

---

## 3. HIGH — Startup full-collection sync runs on every boot

**File:** `roadmap-service/src/main/java/com/techmap/roadmap/config/ProfileSyncStartupConfig.java`

### Bug
`@EventListener(ApplicationReadyEvent.class)` triggers `syncRolesForAllRoadmaps()` on **every** boot, every replica. The method does `findAll()` + N `roadMapRepository.save(roadmap)` calls + a gRPC fan-out to profile-service. With 10k roadmaps and rolling deploys this hammers MongoDB and profile-service every release.

There is also a partial fix already deployed (the method short-circuits when no permission has `role == null`), but the full scan still happens.

### Fix proposal
1. Remove the on-startup trigger entirely. This was clearly written as a one-shot backfill — once data is migrated, it should not run on every boot.
2. Replace it with either:
   - A Spring `@Scheduled(cron = "@daily")` job that runs in **one** replica (use ShedLock or RabbitMQ leader election), filtering at the DB layer with `{ "permissions.role": null }`; or
   - A standalone CLI command (`./mvnw spring-boot:run -Dspring-boot.run.arguments="--app.task=sync-roles"`) you run once during a release.
3. If you keep it as a backfill, narrow the query: add a Mongo aggregation that returns only roadmaps whose `permissions[].role` is null.

---

## 4. HIGH — `QuizRepository` JVM-side filter + paginate

**File:** `roadmap-service/src/main/java/com/techmap/learning/repository/QuizRepository.java`

### Bug
`findByTeacherIdWithFilters`, `findByTeacherIdAndNotDeleted`, `countByTeacherIdAndNotDeleted`, and `findAvailableByRoadmapIds` all call `findByTeacherIdAndDeletedFalse(...)` (which returns the full list) and then `.stream().filter(...).toList()` + a JVM-side `Comparator` sort + a manual `subList(...)` for the page. This means:

- The DB returns every quiz for the teacher regardless of page size or filter.
- Sorting and pagination are done in app memory.
- `count` runs the same full fetch and returns `.size()`.

For active teachers this is O(N) per request and grows with their content.

### Fix proposal
Express each query in Spring Data Mongo:
```java
Page<Quiz> findByTeacherIdAndDeletedFalseAndTitleContainingIgnoreCaseAndVisible(
    String teacherId, String title, Boolean visible, Pageable pageable);

@Query("{ 'roadmapId': { $in: ?0 }, 'deleted': false, 'visible': true, " +
       "$and: [ { $or:[{startDate:null},{startDate:{$lte:?1}}] }, " +
       "        { $or:[{endDate:null},{endDate:{$gte:?1}}] } ] }")
Page<Quiz> findAvailableByRoadmapIds(List<String> roadmapIds, Instant now, Pageable pageable);

long countByTeacherIdAndDeletedFalse(String teacherId);
```
Push sorting into `Pageable.getSort()` — Mongo will use it. Add the matching compound indexes (`{teacherId:1, deleted:1, createdAt:-1}` and `{roadmapId:1, deleted:1, visible:1, startDate:1, endDate:1}`).

---

## 5. HIGH — Silent gRPC failures masquerade as "no role"

**File:** `roadmap-service/src/main/java/com/techmap/roadmap/service/impl/ProfileSyncService.java:155`

### Bug
```java
} catch (Exception e) {
    log.error("Failed to fetch role for userId {}: {}", userId, e.getMessage());
}
return null;
```
`fetchUserRole` and `fetchUserRoles` swallow every exception and return `null` / partial map. The caller in `updateRoadmapPermissionRoles` then sees "no role for user" and leaves the permission alone — but a transient gRPC `UNAVAILABLE` is indistinguishable from "user has no role". On a subsequent successful sync nothing changes. Worse, if the catch path were ever to be moved into a write-path, it could silently downgrade the user's permission.

### Fix proposal
1. Distinguish transient from permanent errors:
   ```java
   } catch (StatusRuntimeException e) {
       if (e.getStatus().getCode() == Status.Code.UNAVAILABLE
           || e.getStatus().getCode() == Status.Code.DEADLINE_EXCEEDED) {
           throw new TransientGrpcException(e); // bubble up; retry later
       }
       log.warn("Permanent gRPC error for {}: {}", userId, e.getStatus());
       return null;
   }
   ```
2. Have `syncRolesForAllRoadmaps` abort (return -1) on a `TransientGrpcException` so the scheduled job retries on the next tick rather than committing a partial result.
3. Add a deadline on the gRPC client (e.g. `withDeadlineAfter(2, SECONDS)`) so a stuck profile-service does not hang startup or the schedule.

---

## 6. MEDIUM — `markAllAsRead` does N writes instead of one

**File:** `notification-serice/src/main/java/com/memap/notificationservice/service/MarkAsReadService.java:38`

### Bug
```java
List<Notification> unread = mongoRepository.findByReceiverAndReadFalse(userId)…
unread.forEach(Notification::markAsRead);
mongoRepository.saveAll(unread.stream().map(NotificationPersistenceMapper::toDocument).toList());
```
This loads every unread notification, mutates them, and writes each back. For a heavy user this is two collection traversals (read + saveAll → individual writes). It also races with new arrivals.

### Fix proposal
Use a single Mongo bulk update:
```java
public void markAllAsRead(String userId) {
    mongoTemplate.updateMulti(
        Query.query(Criteria.where("receiver").is(userId).and("read").is(false)),
        new Update().set("read", true).set("readAt", Instant.now()),
        NotificationDocument.class);
}
```
Add `{receiver:1, read:1}` index if not already present. Same pattern can be applied to any `mark…all…` style operations.

---

## 7. MEDIUM — `LocalDateTime` violates the project convention of `Instant`

**Convention reference:** `notification-serice/CODE_CONVENTIONS.md` and root `CLAUDE.md` ("Timestamps: Use `Instant` everywhere, never `LocalDateTime`").

### Bug
114 occurrences across 16 files including persisted entities (`Quiz`, `QuizAttempt`, `BaseModel`, `User`, `Invitation`) and DTOs. `LocalDateTime` carries no timezone and silently uses the JVM default — across containers in different TZs the same write can produce different absolute instants. It also confuses comparisons across services that already use `Instant`.

### Fix proposal
1. Convert entity fields to `Instant`. Treat this as one PR per service to keep diffs reviewable. Use `@JsonFormat(shape = STRING)` if you need ISO output.
2. Provide migration scripts where the underlying type changes (Mongo: BSON Date is fine; MySQL: `DATETIME(6)` already stores enough precision but verify timezone handling on the JDBC driver — `serverTimezone=UTC`).
3. Update DTOs and request/response mappers in lockstep.
4. Add an Archunit / Checkstyle rule to fail the build on new `LocalDateTime` imports outside of explicit "calendar / wall-clock" use cases.

---

## 8. MEDIUM — Gateway logs at DEBUG by default

**File:** `api-gateway/src/main/resources/application.yml` (last block)

### Bug
```yaml
logging:
  level:
    org.springframework.cloud.gateway: DEBUG
    org.springframework.web: DEBUG
```
This is committed as the baseline. In production it generates per-request stack traces, body buffers, and header dumps; it slows the gateway and can leak `Authorization` headers / cookies into logs.

### Fix proposal
1. Default to `INFO`. Move the DEBUG block to `application-dev.yml`.
2. Add a `application-prod.yml` with `org.springframework: WARN`, plus a small structured-logging filter that omits `Authorization` and `Cookie` headers.

---

## 9. MEDIUM — `CallerRunsPolicy` blocks Tomcat threads on async overflow

**File:** `profile-service/src/main/java/com/techmap/profileservice/config/AsyncConfig.java`

### Bug
The async pool is sized 5–10 threads with a queue of 100 and rejection policy `CallerRunsPolicy`. `EmailService.sendHtmlEmail` is `@Async` — when the queue is full (e.g. an email-blast invitation flood), the **HTTP request thread that called `sendHtmlEmail`** runs the SMTP send synchronously. This silently turns a fire-and-forget call into a blocking one, exhausting Tomcat threads and tipping the service into a slow-down.

### Fix proposal
Pick one of:
- `AbortPolicy` + a dead-letter queue / retry table for emails (fail fast, observe overflow in metrics).
- `DiscardPolicy` for non-critical mail with explicit alerting.
- Increase `queueCapacity` and add a circuit breaker around the SMTP client.
Whatever you pick, expose `ThreadPoolTaskExecutor#getActiveCount` / `getQueueSize` via Micrometer so overflow is visible.

---

## 10. MEDIUM — Swagger endpoints `permitAll()` in all profiles

**File:** `api-gateway/src/main/java/com/memap/apigateway/config/security/SecurityConfig.java`

### Bug
The `swaggerFilterChain` is order-0 and matches `/v3/api-docs/**`, `/profile/api-docs/**`, etc., with `anyRequest().permitAll()`. This exposes the full API surface (paths, parameters, error codes) to anyone who can reach the gateway.

### Fix proposal
Gate the swagger chain on a `@Profile("dev","staging")` bean, or wrap with a basic-auth filter using a single static credential set via env vars in production.

---

## 11. MEDIUM — RabbitMQ default credentials hard-coded

**File:** `payment-service/src/main/resources/application.yml:21`

### Bug
```yaml
spring.rabbitmq.password: ${RABBITMQ_PASSWORD:admin}
```
Same default appears in `docker-compose.yml`. If `RABBITMQ_PASSWORD` is unset in any environment, the service silently uses `admin/admin`.

### Fix proposal
Drop the default — `${RABBITMQ_PASSWORD}` (no default). Spring will fail fast at startup if missing, which is what you want. Keep `admin` only inside `docker-compose.override.yml` for local dev.

---

## 12. LOW — Misspelled service directory `notification-serice`

The directory `notification-serice/` (note the missing `v`) propagates into Docker image tags, mounts in `docker-compose.yml`, and imports across the repo. The published artifact is named `notification-service` correctly.

### Fix proposal
Plan a one-shot rename PR:
1. `git mv notification-serice notification-service`
2. Update `docker-compose.yml`, `pull-all.sh`, `push-all.sh`, `.gitmodules`, GitHub Actions, and any `cd notification-serice` references.
3. Re-tag the Docker image.
Do this in isolation; nothing else should be in the same PR.

---

## 13. LOW — `@EnableAsync` without explicit `TaskExecutor` in roadmap-service

**File:** `roadmap-service/src/main/java/com/techmap/roadmap/RoadmapServiceApplication.java:11`

### Bug
`@EnableAsync` is on, but no `AsyncConfigurer` / `taskExecutor` bean is defined (unlike profile-service). Spring falls back to `SimpleAsyncTaskExecutor`, which spawns a **new thread per task with no upper bound**. The startup sync (#3) plus any ad-hoc `@Async` will compound this risk.

### Fix proposal
Copy `profile-service/AsyncConfig` (with sane numbers, e.g. core 3 / max 10 / queue 50) into roadmap-service. Choose a non-`CallerRunsPolicy` strategy per #9.

---

## 14. LOW — `InternalFileController.deleteInternal` relies solely on header filter

**File:** `storage-service/src/main/java/com/memap/storage/controller/InternalFileController.java`

### Bug
The endpoint deletes any file by id and is not annotated with `@PreAuthorize`. The only protection is `InternalApiKeyFilter`. If the filter chain is reordered or the matcher changes, the endpoint becomes wide-open. Defense in depth is missing.

### Fix proposal
Add `@PreAuthorize("hasAuthority('SCOPE_INTERNAL')")` (or whatever role the internal filter populates) and an integration test that hits the endpoint without the API key and asserts 401.

---

## Recommended next steps (ordered)

1. **Today:** Patch finding #1. Either short-circuit dispatch with a 5xx, or implement the handlers — but stop ack-ing un-handled events.
2. **This sprint:** #2, #3, #4 — these are visible perf wins on every login / quiz list / startup.
3. **Next sprint:** #5, #6, #7 — correctness + convention drift.
4. **Hardening sweep:** #8–#14 batched into one "ops hygiene" PR per service.

Each item is independent — no ordering dependency. Items #1 and #5 deserve regression tests because both are silent-failure modes.

---

## What was NOT covered

- Frontend code (`memap-frontend`, `memap-admin-frontend`) — out of scope for this sweep.
- `hocuspocus-server` (Node.js) — out of scope, but worth a follow-up review of Yjs persistence and Redis pub/sub fan-out.
- Detailed thread/lock analysis of `roadmap-ai-service`'s pgvector + AMQP flow — needs runtime profiling, not static review.
- Test coverage gaps — would require running `./mvnw test jacoco:report` on each service.
