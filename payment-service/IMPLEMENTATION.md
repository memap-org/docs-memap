# Payment Service Implementation Plan

## Overview

This document outlines the step-by-step implementation plan for the Payment Service, a NestJS-based microservice handling credit management, subscription billing, and Stripe payment processing for the MeMap platform.

## Implementation Phases

```
Phase 1: Project Setup & Core Infrastructure     [Week 1]
Phase 2: Database Schema & Prisma Setup          [Week 1-2]
Phase 3: Credit Management Module                [Week 2-3]
Phase 4: Stripe Integration                      [Week 3-4]
Phase 5: Subscription Module                     [Week 4-5]
Phase 6: Webhook Processing                      [Week 5]
Phase 7: Service Integration & Testing           [Week 6]
Phase 8: Documentation & Deployment              [Week 6-7]
```

---

## Phase 1: Project Setup & Core Infrastructure

### 1.1 Initialize NestJS Project

```bash
# Create new NestJS project
nest new payment-service
cd payment-service

# Install core dependencies
npm install @nestjs/config @nestjs/swagger swagger-ui-express
npm install class-validator class-transformer
npm install @nestjs/passport passport passport-jwt
npm install -D @types/passport-jwt
```

### 1.2 Project Structure Setup

```
payment-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   ├── app.controller.ts
│   │
│   ├── config/
│   │   ├── config.module.ts
│   │   ├── database.config.ts
│   │   ├── stripe.config.ts
│   │   └── keycloak.config.ts        # Keycloak configuration
│   │
│   ├── prisma/
│   │   ├── prisma.module.ts
│   │   ├── prisma.service.ts
│   │   └── prisma.middleware.ts
│   │
│   ├── common/
│   │   ├── strategies/
│   │   │   └── keycloak-jwt.strategy.ts  # Keycloak JWT validation
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts         # Keycloak auth guard
│   │   │   ├── roles.guard.ts            # Role-based access control
│   │   │   └── service-auth.guard.ts
│   │   ├── decorators/
│   │   │   ├── current-user.decorator.ts # Extract user from JWT
│   │   │   └── roles.decorator.ts        # Role annotation
│   │   ├── filters/
│   │   │   └── http-exception.filter.ts
│   │   ├── interceptors/
│   │   │   ├── response.interceptor.ts
│   │   │   └── logging.interceptor.ts
│   │   ├── dto/
│   │   │   └── api-response.dto.ts
│   │   ├── interfaces/
│   │   │   └── user-context.interface.ts # User context from Keycloak
│   │   └── enums/
│   │       ├── transaction-type.enum.ts
│   │       ├── feature-type.enum.ts
│   │       ├── plan-type.enum.ts
│   │       └── payment-status.enum.ts
│   │
│   └── modules/
│       ├── auth/                         # Authentication module
│       │   ├── auth.module.ts
│       │   └── auth.service.ts
│       ├── credits/
│       ├── subscriptions/
│       ├── payments/
│       ├── packages/
│       ├── webhooks/
│       └── stripe/
│
├── prisma/
│   ├── schema.prisma
│   ├── migrations/
│   └── seed.ts
│
├── test/
│   ├── app.e2e-spec.ts
│   └── jest-e2e.json
│
├── docker-compose.yml
├── Dockerfile
├── .env.example
└── package.json
```

### 1.3 Environment Configuration

```typescript
// src/config/config.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule as NestConfigModule } from '@nestjs/config';

@Module({
  imports: [
    NestConfigModule.forRoot({
      isGlobal: true,
      envFilePath: ['.env.local', '.env'],
    }),
  ],
})
export class ConfigModule {}
```

**.env.example:**

```env
# Server
PORT=8087
NODE_ENV=development

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/payment_db

# Stripe
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_PUBLISHABLE_KEY=pk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx

# Keycloak OAuth2 Configuration
KEYCLOAK_ISSUER_URI=http://localhost:8080/realms/memap
KEYCLOAK_JWK_SET_URI=http://localhost:8080/realms/memap/protocol/openid-connect/certs
KEYCLOAK_CLIENT_ID=payment-service
KEYCLOAK_CLIENT_SECRET=your-client-secret

# Service URLs
IDENTITY_SERVICE_URL=http://localhost:8081
PROFILE_SERVICE_URL=http://localhost:8082
```

### 1.4 Keycloak Authentication Setup

