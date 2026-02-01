# Payment Service Architecture

## Service Overview

The Payment Service is a NestJS-based microservice responsible for all payment processing, credit management, and subscription handling in the MeMap platform. It integrates with Stripe for secure payment processing and Keycloak for OAuth2/JWT authentication, and maintains credit accounts for metered feature usage.

## Authentication Architecture

### Keycloak Integration

The Payment Service uses Keycloak as the OAuth2 Resource Server for authentication and authorization. All requests (except webhooks) must include a valid JWT token issued by Keycloak.

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          Keycloak Authentication Flow                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌──────────┐      ┌──────────────┐      ┌─────────────────┐      ┌──────────┐ │
│  │  Client  │      │   Keycloak   │      │ Payment Service │      │  Stripe  │ │
│  └────┬─────┘      └──────┬───────┘      └────────┬────────┘      └────┬─────┘ │
│       │                   │                       │                    │        │
│       │ 1. Authenticate   │                       │                    │        │
│       │──────────────────>│                       │                    │        │
│       │                   │                       │                    │        │
│       │ 2. JWT Token      │                       │                    │        │
│       │   (access_token)  │                       │                    │        │
│       │<──────────────────│                       │                    │        │
│       │                   │                       │                    │        │
│       │ 3. API Request    │                       │                    │        │
│       │   + Bearer Token  │                       │                    │        │
│       │───────────────────────────────────────────>                    │        │
│       │                   │                       │                    │        │
│       │                   │ 4. Validate JWT       │                    │        │
│       │                   │   (JWK Set URI)       │                    │        │
│       │                   │<──────────────────────│                    │        │
│       │                   │                       │                    │        │
│       │                   │ 5. Public Keys        │                    │        │
│       │                   │──────────────────────>│                    │        │
│       │                   │                       │                    │        │
│       │                   │                       │ 6. Payment Op      │        │
│       │                   │                       │───────────────────>│        │
│       │                   │                       │                    │        │
│       │                   │                       │ 7. Result          │        │
│       │                   │                       │<───────────────────│        │
│       │                   │                       │                    │        │
│       │ 8. Response       │                       │                    │        │
│       │<──────────────────────────────────────────│                    │        │
│       │                   │                       │                    │        │
│  └────┴─────┘      └──────┴───────┘      └────────┴────────┘      └────┴─────┘ │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Token Resolution Strategy

The service supports multiple token sources for flexibility:

| Priority | Source               | Header/Cookie                   | Usage           |
| -------- | -------------------- | ------------------------------- | --------------- |
| 1        | Authorization Header | `Authorization: Bearer {token}` | Standard OAuth2 |
| 2        | Bear Header          | `Bear: {token}`                 | Swagger/Testing |
| 3        | Cookie               | `token={token}`                 | Web Clients     |

