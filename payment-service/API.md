# Payment Service API Documentation

## Base URL

```
http://localhost:8087/payment
```

## Authentication

All endpoints (except webhooks) require JWT authentication via Keycloak. The Payment Service validates JWT tokens issued by Keycloak using the OAuth2 Resource Server pattern.

### Token Sources

Include the access token via one of the following methods:

**1. Authorization Header (Recommended):**

```
Authorization: Bearer {access_token}
```

**2. HTTP-only Cookie:**

```
Cookie: token={access_token}
```

**3. Bear Header (Swagger/Testing):**

```
Bear: {access_token}
```

### Keycloak Integration Flow

```
┌─────────────┐     ┌─────────────┐     ┌─────────────────┐     ┌─────────────┐
│   Client    │     │  Keycloak   │     │ Payment Service │     │   Stripe    │
└──────┬──────┘     └──────┬──────┘     └────────┬────────┘     └──────┬──────┘
       │                   │                     │                     │
       │ 1. Login Request  │                     │                     │
       │──────────────────>│                     │                     │
       │                   │                     │                     │
       │ 2. JWT Token      │                     │                     │
       │<──────────────────│                     │                     │
       │                   │                     │                     │
       │ 3. API Request + JWT Token              │                     │
       │────────────────────────────────────────>│                     │
       │                   │                     │                     │
       │                   │ 4. Validate Token   │                     │
       │                   │<────────────────────│                     │
       │                   │                     │                     │
       │                   │ 5. Token Valid +    │                     │
       │                   │    User Info/Roles  │                     │
       │                   │────────────────────>│                     │
       │                   │                     │                     │
       │                   │                     │ 6. Process Payment  │
       │                   │                     │────────────────────>│
       │                   │                     │                     │
       │                   │                     │ 7. Payment Result   │
       │                   │                     │<────────────────────│
       │                   │                     │                     │
       │ 8. Response       │                     │                     │
       │<────────────────────────────────────────│                     │
       │                   │                     │                     │
```

### Token Claims

The JWT token from Keycloak includes the following claims used by Payment Service:

| Claim                | Description       | Usage                          |
| -------------------- | ----------------- | ------------------------------ |
| `sub`                | Subject (User ID) | Identify user's credit account |
| `email`              | User's email      | Stripe customer creation       |
| `preferred_username` | Username          | Display and logging            |
| `realm_access.roles` | Realm roles       | Authorization checks           |
| `resource_access`    | Client roles      | Service-specific permissions   |

### Role-Based Access

| Role    | Permissions                                                         |
| ------- | ------------------------------------------------------------------- |
| `USER`  | View balance, history, purchase credits, manage subscription        |
| `ADMIN` | All user permissions + manage all accounts, refunds, manual credits |

---

## Endpoints

### Credit Account

#### Get Credit Balance

Get the current user's credit account information.

```http
GET /payment/credits/balance
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "accountId": "550e8400-e29b-41d4-a716-446655440000",
    "userId": "user_123",
    "balance": 450,
    "lifetimeEarned": 1000,
    "lifetimeSpent": 550,
    "subscription": {
      "plan": "PRO",
      "status": "ACTIVE",
      "monthlyCredits": 500,
      "creditsUsedThisMonth": 150,
      "periodEnd": "2026-02-28T23:59:59Z"
    }
  }
}
```

**Error Response (401 Unauthorized):**

```json
{
  "code": 7001,
  "message": "Authentication required",
  "result": null
}
```

---

#### Get Credit History

Retrieve credit transaction history with pagination.

```http
GET /payment/credits/history?page=0&size=20&type=USAGE
Authorization: Bearer {token}
```

**Query Parameters:**