**Install Passport JWT dependencies:**

```bash
npm install @nestjs/passport passport passport-jwt jwks-rsa
npm install -D @types/passport-jwt
```

**Tasks:**

- [ ] Create Keycloak JWT strategy
- [ ] Create JWT auth guard
- [ ] Create custom authorities converter for Keycloak roles
- [ ] Create current user decorator
- [ ] Configure bearer token resolver (header, cookie, custom header)

**Keycloak JWT Strategy:**

```typescript
// src/common/strategies/keycloak-jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { passportJwtSecret } from 'jwks-rsa';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class KeycloakJwtStrategy extends PassportStrategy(
  Strategy,
  'keycloak',
) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromExtractors([
        // 1. Standard Authorization header
        ExtractJwt.fromAuthHeaderAsBearerToken(),
        // 2. Custom "Bear" header for Swagger testing
        (request) => request?.headers?.['bear'],
        // 3. Cookie named 'token'
        (request) => request?.cookies?.['token'],
      ]),
      secretOrKeyProvider: passportJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: configService.get('KEYCLOAK_JWK_SET_URI'),
      }),
      issuer: configService.get('KEYCLOAK_ISSUER_URI'),
      algorithms: ['RS256'],
    });
  }

  async validate(payload: KeycloakTokenPayload): Promise<UserContext> {
    return {
      userId: payload.sub,
      email: payload.email,
      username: payload.preferred_username,
      roles: this.extractRoles(payload),
    };
  }

  private extractRoles(payload: KeycloakTokenPayload): string[] {
    const realmRoles = payload.realm_access?.roles || [];
    const clientRoles =
      payload.resource_access?.['payment-service']?.roles || [];
    return [...realmRoles, ...clientRoles].map(
      (role) => `ROLE_${role.toUpperCase()}`,
    );
  }
}

// Types
interface KeycloakTokenPayload {
  sub: string;
  email: string;
  preferred_username: string;
  realm_access?: { roles: string[] };
  resource_access?: { [key: string]: { roles: string[] } };
  iat: number;
  exp: number;
}

interface UserContext {
  userId: string;
  email: string;
  username: string;
  roles: string[];
}
```

**JWT Auth Guard:**

```typescript
// src/common/guards/jwt-auth.guard.ts
import { Injectable, ExecutionContext } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('keycloak') {
  canActivate(context: ExecutionContext) {
    return super.canActivate(context);
  }
}
```