### JWT Claims Mapping

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         Keycloak JWT Token Structure                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  {                                                                               │
│    "sub": "user-uuid-123",              // → userId for credit account           │
│    "email": "user@example.com",         // → Stripe customer email               │
│    "preferred_username": "johndoe",     // → Display name                        │
│    "realm_access": {                                                             │
│      "roles": ["USER", "PREMIUM"]       // → Authorization roles                 │
│    },                                                                            │
│    "resource_access": {                                                          │
│      "payment-service": {                                                        │
│        "roles": ["manage_subscription"] // → Service-specific permissions        │
│      }                                                                           │
│    },                                                                            │
│    "iat": 1706659200,                   // → Issued at                           │
│    "exp": 1706662800                    // → Expiration                          │
│  }                                                                               │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              Client Layer                                        │
│                    (Frontend, Mobile Apps, Other Services)                       │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────────┐
│                           API Gateway / Load Balancer                            │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────────┐
│                          Controller Layer (NestJS)                               │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │  CreditController   │  │ SubscriptionCtrl    │  │  WebhookController      │ │
│  │  - GET /balance     │  │ - GET /plans        │  │  - POST /stripe         │ │
│  │  - GET /history     │  │ - GET /current      │  │  - Signature verify     │ │
│  │  - POST /check      │  │ - POST /subscribe   │  │  - Event routing        │ │
│  │  - POST /consume    │  │ - PUT /change       │  │                         │ │
│  └─────────────────────┘  │ - POST /cancel      │  └─────────────────────────┘ │
│                           └─────────────────────┘                               │
│  ┌─────────────────────┐  ┌─────────────────────┐                               │
│  │  PackageController  │  │  PaymentController  │                               │
│  │  - GET /packages    │  │  - GET /history     │                               │
│  │  - POST /purchase   │  │  - GET /invoices    │                               │
│  └─────────────────────┘  │  - GET /storage     │                               │
│                           └─────────────────────┘                               │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────────┐
│                            Service Layer                                         │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ CreditService                                                             │   │
│  │ - getBalance(userId)          - consumeCredits(userId, feature, amount)  │   │
│  │ - getHistory(userId, filters) - addCredits(userId, amount, source)       │   │
│  │ - checkAvailability(userId)   - refundCredits(transactionId)             │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ SubscriptionService                                                       │   │
│  │ - getPlans()                  - createSubscription(userId, planId)       │   │
│  │ - getCurrentSubscription()    - changeSubscription(userId, newPlanId)    │   │
│  │ - cancelSubscription()        - renewMonthlyCredits(subscriptionId)      │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ PaymentService                                                            │   │
│  │ - createCheckoutSession()     - getPaymentHistory(userId)                │   │
│  │ - handlePaymentSuccess()      - getInvoice(invoiceId)                    │   │
│  │ - handlePaymentFailure()      - refundPayment(paymentId)                 │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ StripeService                                                             │   │
│  │ - createCustomer()            - createCheckoutSession()                  │   │
│  │ - createSubscription()        - cancelSubscription()                     │   │
│  │ - constructWebhookEvent()     - getPaymentIntent()                       │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
│  ┌──────────────────────────────────────────────────────────────────────────┐   │
│  │ WebhookService                                                            │   │
│  │ - handleCheckoutComplete()    - handleSubscriptionCreated()              │   │
│  │ - handlePaymentSucceeded()    - handleSubscriptionUpdated()              │   │
│  │ - handlePaymentFailed()       - handleSubscriptionDeleted()              │   │
│  └──────────────────────────────────────────────────────────────────────────┘   │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┼────────────────────────────────────────────┐
│                          Repository Layer (TypeORM)                              │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │ CreditAccountRepo   │  │ TransactionRepo     │  │ SubscriptionRepo        │ │
│  │ - findByUserId      │  │ - createTransaction │  │ - findByUserId          │ │
│  │ - updateBalance     │  │ - findByAccount     │  │ - updateStatus          │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────────┘ │
│  ┌─────────────────────┐  ┌─────────────────────┐                               │
│  │ PaymentHistoryRepo  │  │ StripeCustomerRepo  │                               │
│  │ - findByUserId      │  │ - findByUserId      │                               │
│  │ - createRecord      │  │ - createMapping     │                               │
│  └─────────────────────┘  └─────────────────────┘                               │
└────────────────────────────────────┬────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                               Data Layer                                         │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                           PostgreSQL Database                               │ │
│  │  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────────────────┐ │ │
│  │  │ credit_accounts │  │ transactions    │  │ subscriptions               │ │ │
│  │  │ - id            │  │ - id            │  │ - id                        │ │ │
│  │  │ - user_id       │  │ - account_id    │  │ - user_id                   │ │ │
│  │  │ - balance       │  │ - type          │  │ - stripe_subscription_id    │ │ │
│  │  │ - lifetime_*    │  │ - amount        │  │ - plan_type                 │ │ │
│  │  └─────────────────┘  └─────────────────┘  │ - status                    │ │ │
│  │                                            └─────────────────────────────┘ │ │
│  │  ┌─────────────────┐  ┌─────────────────┐                                  │ │
│  │  │ payment_history │  │ stripe_customers│                                  │ │
│  │  │ - id            │  │ - id            │                                  │ │
│  │  │ - user_id       │  │ - user_id       │                                  │ │
│  │  │ - stripe_id     │  │ - stripe_id     │                                  │ │
│  │  │ - amount        │  │ - email         │                                  │ │
│  │  │ - status        │  │                 │                                  │ │
│  │  └─────────────────┘  └─────────────────┘                                  │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
                                     │