| Parameter | Type   | Default | Description                                    |
| --------- | ------ | ------- | ---------------------------------------------- |
| page      | int    | 0       | Page number (0-indexed)                        |
| size      | int    | 20      | Items per page (max 100)                       |
| type      | string | -       | Filter by type: PURCHASE, USAGE, REFUND, BONUS |
| startDate | string | -       | Filter from date (ISO 8601)                    |
| endDate   | string | -       | Filter to date (ISO 8601)                      |

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "content": [
      {
        "id": "txn_001",
        "type": "USAGE",
        "amount": -10,
        "featureType": "AI_ROADMAP_GENERATION",
        "description": "Generated roadmap: Java Backend Developer",
        "createdAt": "2026-01-31T10:30:00Z"
      },
      {
        "id": "txn_002",
        "type": "PURCHASE",
        "amount": 500,
        "featureType": null,
        "description": "Purchased Basic package",
        "referenceId": "pi_3abc123",
        "createdAt": "2026-01-30T15:00:00Z"
      }
    ],
    "totalElements": 45,
    "totalPages": 3,
    "page": 0,
    "size": 20
  }
}
```

---

#### Check Feature Availability

Check if user has enough credits for a specific feature.

```http
POST /payment/credits/check
Authorization: Bearer {token}
Content-Type: application/json

{
  "featureType": "AI_ROADMAP_GENERATION",
  "quantity": 1
}
```

**Request Body:**

| Field       | Type   | Required | Description                 |
| ----------- | ------ | -------- | --------------------------- |
| featureType | string | Yes      | Feature to check            |
| quantity    | int    | No       | Number of uses (default: 1) |

**Feature Types:**

| Feature Type          | Credit Cost |
| --------------------- | ----------- |
| AI_ROADMAP_GENERATION | 10          |
| AI_NODE_SUMMARY       | 2           |
| AI_QUIZ_GENERATION    | 5           |
| AI_QA_CHAT            | 3           |
| STORAGE_100MB         | 5           |

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "available": true,
    "currentBalance": 450,
    "requiredCredits": 10,
    "remainingAfter": 440
  }
}
```

**Response (200 OK - Insufficient Credits):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "available": false,
    "currentBalance": 5,
    "requiredCredits": 10,
    "remainingAfter": -5,
    "suggestedPackage": {
      "name": "Starter",
      "credits": 100,
      "price": 4.99
    }
  }
}
```

---

#### Consume Credits

Deduct credits for feature usage (Internal API - called by other services).

```http
POST /payment/credits/consume
Authorization: Bearer {service_token}
Content-Type: application/json

{
  "userId": "user_123",
  "featureType": "AI_ROADMAP_GENERATION",
  "quantity": 1,
  "description": "Generated roadmap: Java Backend Developer",
  "referenceId": "roadmap_456"
}
```

**Request Body:**

| Field       | Type   | Required | Description                 |
| ----------- | ------ | -------- | --------------------------- |
| userId      | string | Yes      | User ID to charge           |
| featureType | string | Yes      | Feature being used          |
| quantity    | int    | No       | Number of uses (default: 1) |
| description | string | No       | Transaction description     |
| referenceId | string | No       | External reference ID       |

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Credits consumed successfully",
  "result": {
    "transactionId": "txn_789",
    "creditsConsumed": 10,
    "newBalance": 440
  }
}
```

**Error Response (402 Payment Required):**

```json
{
  "code": 7010,
  "message": "Insufficient credits",
  "result": {
    "currentBalance": 5,
    "requiredCredits": 10
  }
}
```

---

### Credit Packages

#### Get Available Packages

List all credit packages available for purchase.

```http
GET /payment/packages
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": [
    {
      "id": "pkg_starter",
      "name": "Starter",
      "credits": 100,
      "price": 4.99,
      "currency": "USD",
      "bonus": 0,
      "popular": false
    },
    {
      "id": "pkg_basic",
      "name": "Basic",
      "credits": 500,
      "bonusCredits": 50,
      "totalCredits": 550,
      "price": 19.99,
      "currency": "USD",
      "bonus": 10,
      "popular": true
    },
    {
      "id": "pkg_pro",
      "name": "Pro",
      "credits": 1500,
      "bonusCredits": 300,
      "totalCredits": 1800,
      "price": 49.99,
      "currency": "USD",
      "bonus": 20,
      "popular": false
    },
    {
      "id": "pkg_enterprise",
      "name": "Enterprise",
      "credits": 5000,
      "bonusCredits": 1500,
      "totalCredits": 6500,
      "price": 149.99,
      "currency": "USD",
      "bonus": 30,
      "popular": false
    }
  ]
}
```

