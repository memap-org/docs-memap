# Payment Service

## Overview

The Payment Service is a **Spring Boot 3.x + MongoDB** microservice that handles monetization for the MeMap platform — credit accounts, subscription plans, and Stripe payment integration.

## Key Features

- **Credit Management**: Per-user credit balance and transaction ledger (purchase, usage, refund, bonus)
- **Stripe Integration**: Checkout sessions, customer mapping, webhook ingestion with signature verification
- **Subscription Plans**: Free / Pro / Enterprise tiers with monthly credit grants
- **Usage Metering**: Track AI feature usage, storage, and quiz consumption
- **Payment History**: Full transaction audit trail
- **Webhook Idempotency**: Processed event store with 30-day TTL

## Technology Stack

- **Framework**: Spring Boot 3.x (Java 21)
- **Database**: MongoDB (database: `payment`, requires replica set for transactions)
- **Cache**: Caffeine (local, 5s TTL) + Redis (distributed, 60s TTL)
- **Messaging**: RabbitMQ AMQP (topic exchange: `payment.events`)
- **Payment**: Stripe Java SDK
- **Auth**: Keycloak OAuth2 Resource Server

## Port & Context Path

- Port: `8087`
- Context path: `/payment`
- Health: `GET /payment/actuator/health`
- API docs: `GET /payment/api-docs`

## API Endpoints

### Credits
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/credits/balance` | Get current user balance |
| GET | `/api/v1/credits/history` | Get transaction history (paginated) |
| POST | `/api/v1/credits/consume` | Consume credits (service-to-service) |

### Subscriptions
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/subscriptions/plans` | List all plans |
| GET | `/api/v1/subscriptions/current` | Get current subscription |
| POST | `/api/v1/subscriptions/subscribe` | Create subscription checkout |
| PUT | `/api/v1/subscriptions/change` | Change plan |
| POST | `/api/v1/subscriptions/cancel` | Cancel subscription |

### Payments & Webhooks
| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/v1/payments/history` | Get payment history (paginated) |
| POST | `/webhooks/stripe` | Stripe webhook endpoint (no auth required) |

## Subscription Plans

| Plan | Monthly Credits | Storage | AI Generations | Price |
|------|----------------|---------|----------------|-------|
| FREE | 50 | 1 GB | 5 | $0 |
| PRO | 500 | 20 GB | 100 | $9.99/mo |
| ENTERPRISE | 2000 | 100 GB | 500 | $29.99/mo |

## Environment Variables

```env
MONGO_URI=mongodb://localhost:27017/payment?replicaSet=rs0
REDIS_HOST=localhost
REDIS_PORT=6379
RABBITMQ_HOST=localhost
RABBITMQ_PORT=5672
RABBITMQ_USERNAME=admin
RABBITMQ_PASSWORD=admin
KEYCLOAK_ISSUER_URI=http://localhost:8080/realms/memap
KEYCLOAK_JWK_SET_URI=http://localhost:8080/realms/memap/protocol/openid-connect/certs
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
```

## Run Locally

```bash
cp .env.example .env
# Fill in .env values — MongoDB must run as replica set
./mvnw spring-boot:run
```

## Test

```bash
./mvnw test
```

## MongoDB Collections

| Collection | Description |
|------------|-------------|
| `tbl_credit_accounts` | Per-user credit balance |
| `tbl_credit_transactions` | Ledger entries |
| `tbl_subscriptions` | User subscriptions |
| `tbl_payment_history` | Stripe payment records |
| `tbl_stripe_customers` | userId → Stripe customerId mapping |
| `tbl_processed_stripe_events` | Webhook idempotency store (30d TTL) |

## Error Codes (7xxx)

| Code | Description | HTTP Status |
|------|-------------|-------------|
| 7000 | Credit account not found | 404 |
| 7001 | Insufficient credits | 402 |
| 7002 | Duplicate transaction | 409 |
| 7011 | Invalid plan | 400 |
| 7012 | Already subscribed | 409 |
| 7013 | No active subscription | 404 |
| 7050 | Invalid Stripe webhook signature | 400 |
| 7099 | Feature pending | 501 |
