# Payment Service Implementation

## Stack

Spring Boot 3.x + MongoDB + Redis + RabbitMQ + Stripe Java SDK + Keycloak.

## Credit Account Lifecycle

Accounts are created lazily on first credit request:

```java
// GetBalanceService
@Cacheable(value = "creditBalance", key = "#userId")
public BalanceResponse getBalance(String userId) {
    CreditAccount account = creditAccountPersistencePort.findOrCreateByUserId(userId);
    return new BalanceResponse(account.getBalance(), account.getLifetimeEarned(), account.getLifetimeSpent());
}
```

## Atomic Credit Consumption

Uses `findAndModify` with a balance condition to prevent overdraft:

```java
// CreditAccountPersistenceAdapter
Query query = Query.query(
    Criteria.where("user_id").is(userId)
            .and("is_active").is(true)
            .and("balance").gte(amount));
Update update = new Update()
    .inc("balance", -amount)
    .inc("lifetime_spent", amount)
    .set("updated_at", Instant.now());
CreditAccountDocument updated = mongoTemplate.findAndModify(
    query, update, FindAndModifyOptions.options().returnNew(true),
    CreditAccountDocument.class);
// null → insufficient balance → throw INSUFFICIENT_CREDITS
```

## Idempotency

Every consume/add operation checks `referenceId` before processing:

```java
Optional<CreditTransaction> existing = transactionPort.findByReferenceId(command.referenceId());
if (existing.isPresent()) {
    // Return previous result — no duplicate deduction
    return new ConsumeResponse(balance, existing.get().getId());
}
```

A unique partial index on `tbl_credit_transactions.reference_id` (where `is_active=true`) enforces this at the DB level.

## Stripe Webhook Flow

```
POST /webhooks/stripe (raw byte[])
  → StripeService.verifyWebhook(payload, sigHeader)
      → Webhook.constructEvent(...)  [throws on bad sig → 400]
  → StripeWebhookDispatchService.dispatch(event)
      → check ProcessedStripeEvent by eventId (idempotency)
      → route to handler by event.getType()
      → record ProcessedStripeEvent (TTL 30d)
  → 200 {"received": true}
```

## Subscription State Machine

```
[no record] → FREE (synthetic, not persisted)
FREE → PRO/ENTERPRISE → (webhook: subscription.created) → persisted ACTIVE
ACTIVE → (cancel) → ACTIVE with cancelAtPeriodEnd=true
ACTIVE + cancelAtPeriodEnd → (webhook: subscription.deleted) → CANCELLED
```

Note: Create/Cancel/Change endpoints currently return `FEATURE_PENDING (7099)` per design D11. Full Stripe flows ship in `wire-stripe-webhooks`.

## Plan Catalog

Plans are read from `application.yml` — no code change needed to update pricing:

```yaml
payment:
  plans:
    pro:
      monthly-credits: 500
      price-usd: 9.99
      stripe-price-id: ${STRIPE_PRO_PRICE_ID:price_pro_REPLACE_ME}
```

## Error Handling

All errors flow through `GlobalExceptionHandler` → `ApiResponse<Void>` with 7xxx codes:

```java
@ExceptionHandler(AppException.class)
public ResponseEntity<ApiResponse<Void>> handleAppException(AppException ex) {
    ErrorCode code = ex.getErrorCode();
    return ResponseEntity.status(code.getStatusCode())
            .body(ApiResponse.error(code.getCode(), code.getMessage()));
}
```
