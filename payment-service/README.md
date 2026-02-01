# Payment Service

## Overview

The Payment Service is a NestJS-based microservice that handles all payment and credit management for the MeMap platform. It integrates with Stripe for secure payment processing and manages user credits for premium features including AI roadmap generation, storage upgrades, and advanced learning features.

## Key Features

- **Credit Management**: Purchase, track, and consume credits for platform features
- **Stripe Integration**: Secure payment processing with Stripe Checkout and webhooks
- **Subscription Plans**: Support for different pricing tiers (Free, Pro, Enterprise)
- **Usage Metering**: Track AI feature usage, storage consumption, and roadmap limits
- **Invoice Management**: Generate and retrieve payment history and invoices
- **Webhook Handling**: Process Stripe events for payment confirmations and failures
- **Credit Transactions**: Detailed transaction history and audit trail

## Technology Stack

- **Framework**: NestJS 10.x
- **Language**: TypeScript 5.x
- **Database**: PostgreSQL 15+
- **ORM**: TypeORM / Prisma
- **Payment Gateway**: Stripe API
- **Authentication**: OAuth2 Resource Server (Keycloak)
- **Validation**: class-validator, class-transformer
- **Documentation**: Swagger/OpenAPI
- **Build Tool**: npm/yarn

## Port Configuration

- **Service Port**: `8087`
- **Base URL**: `http://localhost:8087`
- **Context Path**: `/payment`

## Credit System

### Credit Types

| Credit Type    | Description             | Usage                                 |
| -------------- | ----------------------- | ------------------------------------- |
| AI_CREDIT      | AI feature credits      | Roadmap generation, summaries, Q&A    |
| STORAGE_CREDIT | Storage space credits   | File uploads, avatar storage          |
| PREMIUM_CREDIT | Premium feature credits | Advanced learning, unlimited roadmaps |

### Credit Packages

| Package    | Credits | Price (USD) | Bonus     |
| ---------- | ------- | ----------- | --------- |
| Starter    | 100     | $4.99       | -         |
| Basic      | 500     | $19.99      | 10% bonus |
| Pro        | 1,500   | $49.99      | 20% bonus |
| Enterprise | 5,000   | $149.99     | 30% bonus |

### Feature Credit Costs

| Feature                | Credits Required |
| ---------------------- | ---------------- |
| AI Roadmap Generation  | 10 credits       |
| Node Summary           | 2 credits        |
| Quiz Generation        | 5 credits        |
| Q&A Chat (per session) | 3 credits        |
| Storage (per 100MB)    | 5 credits        |

## Subscription Plans

### Free Tier

- 50 credits/month
- 100MB storage
- 3 AI generations/month
- Basic roadmap features

### Pro Tier ($9.99/month)

- 500 credits/month
- 1GB storage
- 50 AI generations/month
- Priority support
- Advanced analytics

### Enterprise Tier ($29.99/month)

- 2,000 credits/month
- 10GB storage
- Unlimited AI generations
- Custom integrations
- Dedicated support

## Security & Authentication

### Keycloak Integration

The Payment Service integrates with Keycloak for OAuth2/JWT-based authentication. All API endpoints (except webhooks) require a valid JWT token issued by Keycloak.

### Authentication Flow

```
User → Keycloak (Login) → JWT Token → Payment Service → Validate via JWK Set
```

### Token Sources

| Method               | Header/Cookie                   | Priority        |
| -------------------- | ------------------------------- | --------------- |
| Authorization Header | `Authorization: Bearer {token}` | 1 (Primary)     |
| Bear Header          | `Bear: {token}`                 | 2 (Testing)     |
| Cookie               | `token={token}`                 | 3 (Web clients) |

### Role-Based Access Control

| Role           | Permissions                                             |
| -------------- | ------------------------------------------------------- |
| `ROLE_USER`    | View balance, purchase credits, manage own subscription |
| `ROLE_PREMIUM` | All user permissions + premium features                 |
| `ROLE_ADMIN`   | All permissions + manage all accounts, refunds          |

### Keycloak Configuration

Required environment variables:

```bash
KEYCLOAK_ISSUER_URI=http://localhost:8080/realms/memap
KEYCLOAK_JWK_SET_URI=http://localhost:8080/realms/memap/protocol/openid-connect/certs
KEYCLOAK_CLIENT_ID=payment-service
KEYCLOAK_CLIENT_SECRET=your-client-secret
```

## Stripe Integration

### Supported Payment Methods

- Credit/Debit Cards (Visa, MasterCard, Amex)
- Digital Wallets (Apple Pay, Google Pay)
- Bank Transfers (SEPA, ACH)

### Webhook Events

| Event                           | Description               |
| ------------------------------- | ------------------------- |
| `checkout.session.completed`    | Payment successful        |
| `payment_intent.succeeded`      | Credit purchase confirmed |
| `customer.subscription.created` | Subscription started      |
| `customer.subscription.updated` | Plan changed              |
| `customer.subscription.deleted` | Subscription cancelled    |
| `invoice.payment_failed`        | Payment failure           |

## Database Schema

### Core Tables

> **Note**: All tables implement soft delete using `deleted_at` timestamp and `is_active` flag. Records with `deleted_at IS NOT NULL` or `is_active = false` are considered inactive and excluded from normal queries.