┌────────────────────────────────────┴────────────────────────────────────────────┐
│                            External Services                                     │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │      Keycloak       │  │    Stripe API       │  │   Profile Service       │ │
│  │  - JWT Validation   │  │  - Checkout         │  │   - User info           │ │
│  │  - JWK Set          │  │  - Subscriptions    │  │   - Email               │ │
│  │  - User Roles       │  │  - Webhooks         │  │                         │ │
│  │  - Token Claims     │  │  - Invoices         │  │                         │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────────┘ │
│                                                                                  │
│  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────────┐ │
│  │  Roadmap AI Service │  │    File Service     │  │  Learning Service       │ │
│  │  - Credit check     │  │  - Storage tracking │  │  - Premium features     │ │
│  │  - Usage reporting  │  │  - Credit deduction │  │  - Credit consumption   │ │
│  └─────────────────────┘  └─────────────────────┘  └─────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## Component Details

### 1. Controller Layer

#### CreditController

**Responsibility**: Handle credit balance and transaction endpoints

```typescript
@Controller('payment/credits')
export class CreditController {
  @Get('balance')
  @UseGuards(JwtAuthGuard)
  async getBalance(@CurrentUser() user: User): Promise<CreditBalanceResponse>;

  @Get('history')
  @UseGuards(JwtAuthGuard)
  async getHistory(
    @CurrentUser() user: User,
    @Query() query: HistoryQueryDto,
  ): Promise<PaginatedResponse<CreditTransaction>>;

  @Post('check')
  @UseGuards(JwtAuthGuard)
  async checkAvailability(
    @CurrentUser() user: User,
    @Body() dto: CheckCreditsDto,
  ): Promise<AvailabilityResponse>;

  @Post('consume')
  @UseGuards(ServiceAuthGuard)
  async consumeCredits(
    @Body() dto: ConsumeCreditsDto,
  ): Promise<ConsumeResponse>;
}
```

#### WebhookController

**Responsibility**: Handle Stripe webhook events with signature verification

```typescript
@Controller('payment/webhooks')
export class WebhookController {
  @Post('stripe')
  @UseInterceptors(RawBodyInterceptor)
  async handleStripeWebhook(
    @Headers('stripe-signature') signature: string,
    @RawBody() rawBody: Buffer,
  ): Promise<{ received: boolean }>;
}
```

### 2. Service Layer

#### CreditService

**Responsibility**: Core credit management logic

```typescript
@Injectable()
export class CreditService {
  async getOrCreateAccount(userId: string): Promise<CreditAccount>;

  async getBalance(userId: string): Promise<CreditBalanceDto>;

  async checkAvailability(
    userId: string,
    featureType: FeatureType,
    quantity: number,
  ): Promise<AvailabilityResult>;

  async consumeCredits(
    userId: string,
    featureType: FeatureType,
    quantity: number,
    description?: string,
    referenceId?: string,
  ): Promise<TransactionResult>;

  async addCredits(
    userId: string,
    amount: number,
    type: TransactionType,
    description: string,
    referenceId?: string,
  ): Promise<TransactionResult>;

  @Transactional()
  async processTransaction(
    account: CreditAccount,
    amount: number,
    type: TransactionType,
    featureType?: FeatureType,
  ): Promise<CreditTransaction>;
}
```

#### StripeService

**Responsibility**: Stripe API integration

```typescript
@Injectable()
export class StripeService {
  private stripe: Stripe;

  async createCustomer(user: User): Promise<Stripe.Customer>;

  async createCheckoutSession(
    customerId: string,
    priceId: string,
    mode: 'payment' | 'subscription',
    successUrl: string,
    cancelUrl: string,
    metadata?: Record<string, string>,
  ): Promise<Stripe.Checkout.Session>;

  async createSubscription(
    customerId: string,
    priceId: string,
  ): Promise<Stripe.Subscription>;

  async cancelSubscription(
    subscriptionId: string,
    cancelAtPeriodEnd: boolean,
  ): Promise<Stripe.Subscription>;

  constructWebhookEvent(payload: Buffer, signature: string): Stripe.Event;
}
```

#### WebhookService

**Responsibility**: Process Stripe webhook events