---

#### Purchase Credits

Create a Stripe checkout session for credit purchase.

```http
POST /payment/packages/purchase
Authorization: Bearer {token}
Content-Type: application/json

{
  "packageId": "pkg_basic",
  "successUrl": "https://memap.com/payment/success",
  "cancelUrl": "https://memap.com/payment/cancel"
}
```

**Request Body:**

| Field      | Type   | Required | Description             |
| ---------- | ------ | -------- | ----------------------- |
| packageId  | string | Yes      | Package ID to purchase  |
| successUrl | string | Yes      | Redirect URL on success |
| cancelUrl  | string | Yes      | Redirect URL on cancel  |

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Checkout session created",
  "result": {
    "sessionId": "cs_test_abc123",
    "checkoutUrl": "https://checkout.stripe.com/pay/cs_test_abc123",
    "expiresAt": "2026-01-31T12:30:00Z"
  }
}
```

---

### Subscriptions

#### Get Subscription Plans

List available subscription plans.

```http
GET /payment/subscriptions/plans
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": [
    {
      "id": "plan_free",
      "name": "Free",
      "price": 0,
      "currency": "USD",
      "interval": "month",
      "features": {
        "monthlyCredits": 50,
        "storageGB": 0.1,
        "aiGenerationsPerMonth": 3,
        "support": "community"
      }
    },
    {
      "id": "plan_pro",
      "name": "Pro",
      "price": 9.99,
      "currency": "USD",
      "interval": "month",
      "stripePriceId": "price_pro_monthly",
      "features": {
        "monthlyCredits": 500,
        "storageGB": 1,
        "aiGenerationsPerMonth": 50,
        "support": "priority",
        "analytics": true
      },
      "popular": true
    },
    {
      "id": "plan_enterprise",
      "name": "Enterprise",
      "price": 29.99,
      "currency": "USD",
      "interval": "month",
      "stripePriceId": "price_enterprise_monthly",
      "features": {
        "monthlyCredits": 2000,
        "storageGB": 10,
        "aiGenerationsPerMonth": -1,
        "support": "dedicated",
        "analytics": true,
        "customIntegrations": true
      }
    }
  ]
}
```

---

#### Get Current Subscription

Get user's current subscription details.

```http
GET /payment/subscriptions/current
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "id": "sub_123",
    "plan": {
      "id": "plan_pro",
      "name": "Pro",
      "price": 9.99
    },
    "status": "ACTIVE",
    "currentPeriodStart": "2026-01-01T00:00:00Z",
    "currentPeriodEnd": "2026-02-01T00:00:00Z",
    "cancelAtPeriodEnd": false,
    "usage": {
      "creditsUsed": 150,
      "creditsRemaining": 350,
      "storageUsedMB": 256,
      "storageLimitMB": 1024,
      "aiGenerationsUsed": 12,
      "aiGenerationsLimit": 50
    }
  }
}
```

---

#### Subscribe to Plan

Create a subscription checkout session.

```http
POST /payment/subscriptions/subscribe
Authorization: Bearer {token}
Content-Type: application/json

{
  "planId": "plan_pro",
  "successUrl": "https://memap.com/subscription/success",
  "cancelUrl": "https://memap.com/subscription/cancel"
}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Subscription checkout created",
  "result": {
    "sessionId": "cs_test_sub123",
    "checkoutUrl": "https://checkout.stripe.com/pay/cs_test_sub123"
  }
}
```

---

#### Change Subscription Plan

Upgrade or downgrade subscription.

```http
PUT /payment/subscriptions/change
Authorization: Bearer {token}
Content-Type: application/json

{
  "newPlanId": "plan_enterprise"
}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Subscription updated successfully",
  "result": {
    "previousPlan": "Pro",
    "newPlan": "Enterprise",
    "effectiveDate": "2026-02-01T00:00:00Z",
    "proratedAmount": 15.5
  }
}
```

---

#### Cancel Subscription

Cancel subscription at period end.

```http
POST /payment/subscriptions/cancel
Authorization: Bearer {token}
Content-Type: application/json