**Current User Decorator:**

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: keyof UserContext | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  },
);
```

**Roles Guard:**

```typescript
// src/common/guards/roles.guard.ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { ROLES_KEY } from '../decorators/roles.decorator';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.getAllAndOverride<string[]>(
      ROLES_KEY,
      [context.getHandler(), context.getClass()],
    );

    if (!requiredRoles) {
      return true;
    }

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some((role) => user.roles?.includes(role));
  }
}
```

### 1.5 Common Components

**Tasks:**

- [ ] Create global exception filter
- [ ] Create response interceptor
- [ ] Create logging interceptor
- [ ] Create service-to-service auth guard
- [ ] Define shared enums
- [ ] Create API response DTO

---

## Phase 2: Database Schema & Prisma Setup

### 2.1 Install Prisma

```bash
npm install prisma @prisma/client
npx prisma init
```

### 2.2 Define Prisma Schema

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Enums
enum TransactionType {
  PURCHASE
  USAGE
  REFUND
  BONUS
  SUBSCRIPTION
}

enum FeatureType {
  AI_GENERATION
  AI_SUMMARY
  AI_QUIZ
  AI_CHAT
  STORAGE
  PREMIUM
}

enum PlanType {
  FREE
  PRO
  ENTERPRISE
}

enum SubscriptionStatus {
  ACTIVE
  CANCELLED
  PAST_DUE
  TRIALING
}

enum PaymentStatus {
  PENDING
  COMPLETED
  FAILED
  REFUNDED
}

// Models
model CreditAccount {
  id             String   @id @default(uuid())
  userId         String   @unique @map("user_id")
  balance        Int      @default(0)
  lifetimeEarned Int      @default(0) @map("lifetime_earned")
  lifetimeSpent  Int      @default(0) @map("lifetime_spent")
  isActive       Boolean  @default(true) @map("is_active")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")
  deletedAt      DateTime? @map("deleted_at")

  transactions CreditTransaction[]

  @@map("credit_accounts")
}

model CreditTransaction {
  id          String          @id @default(uuid())
  accountId   String          @map("account_id")
  type        TransactionType
  amount      Int
  featureType FeatureType?    @map("feature_type")
  description String?
  referenceId String?         @map("reference_id")
  metadata    Json?
  isActive    Boolean         @default(true) @map("is_active")
  createdAt   DateTime        @default(now()) @map("created_at")
  deletedAt   DateTime?       @map("deleted_at")

  account CreditAccount @relation(fields: [accountId], references: [id])

  @@index([accountId])
  @@index([type])
  @@index([featureType])
  @@index([createdAt])
  @@map("credit_transactions")
}

model Subscription {
  id                   String             @id @default(uuid())
  userId               String             @unique @map("user_id")
  stripeCustomerId     String?            @map("stripe_customer_id")
  stripeSubscriptionId String?            @map("stripe_subscription_id")
  planType             PlanType           @default(FREE) @map("plan_type")
  status               SubscriptionStatus @default(ACTIVE)
  currentPeriodStart   DateTime?          @map("current_period_start")
  currentPeriodEnd     DateTime?          @map("current_period_end")
  monthlyCreditsLimit  Int                @default(50) @map("monthly_credits_limit")
  monthlyCreditsUsed   Int                @default(0) @map("monthly_credits_used")
  storageLimitMb       Int                @default(100) @map("storage_limit_mb")
  aiGenerationsLimit   Int                @default(3) @map("ai_generations_limit")
  aiGenerationsUsed    Int                @default(0) @map("ai_generations_used")
  cancelAtPeriodEnd    Boolean            @default(false) @map("cancel_at_period_end")
  isActive             Boolean            @default(true) @map("is_active")
  createdAt            DateTime           @default(now()) @map("created_at")
  updatedAt            DateTime           @updatedAt @map("updated_at")
  deletedAt            DateTime?          @map("deleted_at")

  @@index([stripeCustomerId])
  @@map("subscriptions")
}

model PaymentHistory {
  id              String        @id @default(uuid())
  userId          String        @map("user_id")
  stripePaymentId String?       @unique @map("stripe_payment_id")
  stripeInvoiceId String?       @map("stripe_invoice_id")
  amount          Decimal       @db.Decimal(10, 2)
  currency        String        @default("USD")
  status          PaymentStatus @default(PENDING)
  paymentMethod   String?       @map("payment_method")
  description     String?
  metadata        Json?
  isActive        Boolean       @default(true) @map("is_active")
  createdAt       DateTime      @default(now()) @map("created_at")
  deletedAt       DateTime?     @map("deleted_at")

  @@index([userId])
  @@map("payment_history")
}

model StorageUsage {
  id             String   @id @default(uuid())
  userId         String   @unique @map("user_id")
  usedBytes      BigInt   @default(0) @map("used_bytes")
  limitBytes     BigInt   @default(104857600) @map("limit_bytes")
  lastCalculated DateTime @default(now()) @map("last_calculated")
  isActive       Boolean  @default(true) @map("is_active")
  createdAt      DateTime @default(now()) @map("created_at")
  updatedAt      DateTime @updatedAt @map("updated_at")
  deletedAt      DateTime? @map("deleted_at")

  @@map("storage_usage")
}

model CreditPackage {
  id           String  @id @default(uuid())
  name         String  @unique
  credits      Int
  bonusCredits Int     @default(0) @map("bonus_credits")
  price        Decimal @db.Decimal(10, 2)
  currency     String  @default("USD")
  stripePriceId String? @map("stripe_price_id")
  isPopular    Boolean @default(false) @map("is_popular")
  isActive     Boolean @default(true) @map("is_active")
  createdAt    DateTime @default(now()) @map("created_at")
  updatedAt    DateTime @updatedAt @map("updated_at")
  deletedAt    DateTime? @map("deleted_at")

  @@map("credit_packages")
}

model SubscriptionPlan {
  id                 String   @id @default(uuid())
  name               String   @unique
  planType           PlanType @map("plan_type")
  price              Decimal  @db.Decimal(10, 2)
  currency           String   @default("USD")
  interval           String   @default("month")
  stripePriceId      String?  @map("stripe_price_id")
  monthlyCredits     Int      @map("monthly_credits")
  storageLimitMb     Int      @map("storage_limit_mb")
  aiGenerationsLimit Int      @map("ai_generations_limit")
  features           Json?
  isPopular          Boolean  @default(false) @map("is_popular")
  isActive           Boolean  @default(true) @map("is_active")
  createdAt          DateTime @default(now()) @map("created_at")
  updatedAt          DateTime @updatedAt @map("updated_at")
  deletedAt          DateTime? @map("deleted_at")

  @@map("subscription_plans")
}
```