```typescript
@Injectable()
export class WebhookService {
  async handleEvent(event: Stripe.Event): Promise<void> {
    switch (event.type) {
      case 'checkout.session.completed':
        await this.handleCheckoutComplete(event.data.object);
        break;
      case 'customer.subscription.created':
        await this.handleSubscriptionCreated(event.data.object);
        break;
      case 'customer.subscription.updated':
        await this.handleSubscriptionUpdated(event.data.object);
        break;
      case 'customer.subscription.deleted':
        await this.handleSubscriptionDeleted(event.data.object);
        break;
      case 'invoice.payment_succeeded':
        await this.handleInvoicePaymentSucceeded(event.data.object);
        break;
      case 'invoice.payment_failed':
        await this.handleInvoicePaymentFailed(event.data.object);
        break;
    }
  }

  private async handleCheckoutComplete(
    session: Stripe.Checkout.Session,
  ): Promise<void> {
    const { userId, packageId, credits } = session.metadata;

    if (session.mode === 'payment') {
      // Credit package purchase
      await this.creditService.addCredits(
        userId,
        parseInt(credits),
        TransactionType.PURCHASE,
        `Purchased ${packageId}`,
        session.payment_intent as string,
      );
    }
  }
}
```

### 3. Data Models

#### Entities

```typescript
// credit-account.entity.ts
@Entity('credit_accounts')
export class CreditAccount {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ unique: true })
  userId: string;

  @Column({ default: 0 })
  balance: number;

  @Column({ default: 0 })
  lifetimeEarned: number;

  @Column({ default: 0 })
  lifetimeSpent: number;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @OneToMany(() => CreditTransaction, (txn) => txn.account)
  transactions: CreditTransaction[];
}

// credit-transaction.entity.ts
@Entity('credit_transactions')
export class CreditTransaction {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @ManyToOne(() => CreditAccount, (account) => account.transactions)
  @JoinColumn({ name: 'account_id' })
  account: CreditAccount;

  @Column({ type: 'enum', enum: TransactionType })
  type: TransactionType;

  @Column()
  amount: number;

  @Column({ nullable: true })
  featureType?: string;

  @Column({ nullable: true })
  description?: string;

  @Column({ nullable: true })
  referenceId?: string;

  @CreateDateColumn()
  createdAt: Date;
}

// subscription.entity.ts
@Entity('subscriptions')
export class Subscription {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column()
  userId: string;

  @Column({ nullable: true })
  stripeCustomerId: string;

  @Column({ nullable: true })
  stripeSubscriptionId: string;

  @Column({ type: 'enum', enum: PlanType, default: PlanType.FREE })
  planType: PlanType;

  @Column({
    type: 'enum',
    enum: SubscriptionStatus,
    default: SubscriptionStatus.ACTIVE,
  })
  status: SubscriptionStatus;

  @Column({ nullable: true })
  currentPeriodStart: Date;

  @Column({ nullable: true })
  currentPeriodEnd: Date;

  @Column({ default: false })
  cancelAtPeriodEnd: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

#### Enums

```typescript
export enum TransactionType {
  PURCHASE = 'PURCHASE',
  USAGE = 'USAGE',
  REFUND = 'REFUND',
  BONUS = 'BONUS',
  SUBSCRIPTION = 'SUBSCRIPTION',
}

export enum FeatureType {
  AI_ROADMAP_GENERATION = 'AI_ROADMAP_GENERATION',
  AI_NODE_SUMMARY = 'AI_NODE_SUMMARY',
  AI_QUIZ_GENERATION = 'AI_QUIZ_GENERATION',
  AI_QA_CHAT = 'AI_QA_CHAT',
  STORAGE_100MB = 'STORAGE_100MB',
}

export enum PlanType {
  FREE = 'FREE',
  PRO = 'PRO',
  ENTERPRISE = 'ENTERPRISE',
}

export enum SubscriptionStatus {
  ACTIVE = 'ACTIVE',
  CANCELLED = 'CANCELLED',
  PAST_DUE = 'PAST_DUE',
  TRIALING = 'TRIALING',
}
```

## Integration Patterns

### Service-to-Service Communication

Other services call Payment Service to check and consume credits:

```
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│  Roadmap AI      │        │  Payment Service │        │    PostgreSQL    │
│  Service         │        │                  │        │                  │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
         │  POST /credits/check      │                           │
         │  {featureType, userId}    │                           │
         │ ──────────────────────────>                           │
         │                           │  SELECT balance           │
         │                           │ ──────────────────────────>
         │                           │                           │
         │                           │  <────────────────────────
         │                           │  balance: 450             │
         │  <──────────────────────── │                           │
         │  {available: true}        │                           │
         │                           │                           │
         │  [Generate roadmap]       │                           │
         │                           │                           │
         │  POST /credits/consume    │                           │
         │  {userId, featureType}    │                           │
         │ ──────────────────────────>                           │
         │                           │  BEGIN TRANSACTION        │
         │                           │ ──────────────────────────>
         │                           │  UPDATE balance           │
         │                           │  INSERT transaction       │
         │                           │  COMMIT                   │
         │                           │ ──────────────────────────>
         │                           │                           │
         │  <──────────────────────── │                           │
         │  {success, newBalance}    │                           │
         │                           │                           │
