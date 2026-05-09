# Payment Service Architecture

## Layered Architecture (Clean/Hexagonal)

```
payment-service/
└── src/main/java/com/techmap/payment/
    ├── domain/           # Pure Java models — no Spring/Lombok/JPA
    │   ├── common/       # SoftDeletable, AuditMetadata
    │   ├── credit/       # CreditAccount, CreditTransaction, enums
    │   ├── subscription/ # Subscription, Plan, enums
    │   └── stripe/       # StripeCustomerMapping, PaymentHistory, ProcessedStripeEvent
    │
    ├── application/      # Use cases, output port interfaces, commands, responses
    │   ├── common/       # ApiResponse<T>, AppException, ErrorCode
    │   ├── credit/port/  # GetBalanceUseCase, ConsumeCreditsUseCase, ...
    │   ├── subscription/port/ # GetPlansUseCase, CreateSubscriptionUseCase, ...
    │   ├── payment/port/ # PaymentGatewayPort, StripeCustomerPersistencePort, ...
    │   ├── credit/service/    # Service implementations
    │   ├── subscription/service/ # Service implementations
    │   ├── payment/service/ # StripeWebhookDispatchService
    │   └── messaging/    # PaymentEventPublisher interface
    │
    ├── infrastructure/   # MongoDB documents, repositories, adapters, config
    │   ├── config/       # MongoConfig, RedisCacheConfig, RabbitConfig, SecurityConfig, ...
    │   ├── credit/       # Documents, repositories, adapters, mappers
    │   ├── subscription/ # Documents, repositories, adapters, PlanCatalogProperties
    │   ├── stripe/       # StripeService, persistence documents/adapters
    │   └── messaging/    # RabbitPaymentEventPublisher
    │
    └── presentation/     # REST controllers and web DTOs
        ├── credit/       # CreditController + DTOs
        ├── subscription/ # SubscriptionController + DTOs
        ├── payment/      # WebhookController, PaymentController
        └── exception/    # GlobalExceptionHandler
```

## MongoDB Collections

| Collection | Index | Note |
|------------|-------|------|
| `tbl_credit_accounts` | unique on `user_id` where `is_active=true` | Lazy-created on first request |
| `tbl_credit_transactions` | compound `(account_id, created_at DESC)`, unique on `reference_id` | Idempotency via referenceId |
| `tbl_subscriptions` | `user_id` | At most one active per user |
| `tbl_payment_history` | `user_id` | Append-only |
| `tbl_stripe_customers` | unique on `user_id` | userId → Stripe customerId |
| `tbl_processed_stripe_events` | unique on `event_id`, TTL 30d on `processed_at` | Webhook dedup |

## Caching Strategy

- `creditBalance` cache: Caffeine (5s local TTL) + Redis (60s distributed TTL)
- `@Cacheable("creditBalance", key="#userId")` on `GetBalanceService`
- `@CacheEvict` on every `consumeCredits` / `addCredits`
- Consume path always reads from primary (not cache) for correctness

## Messaging (RabbitMQ)

Exchange: `payment.events` (topic)

| Routing Key | Event |
|-------------|-------|
| `credit.granted` | Credits added to a user |
| `credit.consumed` | Credits deducted from a user |
| `subscription.created` | New paid subscription activated |
| `subscription.cancelled` | Subscription cancelled |

## Security

- All endpoints require Keycloak JWT except `/webhooks/**` and `/actuator/**`
- `/credits/consume` intended for service-to-service (caller provides userId in body)
- Stripe webhook authenticated by signature verification (`Stripe-Signature` header)
- Raw `byte[]` body preserves Stripe's hash

## Key Design Decisions

- **MongoDB transactions** for credit deduction (findAndModify with balance condition)
- **Unique index + sparse** on `reference_id` prevents double-spend
- **Webhook stub** pattern: handlers log at WARN, record idempotency, no-op otherwise
- **Synthetic FREE subscription**: users with no DB subscription get a virtual FREE response