### 2.3 Prisma Service Setup

```typescript
// src/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService
  extends PrismaClient
  implements OnModuleInit, OnModuleDestroy
{
  constructor() {
    super({
      log: ['query', 'info', 'warn', 'error'],
    });
  }

  async onModuleInit() {
    await this.$connect();

    // Soft delete middleware
    this.$use(async (params, next) => {
      // Soft delete: convert delete to update
      if (params.action === 'delete') {
        params.action = 'update';
        params.args['data'] = { deletedAt: new Date(), isActive: false };
      }
      if (params.action === 'deleteMany') {
        params.action = 'updateMany';
        if (params.args.data !== undefined) {
          params.args.data['deletedAt'] = new Date();
          params.args.data['isActive'] = false;
        } else {
          params.args['data'] = { deletedAt: new Date(), isActive: false };
        }
      }

      // Filter soft-deleted records on find
      if (params.action === 'findUnique' || params.action === 'findFirst') {
        params.action = 'findFirst';
        params.args.where['deletedAt'] = null;
      }
      if (params.action === 'findMany') {
        if (params.args.where) {
          if (params.args.where.deletedAt === undefined) {
            params.args.where['deletedAt'] = null;
          }
        } else {
          params.args['where'] = { deletedAt: null };
        }
      }

      return next(params);
    });
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

### 2.4 Run Migrations

```bash
# Create initial migration
npx prisma migrate dev --name init

# Generate Prisma client
npx prisma generate

# Seed database
npx prisma db seed
```

**Tasks:**

- [ ] Define complete Prisma schema
- [ ] Create PrismaService with soft delete middleware
- [ ] Create PrismaModule
- [ ] Run initial migration
- [ ] Create seed data for packages and plans

---

## Phase 3: Credit Management Module

### 3.1 Module Structure

```
src/modules/credits/
├── credits.module.ts
├── credits.controller.ts
├── credits.service.ts
├── dto/
│   ├── credit-balance.dto.ts
│   ├── check-credits.dto.ts
│   ├── consume-credits.dto.ts
│   ├── add-credits.dto.ts
│   └── transaction-history.dto.ts
└── tests/
    ├── credits.controller.spec.ts
    └── credits.service.spec.ts
```

### 3.2 Credit Service Implementation

```typescript
// src/modules/credits/credits.service.ts
import { Injectable, BadRequestException } from '@nestjs/common';
import { PrismaService } from '../../prisma/prisma.service';
import { TransactionType, FeatureType } from '@prisma/client';

@Injectable()
export class CreditService {
  constructor(private prisma: PrismaService) {}

  async getOrCreateAccount(userId: string) {
    let account = await this.prisma.creditAccount.findUnique({
      where: { userId },
    });

    if (!account) {
      account = await this.prisma.creditAccount.create({
        data: { userId },
      });
    }

    return account;
  }

  async getBalance(userId: string) {
    const account = await this.getOrCreateAccount(userId);
    const subscription = await this.prisma.subscription.findUnique({
      where: { userId },
    });

    return {
      accountId: account.id,
      userId: account.userId,
      balance: account.balance,
      lifetimeEarned: account.lifetimeEarned,
      lifetimeSpent: account.lifetimeSpent,
      subscription: subscription
        ? {
            plan: subscription.planType,
            status: subscription.status,
            monthlyCredits: subscription.monthlyCreditsLimit,
            creditsUsedThisMonth: subscription.monthlyCreditsUsed,
            periodEnd: subscription.currentPeriodEnd,
          }
        : null,
    };
  }

  async checkAvailability(
    userId: string,
    featureType: FeatureType,
    quantity = 1,
  ) {
    const account = await this.getOrCreateAccount(userId);
    const creditCost = this.getFeatureCost(featureType);
    const requiredCredits = creditCost * quantity;

    return {
      available: account.balance >= requiredCredits,
      currentBalance: account.balance,
      requiredCredits,
      remainingAfter: account.balance - requiredCredits,
    };
  }