```
┌─────────────────────────────────────────────────────────────┐
│                    credit_accounts                          │
├─────────────────────────────────────────────────────────────┤
│ id (PK)         │ UUID                                      │
│ user_id         │ VARCHAR(255) - External user reference    │
│ balance         │ INTEGER - Current credit balance          │
│ lifetime_earned │ INTEGER - Total credits ever earned       │
│ lifetime_spent  │ INTEGER - Total credits ever spent        │
│ is_active       │ BOOLEAN DEFAULT true - Active status      │
│ created_at      │ TIMESTAMP                                 │
│ updated_at      │ TIMESTAMP                                 │
│ deleted_at      │ TIMESTAMP NULL - Soft delete marker       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    credit_transactions                       │
├─────────────────────────────────────────────────────────────┤
│ id (PK)         │ UUID                                      │
│ account_id (FK) │ UUID - Reference to credit_accounts       │
│ type            │ ENUM (PURCHASE, USAGE, REFUND, BONUS)     │
│ amount          │ INTEGER - Positive for credit, negative   │
│ feature_type    │ VARCHAR - AI_GENERATION, STORAGE, etc.    │
│ description     │ TEXT - Transaction description            │
│ reference_id    │ VARCHAR - External reference (Stripe ID)  │
│ is_active       │ BOOLEAN DEFAULT true - Active status      │
│ created_at      │ TIMESTAMP                                 │
│ deleted_at      │ TIMESTAMP NULL - Soft delete marker       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    subscriptions                             │
├─────────────────────────────────────────────────────────────┤
│ id (PK)             │ UUID                                  │
│ user_id             │ VARCHAR(255)                          │
│ stripe_customer_id  │ VARCHAR(255)                          │
│ stripe_subscription │ VARCHAR(255)                          │
│ plan_type           │ ENUM (FREE, PRO, ENTERPRISE)          │
│ status              │ ENUM (ACTIVE, CANCELLED, PAST_DUE)    │
│ current_period_start│ TIMESTAMP                             │
│ current_period_end  │ TIMESTAMP                             │
│ is_active           │ BOOLEAN DEFAULT true - Active status  │
│ created_at          │ TIMESTAMP                             │
│ updated_at          │ TIMESTAMP                             │
│ deleted_at          │ TIMESTAMP NULL - Soft delete marker   │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                    payment_history                           │
├─────────────────────────────────────────────────────────────┤
│ id (PK)             │ UUID                                  │
│ user_id             │ VARCHAR(255)                          │
│ stripe_payment_id   │ VARCHAR(255)                          │
│ amount              │ DECIMAL(10,2)                         │
│ currency            │ VARCHAR(3)                            │
│ status              │ ENUM (PENDING, COMPLETED, FAILED)     │
│ payment_method      │ VARCHAR(50)                           │
│ description         │ TEXT                                  │
│ is_active           │ BOOLEAN DEFAULT true - Active status  │
│ created_at          │ TIMESTAMP                             │
│ deleted_at          │ TIMESTAMP NULL - Soft delete marker   │
└─────────────────────────────────────────────────────────────┘
```

### Soft Delete Pattern

```typescript
// TypeORM Entity Example
@Entity('credit_accounts')
export class CreditAccount {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column({ default: 0 })
  balance: number;

  @Column({ default: true })
  isActive: boolean; // Active status flag

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn()
  deletedAt?: Date; // Soft delete column
}

// Repository usage
await creditAccountRepo.softDelete(id); // Soft delete
await creditAccountRepo.restore(id); // Restore deleted record
await creditAccountRepo.find(); // Excludes soft-deleted
await creditAccountRepo.findWithDeleted(); // Includes soft-deleted
```

## Getting Started

### Prerequisites

- Node.js 18+
- npm or yarn
- PostgreSQL 15+
- Stripe account with API keys
- Keycloak server (for authentication)

### Environment Variables

```env
# Server
PORT=8087
NODE_ENV=development

# Database
DATABASE_URL=postgresql://postgres:password@localhost:5432/payment_db

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Service URLs
IDENTITY_SERVICE_URL=http://localhost:8081
PROFILE_SERVICE_URL=http://localhost:8082

# Keycloak Configuration
KEYCLOAK_ISSUER_URI=http://localhost:8080/realms/memap
KEYCLOAK_JWK_SET_URI=http://localhost:8080/realms/memap/protocol/openid-connect/certs
KEYCLOAK_CLIENT_ID=payment-service
KEYCLOAK_CLIENT_SECRET=your-client-secret
```

### Running the Service

```bash
# Navigate to service directory
cd payment-service

# Install dependencies
npm install

# Run database migrations
npm run migration:run

# Start development server
npm run start:dev

# Start production server
npm run start:prod
```

### Testing Stripe Integration

Use Stripe's test card numbers:

- **Success**: `4242 4242 4242 4242`
- **Requires Auth**: `4000 0025 0000 3155`
- **Declined**: `4000 0000 0000 9995`

## Related Services

- **Identity Service**: User authentication and token validation
- **Profile Service**: User profile and quota information
- **Roadmap AI Service**: AI credit consumption tracking
- **File Service**: Storage credit management
- **Learning Service**: Premium feature access

## Documentation

- **[API Documentation](./API.md)** - Complete API reference with examples
- **[Architecture](./ARCHITECTURE.md)** - System design and patterns

## Error Handling

See [ERROR_CODES.md](../ERROR_CODES.md) for payment-specific error codes (7xxx range).