```

### Stripe Webhook Flow

```
┌──────────────────┐        ┌──────────────────┐        ┌──────────────────┐
│     Stripe       │        │  Payment Service │        │    PostgreSQL    │
│                  │        │                  │        │                  │
└────────┬─────────┘        └────────┬─────────┘        └────────┬─────────┘
         │                           │                           │
         │  POST /webhooks/stripe    │                           │
         │  checkout.session.completed│                           │
         │ ──────────────────────────>                           │
         │                           │                           │
         │                           │  Verify signature         │
         │                           │                           │
         │                           │  Extract metadata         │
         │                           │  (userId, packageId)      │
         │                           │                           │
         │                           │  INSERT credit_account    │
         │                           │  (if not exists)          │
         │                           │ ──────────────────────────>
         │                           │                           │
         │                           │  UPDATE balance           │
         │                           │  INSERT transaction       │
         │                           │ ──────────────────────────>
         │                           │                           │
         │                           │  INSERT payment_history   │
         │                           │ ──────────────────────────>
         │                           │                           │
         │  <──────────────────────── │                           │
         │  { received: true }       │                           │
         │                           │                           │
```

## Security Considerations

### Authentication & Authorization

```typescript
// JWT validation for user endpoints
@UseGuards(JwtAuthGuard)
@Get('balance')
async getBalance(@CurrentUser() user: User) {}

// Service-to-service authentication
@UseGuards(ServiceAuthGuard)
@Post('consume')
async consumeCredits(@Body() dto: ConsumeCreditsDto) {}

// Stripe webhook signature verification
@Post('stripe')
async handleWebhook(
  @Headers('stripe-signature') signature: string,
  @RawBody() body: Buffer,
) {
  const event = this.stripeService.constructWebhookEvent(body, signature);
  // Process event...
}
```

### Data Protection

1. **PCI Compliance**: No card data stored locally - all handled by Stripe
2. **Webhook Security**: Verify Stripe signatures on all webhook calls
3. **Transaction Isolation**: Use database transactions for balance updates
4. **Audit Trail**: Log all credit transactions with timestamps and references
5. **Rate Limiting**: Protect purchase endpoints from abuse

### Input Validation

```typescript
// DTOs with validation
export class ConsumeCreditsDto {
  @IsString()
  @IsNotEmpty()
  userId: string;

  @IsEnum(FeatureType)
  featureType: FeatureType;

  @IsInt()
  @Min(1)
  @IsOptional()
  quantity?: number = 1;

  @IsString()
  @IsOptional()
  description?: string;
}
```

## Error Handling

### Custom Exceptions

```typescript
// Insufficient credits exception
export class InsufficientCreditsException extends HttpException {
  constructor(currentBalance: number, required: number) {
    super(
      {
        code: 7010,
        message: 'Insufficient credits',
        result: { currentBalance, requiredCredits: required },
      },
      HttpStatus.PAYMENT_REQUIRED,
    );
  }
}

// Payment failed exception
export class PaymentFailedException extends HttpException {
  constructor(reason: string) {
    super(
      {
        code: 7020,
        message: 'Payment failed',
        result: { reason },
      },
      HttpStatus.BAD_REQUEST,
    );
  }
}
```

### Global Exception Filter

```typescript
@Catch()
export class PaymentExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();

    if (exception instanceof HttpException) {
      const status = exception.getStatus();
      response.status(status).json(exception.getResponse());
    } else {
      // Log unexpected errors
      this.logger.error('Unexpected error', exception);
      response.status(500).json({
        code: 7099,
        message: 'Payment service error',
        result: null,
      });
    }
  }
}
```

## Performance Optimizations

### Database Indexes

```sql
-- Credit accounts lookup
CREATE INDEX idx_credit_accounts_user_id ON credit_accounts(user_id);