  async consumeCredits(
    userId: string,
    featureType: FeatureType,
    quantity = 1,
    description?: string,
    referenceId?: string,
  ) {
    const creditCost = this.getFeatureCost(featureType);
    const amount = creditCost * quantity;

    return this.prisma.$transaction(async (tx) => {
      const account = await tx.creditAccount.findUnique({
        where: { userId },
      });

      if (!account || account.balance < amount) {
        throw new BadRequestException({
          code: 7010,
          message: 'Insufficient credits',
          result: {
            currentBalance: account?.balance || 0,
            requiredCredits: amount,
          },
        });
      }

      // Deduct credits
      await tx.creditAccount.update({
        where: { id: account.id },
        data: {
          balance: { decrement: amount },
          lifetimeSpent: { increment: amount },
        },
      });

      // Create transaction record
      const transaction = await tx.creditTransaction.create({
        data: {
          accountId: account.id,
          type: TransactionType.USAGE,
          amount: -amount,
          featureType,
          description: description || `Used ${featureType}`,
          referenceId,
        },
      });

      return {
        transactionId: transaction.id,
        creditsConsumed: amount,
        newBalance: account.balance - amount,
      };
    });
  }

  async addCredits(
    userId: string,
    amount: number,
    type: TransactionType,
    description: string,
    referenceId?: string,
  ) {
    return this.prisma.$transaction(async (tx) => {
      const account = await this.getOrCreateAccount(userId);

      await tx.creditAccount.update({
        where: { id: account.id },
        data: {
          balance: { increment: amount },
          lifetimeEarned: { increment: amount },
        },
      });

      const transaction = await tx.creditTransaction.create({
        data: {
          accountId: account.id,
          type,
          amount,
          description,
          referenceId,
        },
      });

      return {
        transactionId: transaction.id,
        creditsAdded: amount,
        newBalance: account.balance + amount,
      };
    });
  }

  private getFeatureCost(featureType: FeatureType): number {
    const costs: Record<FeatureType, number> = {
      AI_GENERATION: 10,
      AI_SUMMARY: 2,
      AI_QUIZ: 5,
      AI_CHAT: 3,
      STORAGE: 5,
      PREMIUM: 1,
    };
    return costs[featureType] || 1;
  }
}
```

### 3.3 Credit Controller

```typescript
// src/modules/credits/credits.controller.ts
@Controller('payment/credits')
@ApiTags('Credits')
export class CreditController {
  constructor(private creditService: CreditService) {}

  @Get('balance')
  @UseGuards(JwtAuthGuard)
  @ApiOperation({ summary: 'Get credit balance' })
  async getBalance(@CurrentUser() user: any) {
    return this.creditService.getBalance(user.userId);
  }

  @Get('history')
  @UseGuards(JwtAuthGuard)
  @ApiOperation({ summary: 'Get transaction history' })
  async getHistory(
    @CurrentUser() user: any,
    @Query() query: TransactionHistoryDto,
  ) {
    return this.creditService.getHistory(user.userId, query);
  }

  @Post('check')
  @UseGuards(JwtAuthGuard)
  @ApiOperation({ summary: 'Check credit availability' })
  async checkAvailability(
    @CurrentUser() user: any,
    @Body() dto: CheckCreditsDto,
  ) {
    return this.creditService.checkAvailability(
      user.userId,
      dto.featureType,
      dto.quantity,
    );
  }

  @Post('consume')
  @UseGuards(ServiceAuthGuard)
  @ApiOperation({ summary: 'Consume credits (internal)' })
  async consumeCredits(@Body() dto: ConsumeCreditsDto) {
    return this.creditService.consumeCredits(
      dto.userId,
      dto.featureType,
      dto.quantity,
      dto.description,
      dto.referenceId,
    );
  }
}
```

**Tasks:**

- [ ] Create CreditModule
- [ ] Implement CreditService with all methods
- [ ] Create CreditController with endpoints
- [ ] Define DTOs with validation
- [ ] Write unit tests for CreditService
- [ ] Write integration tests for CreditController

---

## Phase 4: Stripe Integration

### 4.1 Install Stripe SDK

```bash
npm install stripe
```

### 4.2 Stripe Module Structure

```
src/modules/stripe/
├── stripe.module.ts
├── stripe.service.ts
├── stripe.config.ts
└── tests/
    └── stripe.service.spec.ts
