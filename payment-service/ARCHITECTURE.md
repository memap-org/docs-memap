# Payment Service Architecture

## Layered Architecture

```
payment-service/
└── src/main/java/com/techmap/payment/
    ├── model/              # MongoDB documents + enums
    │   ├── credit/         # CreditAccountDocument, CreditTransactionDocument, FeatureType, TransactionType
    │   ├── plan/           # PlanDocument
    │   ├── subscription/   # SubscriptionDocument, SubscriptionStatus
    │   └── stripe/         # PaymentHistoryDocument, ProcessedStripeEventDocument, StripeCustomerDocument
    │
    ├── repository/         # Spring Data MongoRepository interfaces
    │   ├── credit/         # CreditAccountRepository, CreditTransactionRepository
    │   ├── plan/           # PlanRepository
    │   ├── subscription/   # SubscriptionRepository
    │   └── stripe/         # PaymentHistoryRepository, ProcessedStripeEventRepository, StripeCustomerRepository
    │
    ├── service/            # Business logic (injects repos directly)
    │   ├── credit/         # CreditService (balance, add, consume, history)
    │   ├── plan/           # PlanService (getAll, findByPlanCode, findByStripePriceId)
    │   ├── subscription/   # SubscriptionService (get, create, change, cancel)
    │   └── stripe/         # StripeService (customer, payment history, webhook events)
    │
    ├── controller/         # REST controllers + request/response DTOs
    │   ├── credit/         # CreditController + DTOs
    │   ├── plan/           # PlanController
    │   ├── subscription/   # SubscriptionController + DTOs
    │   └── payment/        # PaymentController, WebhookController
    │
    ├── config/             # Configuration classes
    │                       # MongoConfig, RedisCacheConfig, RabbitConfig, SecurityConfig, StripeConfig, ...
    │
    ├── common/             # Shared utilities
    │                       # ApiResponse<T>, AppException, ErrorCode
    │
    ├── exception/          # Global error handling
    │                       # GlobalExceptionHandler
    │
    └── messaging/          # Event publishing
                            # PaymentEventPublisher interface, RabbitPaymentEventPublisher
```

## MongoDB Collections

| Collection | Index | Note |
|------------|-------|------|
| `tbl_plans` | unique on `plan_code`, unique on `stripe_price_id` | Plan catalog |
| `tbl_credit_accounts` | unique on `user_id` where `is_active=true` | Lazy-created on first request |
| `tbl_credit_transactions` | compound `(account_id, created_at DESC)`, unique on `reference_id` | Idempotency via referenceId |
| `tbl_subscriptions` | `user_id`, partial unique: one active/trialing per user | At most one active per user |
| `tbl_payment_history` | `user_id` | Append-only |
| `tbl_stripe_customers` | unique on `user_id` | userId → Stripe customerId |
| `tbl_processed_stripe_events` | unique on `event_id`, TTL 30d on `processed_at` | Webhook dedup |

## Caching Strategy

- `creditBalance` cache: Caffeine (5s local TTL) + Redis (60s distributed TTL)
- `@Cacheable("creditBalance", key="#userId")` on `CreditService.getBalance`
- `@CacheEvict` on every `consume` / `addCredits`
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

## Key Design Decisions

- **Layered architecture** — model → repository → service → controller (no port/adapter abstractions)
- **MongoDB documents** serve as both persistence and domain models (no separate domain entities)
- **Services inject repositories directly** — no port interfaces between layers
- **MongoDB transactions** for credit deduction (findAndModify with balance condition)
- **Unique index + sparse** on `reference_id` prevents double-spend
- **Webhook stub** pattern: handlers log at WARN, record idempotency, no-op otherwise
- **Synthetic FREE subscription**: users with no DB subscription get a virtual FREE response