-- Transaction history queries
CREATE INDEX idx_transactions_account_id ON credit_transactions(account_id);
CREATE INDEX idx_transactions_created_at ON credit_transactions(created_at);
CREATE INDEX idx_transactions_type ON credit_transactions(type);

-- Subscription lookups
CREATE INDEX idx_subscriptions_user_id ON subscriptions(user_id);
CREATE INDEX idx_subscriptions_stripe_id ON subscriptions(stripe_subscription_id);
```

### Caching Strategy

```typescript
@Injectable()
export class CreditService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async getBalance(userId: string): Promise<CreditBalanceDto> {
    const cacheKey = `credit_balance:${userId}`;

    // Try cache first
    const cached = await this.cacheManager.get<CreditBalanceDto>(cacheKey);
    if (cached) return cached;

    // Fetch from database
    const balance = await this.calculateBalance(userId);

    // Cache for 1 minute
    await this.cacheManager.set(cacheKey, balance, 60000);

    return balance;
  }

  async invalidateCache(userId: string): Promise<void> {
    await this.cacheManager.del(`credit_balance:${userId}`);
  }
}
```

## Module Structure

```
payment-service/
├── src/
│   ├── main.ts
│   ├── app.module.ts
│   │
│   ├── config/
│   │   ├── database.config.ts
│   │   ├── stripe.config.ts
│   │   └── jwt.config.ts
│   │
│   ├── modules/
│   │   ├── credits/
│   │   │   ├── credits.module.ts
│   │   │   ├── credits.controller.ts
│   │   │   ├── credits.service.ts
│   │   │   ├── entities/
│   │   │   │   ├── credit-account.entity.ts
│   │   │   │   └── credit-transaction.entity.ts
│   │   │   ├── dto/
│   │   │   │   ├── check-credits.dto.ts
│   │   │   │   ├── consume-credits.dto.ts
│   │   │   │   └── credit-balance.dto.ts
│   │   │   └── repositories/
│   │   │       ├── credit-account.repository.ts
│   │   │       └── credit-transaction.repository.ts
│   │   │
│   │   ├── subscriptions/
│   │   │   ├── subscriptions.module.ts
│   │   │   ├── subscriptions.controller.ts
│   │   │   ├── subscriptions.service.ts
│   │   │   ├── entities/
│   │   │   │   └── subscription.entity.ts
│   │   │   └── dto/
│   │   │       ├── subscribe.dto.ts
│   │   │       └── subscription-response.dto.ts
│   │   │
│   │   ├── payments/
│   │   │   ├── payments.module.ts
│   │   │   ├── payments.controller.ts
│   │   │   ├── payments.service.ts
│   │   │   └── entities/
│   │   │       └── payment-history.entity.ts
│   │   │
│   │   ├── packages/
│   │   │   ├── packages.module.ts
│   │   │   ├── packages.controller.ts
│   │   │   └── packages.service.ts
│   │   │
│   │   ├── webhooks/
│   │   │   ├── webhooks.module.ts
│   │   │   ├── webhooks.controller.ts
│   │   │   └── webhooks.service.ts
│   │   │
│   │   └── stripe/
│   │       ├── stripe.module.ts
│   │       └── stripe.service.ts
│   │
│   ├── common/
│   │   ├── guards/
│   │   │   ├── jwt-auth.guard.ts
│   │   │   └── service-auth.guard.ts
│   │   ├── decorators/
│   │   │   └── current-user.decorator.ts
│   │   ├── filters/
│   │   │   └── payment-exception.filter.ts
│   │   ├── interceptors/
│   │   │   └── raw-body.interceptor.ts
│   │   └── enums/
│   │       ├── transaction-type.enum.ts
│   │       ├── feature-type.enum.ts
│   │       └── plan-type.enum.ts
│   │
│   └── shared/
│       └── constants/
│           ├── credit-costs.ts
│           └── package-configs.ts
│
├── test/
│   ├── credits.e2e-spec.ts
│   ├── subscriptions.e2e-spec.ts
│   └── webhooks.e2e-spec.ts
│
├── migrations/
│   ├── 001_create_credit_accounts.ts
│   ├── 002_create_transactions.ts
│   ├── 003_create_subscriptions.ts
│   └── 004_create_payment_history.ts
│
├── docker-compose.yml
├── Dockerfile
├── package.json
├── tsconfig.json
└── nest-cli.json
```

## Deployment Considerations

### Docker Configuration

```dockerfile
FROM node:18-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY dist ./dist