{
  "reason": "Too expensive",
  "feedback": "Would use again if price was lower"
}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Subscription will be cancelled at period end",
  "result": {
    "cancellationDate": "2026-02-01T00:00:00Z",
    "accessUntil": "2026-02-01T00:00:00Z"
  }
}
```

---

### Payment History

#### Get Payment History

Retrieve user's payment history.

```http
GET /payment/history?page=0&size=10
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "content": [
      {
        "id": "pay_001",
        "stripePaymentId": "pi_3abc123",
        "amount": 19.99,
        "currency": "USD",
        "status": "COMPLETED",
        "description": "Basic credit package (550 credits)",
        "paymentMethod": "card",
        "cardLast4": "4242",
        "cardBrand": "visa",
        "createdAt": "2026-01-30T15:00:00Z",
        "receiptUrl": "https://pay.stripe.com/receipts/..."
      },
      {
        "id": "pay_002",
        "stripePaymentId": "in_xyz789",
        "amount": 9.99,
        "currency": "USD",
        "status": "COMPLETED",
        "description": "Pro subscription - January 2026",
        "paymentMethod": "card",
        "cardLast4": "4242",
        "cardBrand": "visa",
        "createdAt": "2026-01-01T00:00:00Z",
        "invoiceUrl": "https://pay.stripe.com/invoice/..."
      }
    ],
    "totalElements": 12,
    "totalPages": 2,
    "page": 0,
    "size": 10
  }
}
```

---

#### Get Invoice

Get specific invoice/receipt details.

```http
GET /payment/invoices/{invoiceId}
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "id": "inv_001",
    "number": "INV-2026-0001",
    "status": "PAID",
    "amount": 19.99,
    "currency": "USD",
    "items": [
      {
        "description": "Basic Credit Package",
        "quantity": 1,
        "unitPrice": 19.99,
        "amount": 19.99
      }
    ],
    "subtotal": 19.99,
    "tax": 0,
    "total": 19.99,
    "paidAt": "2026-01-30T15:00:00Z",
    "receiptUrl": "https://pay.stripe.com/receipts/...",
    "pdfUrl": "https://pay.stripe.com/invoice/pdf/..."
  }
}
```

---

### Stripe Webhooks

#### Handle Stripe Webhook

Endpoint for Stripe webhook events. **No authentication required** - verified by Stripe signature.

```http
POST /payment/webhooks/stripe
Content-Type: application/json
Stripe-Signature: t=xxx,v1=xxx

{raw event body}
```

**Handled Events:**

| Event                         | Action                             |
| ----------------------------- | ---------------------------------- |
| checkout.session.completed    | Credit credits to user account     |
| payment_intent.succeeded      | Log successful payment             |
| payment_intent.payment_failed | Log failed payment, notify user    |
| customer.subscription.created | Activate subscription              |
| customer.subscription.updated | Update subscription status         |
| customer.subscription.deleted | Cancel subscription                |
| invoice.payment_succeeded     | Log subscription payment           |
| invoice.payment_failed        | Handle failed subscription payment |

**Response (200 OK):**

```json
{
  "received": true
}
```

---

### Storage Management

#### Get Storage Usage

Get user's current storage usage and limits.

```http
GET /payment/storage/usage
Authorization: Bearer {token}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Success",
  "result": {
    "usedBytes": 268435456,
    "usedMB": 256,
    "limitBytes": 1073741824,
    "limitMB": 1024,
    "percentUsed": 25,
    "canUpgrade": true,
    "upgradeOptions": [
      {
        "additionalMB": 500,
        "creditsCost": 25
      },
      {
        "additionalMB": 1000,
        "creditsCost": 45
      }
    ]
  }
}
```

---

#### Purchase Additional Storage

Purchase extra storage using credits.

```http
POST /payment/storage/purchase
Authorization: Bearer {token}
Content-Type: application/json