```

### 4.3 Stripe Service Implementation

```typescript
// src/modules/stripe/stripe.service.ts
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import Stripe from 'stripe';

@Injectable()
export class StripeService {
  private stripe: Stripe;

  constructor(private configService: ConfigService) {
    this.stripe = new Stripe(this.configService.get('STRIPE_SECRET_KEY'), {
      apiVersion: '2023-10-16',
    });
  }

  async createCustomer(
    email: string,
    userId: string,
  ): Promise<Stripe.Customer> {
    return this.stripe.customers.create({
      email,
      metadata: { userId },
    });
  }

  async createCheckoutSession(
    customerId: string,
    priceId: string,
    mode: 'payment' | 'subscription',
    successUrl: string,
    cancelUrl: string,
    metadata?: Record<string, string>,
  ): Promise<Stripe.Checkout.Session> {
    return this.stripe.checkout.sessions.create({
      customer: customerId,
      payment_method_types: ['card'],
      line_items: [{ price: priceId, quantity: 1 }],
      mode,
      success_url: successUrl,
      cancel_url: cancelUrl,
      metadata,
    });
  }

  async createSubscription(
    customerId: string,
    priceId: string,
  ): Promise<Stripe.Subscription> {
    return this.stripe.subscriptions.create({
      customer: customerId,
      items: [{ price: priceId }],
    });
  }

  async cancelSubscription(
    subscriptionId: string,
    cancelAtPeriodEnd = true,
  ): Promise<Stripe.Subscription> {
    return this.stripe.subscriptions.update(subscriptionId, {
      cancel_at_period_end: cancelAtPeriodEnd,
    });
  }

  constructWebhookEvent(payload: Buffer, signature: string): Stripe.Event {
    return this.stripe.webhooks.constructEvent(
      payload,
      signature,
      this.configService.get('STRIPE_WEBHOOK_SECRET'),
    );
  }
}
```

**Tasks:**

- [ ] Create StripeModule
- [ ] Implement StripeService with all Stripe operations
- [ ] Configure Stripe webhook signature verification
- [ ] Create test helpers for mocking Stripe API
- [ ] Write unit tests

---

## Phase 5: Subscription Module

### 5.1 Module Structure

```
src/modules/subscriptions/
├── subscriptions.module.ts
├── subscriptions.controller.ts
├── subscriptions.service.ts
├── dto/
│   ├── subscribe.dto.ts
│   ├── change-plan.dto.ts
│   ├── cancel-subscription.dto.ts
│   └── subscription-response.dto.ts
└── tests/
    ├── subscriptions.controller.spec.ts
    └── subscriptions.service.spec.ts
```

### 5.2 Key Features

- Get subscription plans
- Subscribe to a plan (create Stripe checkout)
- Get current subscription
- Change subscription plan
- Cancel subscription
- Renew monthly credits (scheduled job)

**Tasks:**

- [ ] Create SubscriptionModule
- [ ] Implement SubscriptionService
- [ ] Create SubscriptionController
- [ ] Implement monthly credit renewal (cron job)
- [ ] Write tests

---

## Phase 6: Webhook Processing

### 6.1 Webhook Controller

```typescript
// src/modules/webhooks/webhooks.controller.ts
@Controller('payment/webhooks')
export class WebhookController {
  constructor(private webhookService: WebhookService) {}

  @Post('stripe')
  @HttpCode(200)
  async handleStripeWebhook(
    @Headers('stripe-signature') signature: string,
    @Req() request: RawBodyRequest<Request>,
  ) {
    const event = this.stripeService.constructWebhookEvent(
      request.rawBody,
      signature,
    );

    await this.webhookService.handleEvent(event);

    return { received: true };
  }
}
```

### 6.2 Webhook Event Handlers

| Event                           | Handler                                       |
| ------------------------------- | --------------------------------------------- |
| `checkout.session.completed`    | Process credit purchase or subscription start |
| `payment_intent.succeeded`      | Log successful payment                        |
| `payment_intent.payment_failed` | Handle failed payment                         |
| `customer.subscription.created` | Activate subscription                         |
| `customer.subscription.updated` | Update subscription status                    |
| `customer.subscription.deleted` | Cancel subscription                           |
| `invoice.payment_succeeded`     | Add subscription credits                      |
| `invoice.payment_failed`        | Handle failed invoice                         |

**Tasks:**

- [ ] Create WebhookModule
- [ ] Implement WebhookService with event handlers
- [ ] Create WebhookController with raw body parsing
- [ ] Implement idempotency (store processed event IDs)
- [ ] Write integration tests

---

## Phase 7: Service Integration & Testing

### 7.1 Integration with Other Services

```typescript
// Service-to-service communication
// Other services call Payment Service before consuming credits