ENV NODE_ENV=production
ENV PORT=8087

EXPOSE 8087

CMD ["node", "dist/main.js"]
```

### Environment-Specific Settings

```typescript
// config/stripe.config.ts
export const stripeConfig = () => ({
  stripe: {
    secretKey: process.env.STRIPE_SECRET_KEY,
    webhookSecret: process.env.STRIPE_WEBHOOK_SECRET,
    apiVersion: '2023-10-16' as const,
  },
});
```

### Health Checks

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([() => this.db.pingCheck('database')]);
  }
}
```

## Keycloak Integration Workflows

### Complete Payment Flow with Authentication

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    Complete Credit Purchase Workflow                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  User        Frontend      Keycloak     Payment Service    Stripe     Database  │
│   │             │             │               │              │            │      │
│   │ 1. Login    │             │               │              │            │      │
│   │────────────>│             │               │              │            │      │
│   │             │ 2. Auth     │               │              │            │      │
│   │             │────────────>│               │              │            │      │
│   │             │             │               │              │            │      │
│   │             │ 3. JWT      │               │              │            │      │
│   │             │<────────────│               │              │            │      │
│   │             │             │               │              │            │      │
│   │ 4. View     │             │               │              │            │      │
│   │   Packages  │             │               │              │            │      │
│   │────────────>│             │               │              │            │      │
│   │             │             │               │              │            │      │
│   │             │ 5. GET /packages + JWT      │              │            │      │
│   │             │────────────────────────────>│              │            │      │
│   │             │             │               │              │            │      │
│   │             │             │ 6. Validate   │              │            │      │
│   │             │             │    JWT        │              │            │      │
│   │             │             │<──────────────│              │            │      │
│   │             │             │               │              │            │      │
│   │             │             │ 7. Token OK   │              │            │      │
│   │             │             │──────────────>│              │            │      │
│   │             │             │               │              │            │      │
│   │             │ 8. Package List             │              │            │      │
│   │             │<────────────────────────────│              │            │      │
│   │             │             │               │              │            │      │
│   │ 9. Select   │             │               │              │            │      │
│   │   Package   │             │               │              │            │      │
│   │────────────>│             │               │              │            │      │
│   │             │             │               │              │            │      │
│   │             │ 10. POST /purchase + JWT    │              │            │      │
│   │             │────────────────────────────>│              │            │      │
│   │             │             │               │              │            │      │
│   │             │             │               │ 11. Create   │            │      │
│   │             │             │               │    Checkout  │            │      │
│   │             │             │               │────────────->│            │      │
│   │             │             │               │              │            │      │
│   │             │             │               │ 12. Session  │            │      │
│   │             │             │               │    URL       │            │      │
│   │             │             │               │<─────────────│            │      │
│   │             │             │               │              │            │      │
│   │             │ 13. Checkout URL            │              │            │      │
│   │             │<────────────────────────────│              │            │      │
│   │             │             │               │              │            │      │
│   │ 14. Redirect to Stripe    │               │              │            │      │
│   │<────────────│             │               │              │            │      │
│   │             │             │               │              │            │      │
│   │ 15. Complete Payment      │               │              │            │      │
│   │─────────────────────────────────────────────────────────>│            │      │
│   │             │             │               │              │            │      │
│   │             │             │               │ 16. Webhook  │            │      │
│   │             │             │               │<─────────────│            │      │
│   │             │             │               │              │            │      │
│   │             │             │               │ 17. Add      │            │      │
│   │             │             │               │   Credits    │            │      │
│   │             │             │               │─────────────────────────->│      │
│   │             │             │               │              │            │      │
│   │ 18. Success Redirect      │               │              │            │      │
│   │<─────────────────────────────────────────────────────────│            │      │
│   │             │             │               │              │            │      │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Service-to-Service Credit Consumption

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│              Service-to-Service Credit Consumption Workflow                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  User        Frontend      Keycloak      AI Service    Payment Service  Database│
│   │             │             │              │               │             │     │
│   │ 1. Generate │             │              │               │             │     │
│   │   Roadmap   │             │              │               │             │     │
│   │────────────>│             │              │               │             │     │
│   │             │             │              │               │             │     │
│   │             │ 2. POST /generate + JWT    │               │             │     │
│   │             │───────────────────────────>│               │             │     │
│   │             │             │              │               │             │     │
│   │             │             │ 3. Validate  │               │             │     │
│   │             │             │    JWT       │               │             │     │
│   │             │             │<─────────────│               │             │     │
│   │             │             │              │               │             │     │
│   │             │             │ 4. OK + User │               │             │     │
│   │             │             │──────────────>               │             │     │
│   │             │             │              │               │             │     │
│   │             │             │              │ 5. Check      │             │     │
│   │             │             │              │   Credits     │             │     │
│   │             │             │              │ POST /check   │             │     │
│   │             │             │              │──────────────>│             │     │
│   │             │             │              │               │             │     │
│   │             │             │              │               │ 6. Query    │     │
│   │             │             │              │               │   Balance   │     │
│   │             │             │              │               │────────────>│     │
│   │             │             │              │               │             │     │
│   │             │             │              │               │ 7. Balance  │     │
│   │             │             │              │               │<────────────│     │
│   │             │             │              │               │             │     │
│   │             │             │              │ 8. Has        │             │     │
│   │             │             │              │   Credits     │             │     │
│   │             │             │              │<──────────────│             │     │
│   │             │             │              │               │             │     │
│   │             │             │              │ 9. Generate   │             │     │
│   │             │             │              │   Roadmap     │             │     │
│   │             │             │              │   (AI Logic)  │             │     │
│   │             │             │              │               │             │     │
│   │             │             │              │ 10. Consume   │             │     │
│   │             │             │              │   Credits     │             │     │
│   │             │             │              │ POST /consume │             │     │
│   │             │             │              │──────────────>│             │     │
│   │             │             │              │               │             │     │
│   │             │             │              │               │ 11. Deduct  │     │
│   │             │             │              │               │    Balance  │     │
│   │             │             │              │               │────────────>│     │
│   │             │             │              │               │             │     │
│   │             │             │              │               │ 12. Log     │     │
│   │             │             │              │               │   Transaction    │
│   │             │             │              │               │────────────>│     │
│   │             │             │              │               │             │     │
│   │             │             │              │ 13. Confirmed │             │     │
│   │             │             │              │<──────────────│             │     │
│   │             │             │              │               │             │     │
│   │             │ 14. Roadmap Result         │               │             │     │
│   │             │<───────────────────────────│               │             │     │
│   │             │             │              │               │             │     │
│   │ 15. Display │             │              │               │             │     │
│   │   Roadmap   │             │              │               │             │     │
│   │<────────────│             │              │               │             │     │
│   │             │             │              │               │             │     │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Keycloak Configuration for Payment Service