{
  "additionalMB": 500
}
```

**Response (200 OK):**

```json
{
  "code": 0,
  "message": "Storage upgraded successfully",
  "result": {
    "creditsCharged": 25,
    "newStorageLimitMB": 1524,
    "newBalance": 425
  }
}
```

---

## Error Codes

| Code | HTTP Status | Description                   |
| ---- | ----------- | ----------------------------- |
| 7001 | 401         | Authentication required       |
| 7002 | 403         | Access denied                 |
| 7010 | 402         | Insufficient credits          |
| 7011 | 400         | Invalid package ID            |
| 7012 | 400         | Invalid plan ID               |
| 7020 | 400         | Payment failed                |
| 7021 | 400         | Invalid payment method        |
| 7022 | 400         | Card declined                 |
| 7030 | 400         | Subscription not found        |
| 7031 | 400         | Already subscribed            |
| 7032 | 400         | Cannot downgrade during trial |
| 7040 | 400         | Invalid webhook signature     |
| 7050 | 400         | Storage limit exceeded        |
| 7099 | 500         | Payment service error         |

---

## Data Models

### CreditAccount

```typescript
interface CreditAccount {
  accountId: string;
  userId: string;
  balance: number;
  lifetimeEarned: number;
  lifetimeSpent: number;
  createdAt: string;
  updatedAt: string;
}
```

### CreditTransaction

```typescript
interface CreditTransaction {
  id: string;
  type: 'PURCHASE' | 'USAGE' | 'REFUND' | 'BONUS';
  amount: number;
  featureType?: string;
  description?: string;
  referenceId?: string;
  createdAt: string;
}
```

### Subscription

```typescript
interface Subscription {
  id: string;
  plan: SubscriptionPlan;
  status: 'ACTIVE' | 'CANCELLED' | 'PAST_DUE' | 'TRIALING';
  currentPeriodStart: string;
  currentPeriodEnd: string;
  cancelAtPeriodEnd: boolean;
  usage: SubscriptionUsage;
}
```

### SubscriptionPlan

```typescript
interface SubscriptionPlan {
  id: string;
  name: string;
  price: number;
  currency: string;
  interval: 'month' | 'year';
  features: PlanFeatures;
}
```

### PaymentRecord

```typescript
interface PaymentRecord {
  id: string;
  stripePaymentId: string;
  amount: number;
  currency: string;
  status: 'PENDING' | 'COMPLETED' | 'FAILED' | 'REFUNDED';
  description: string;
  paymentMethod: string;
  createdAt: string;
  receiptUrl?: string;
}
```

---

## Rate Limiting

| Endpoint Category  | Rate Limit           |
| ------------------ | -------------------- |
| Read endpoints     | 100 requests/minute  |
| Purchase endpoints | 10 requests/minute   |
| Webhook endpoint   | 1000 requests/minute |

---

## Examples

### Complete Credit Purchase Flow

**1. Get available packages:**

```bash
curl -X GET http://localhost:8087/payment/packages
```

**2. Create checkout session:**

```bash
curl -X POST http://localhost:8087/payment/packages/purchase \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "packageId": "pkg_basic",
    "successUrl": "https://memap.com/payment/success",
    "cancelUrl": "https://memap.com/payment/cancel"
  }'
```

**3. Redirect user to checkoutUrl**

**4. After successful payment, check balance:**

```bash
curl -X GET http://localhost:8087/payment/credits/balance \
  -H "Authorization: Bearer {token}"
```

### Check and Consume Credits (Service-to-Service)

**1. Check availability:**

```bash
curl -X POST http://localhost:8087/payment/credits/check \
  -H "Authorization: Bearer {token}" \
  -H "Content-Type: application/json" \
  -d '{
    "featureType": "AI_ROADMAP_GENERATION",
    "quantity": 1
  }'
```

**2. If available, consume credits:**

```bash
curl -X POST http://localhost:8087/payment/credits/consume \
  -H "Authorization: Bearer {service_token}" \
  -H "Content-Type: application/json" \
  -d '{
    "userId": "user_123",
    "featureType": "AI_ROADMAP_GENERATION",
    "quantity": 1,
    "description": "Generated roadmap: Java Backend"
  }'
```