// Example: Roadmap AI Service integration
@Injectable()
export class AiGenerationService {
  async generateRoadmap(userId: string, topic: string) {
    // 1. Check credit availability
    const available = await this.paymentClient.checkCredits(
      userId,
      'AI_GENERATION',
    );

    if (!available) {
      throw new InsufficientCreditsException();
    }

    // 2. Generate roadmap
    const roadmap = await this.aiService.generate(topic);

    // 3. Consume credits
    await this.paymentClient.consumeCredits(userId, 'AI_GENERATION');

    return roadmap;
  }
}
```

### 7.2 Testing Strategy

```bash
# Unit tests
npm run test

# E2E tests
npm run test:e2e

# Test coverage
npm run test:cov

# Stripe webhook testing (use Stripe CLI)
stripe listen --forward-to localhost:8087/payment/webhooks/stripe
```

**Tasks:**

- [ ] Implement HTTP client for internal service calls
- [ ] Create integration tests for credit consumption flow
- [ ] Test Stripe webhook processing with Stripe CLI
- [ ] Load testing for high-traffic scenarios
- [ ] Security testing (auth, rate limiting)

---

## Phase 8: Documentation & Deployment

### 8.1 API Documentation (Swagger)

```typescript
// main.ts
const config = new DocumentBuilder()
  .setTitle('Payment Service API')
  .setDescription('Credit management and payment processing')
  .setVersion('1.0')
  .addBearerAuth()
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('api-docs', app, document);
```

### 8.2 Docker Setup

```dockerfile
# Dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
COPY prisma ./prisma/

RUN npm ci --only=production
RUN npx prisma generate

COPY dist ./dist

ENV NODE_ENV=production
EXPOSE 8087

CMD ["node", "dist/main.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  payment-service:
    build: .
    ports:
      - '8087:8087'
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/payment_db
      - STRIPE_SECRET_KEY=${STRIPE_SECRET_KEY}
      - STRIPE_WEBHOOK_SECRET=${STRIPE_WEBHOOK_SECRET}
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=payment_db
    volumes:
      - postgres_data:/var/lib/postgresql/data

volumes:
  postgres_data:
```

### 8.3 Deployment Checklist

- [ ] Environment variables configured
- [ ] Database migrations applied
- [ ] Stripe webhook endpoint registered
- [ ] Health check endpoints working
- [ ] Logging and monitoring configured
- [ ] Rate limiting enabled
- [ ] SSL/TLS configured
- [ ] Backup strategy implemented

---

## Implementation Checklist Summary

### Week 1

- [ ] Initialize NestJS project
- [ ] Setup project structure
- [ ] Configure environment
- [ ] Create common components (guards, filters, interceptors)
- [ ] Setup Prisma and define schema
- [ ] Run initial migration

### Week 2

- [ ] Implement CreditService
- [ ] Create CreditController
- [ ] Write credit module tests

### Week 3

- [ ] Implement StripeService
- [ ] Create PackagesModule
- [ ] Implement credit purchase flow

### Week 4

- [ ] Implement SubscriptionService
- [ ] Create SubscriptionController
- [ ] Implement plan change logic

### Week 5

- [ ] Implement WebhookService
- [ ] Handle all Stripe events
- [ ] Implement idempotency

### Week 6

- [ ] Integration testing
- [ ] E2E testing
- [ ] Performance testing

### Week 7

- [ ] API documentation
- [ ] Docker configuration
- [ ] Deployment scripts
- [ ] Production deployment

---

## Related Documentation

- [README.md](./README.md) - Service overview
- [API.md](./API.md) - API endpoint reference
- [ARCHITECTURE.md](./ARCHITECTURE.md) - System architecture
- [../ERROR_CODES.md](../ERROR_CODES.md) - Error code reference