#### Realm Configuration

| Setting                | Value      | Description               |
| ---------------------- | ---------- | ------------------------- |
| Realm                  | `memap`    | Main application realm    |
| Access Token Lifespan  | 5 minutes  | Short-lived for security  |
| Refresh Token Lifespan | 30 minutes | Standard session duration |

#### Client Configuration

| Setting          | Value             | Description           |
| ---------------- | ----------------- | --------------------- |
| Client ID        | `payment-service` | Service identifier    |
| Client Protocol  | `openid-connect`  | OAuth2/OIDC protocol  |
| Access Type      | `confidential`    | Server-side service   |
| Standard Flow    | Disabled          | Not user-facing       |
| Service Accounts | Enabled           | For S2S communication |

#### Roles

| Role                  | Scope  | Permissions                |
| --------------------- | ------ | -------------------------- |
| `USER`                | Realm  | Basic credit operations    |
| `PREMIUM`             | Realm  | Extended features          |
| `ADMIN`               | Realm  | Full administrative access |
| `manage_subscription` | Client | Subscription management    |
| `view_all_accounts`   | Client | Admin view capabilities    |

#### Token Claims Mapping

```json
{
  "protocol": "openid-connect",
  "protocolMapper": "oidc-usermodel-attribute-mapper",
  "name": "user_id",
  "config": {
    "claim.name": "user_id",
    "user.attribute": "user_id",
    "access.token.claim": "true"
  }
}
```
