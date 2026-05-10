# Stripe Database Design

## Overview

This document defines the MongoDB collections and data models for Stripe integration within the payment service. The design supports customer management, payment processing, webhook handling, and subscription lifecycle tracking.

## Entity Relationship Diagram (ERD)

### High-Level Relationships

```
┌─────────────────────┐
│     tbl_plans       │
│                     │
│ PK _id              │
│ UK plan_code        │────────────────────────────────┐
│ UK stripe_price_id  │                                │
└────────┬────────────┘                                │
         │ 1                                           │
         │                                             │
         │ N                                           │
         │                                             │
         ▼                                             │
┌─────────────────────┐                                │
│ tbl_stripe_customers│                                │
│                     │                                │
│ PK _id              │                                │
│ UK user_id ─────────┼───────┐                        │
│ UK stripe_customer_id       │                        │
└────────┬────────────┘       │                        │
         │                    │                        │
         │ 1                  │ 1                      │
         │                    │                        │
         ▼                    ▼                        │
┌─────────────────────┐  ┌──────────────────────────┐  │
│ tbl_payment_history │  │ tbl_stripe_subscriptions │  │
│                     │  │                          │  │
│ PK _id              │  │ PK _id                   │  │
│ FK user_id          │  │ FK user_id               │◄─┘
│ UK stripe_payment_intent_id  │  │ UK stripe_subscription_id
│                     │  │ FK stripe_customer_id
│ FK subscription_id ─┼──┼──┤ FK plan_id
└─────────────────────┘  └──────────┬───────────────┘
                                    │
                                    │ 1
                                    │
                                    ▼
                    ┌──────────────────────────┐
                    │   tbl_stripe_invoices    │
                    │                          │
                    │ PK _id                   │
                    │ UK stripe_invoice_id     │
                    │ FK stripe_subscription_id│
                    │ FK stripe_customer_id    │
                    │ FK stripe_payment_intent_id
                    └──────────────────────────┘


┌──────────────────────────┐
│ tbl_stripe_payment_methods│
│                          │
│ PK _id                   │
│ FK user_id               │
│ UK stripe_payment_method_id
│ FK stripe_customer_id    │
└──────────────────────────┘


┌──────────────────────────┐     ┌──────────────────────────┐
│ tbl_processed_stripe_events│     │ tbl_stripe_webhook_logs  │
│                          │     │                          │
│ PK _id                   │     │ PK _id                   │
│ UK event_id ─────────────┼─────┤ UK event_id              │
│ TTL processed_at (30d)   │     │ TTL received_at (90d)    │
└──────────────────────────┘     └──────────────────────────┘
```

**Business Rule: One Active Subscription Per User**

```
At any given moment, a user can only have ONE active subscription.

Valid transitions:
  (no subscription) ──subscribe──▶ ACTIVE (plan A)
  ACTIVE (plan A) ──cancel──▶ CANCELED/EXPIRED
  ACTIVE (plan A) ──switch──▶ ACTIVE (plan B)  [cancels A first]
  EXPIRED (plan A) ──resubscribe──▶ ACTIVE (plan A or B)

Enforced by partial unique index:
  { user_id: 1 } where status IN ['ACTIVE', 'TRIALING']
```

### Detailed Table ERD with Fields

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              tbl_plans                                           │
│  (Plan catalog — defines available subscription tiers and pricing)              │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ plan_code                │ String   │ Unique human-readable code (e.g., "FREE",  │
│                          │          │   "BASIC", "PRO", "ENTERPRISE")            │
│ name                     │ String   │ Display name shown to users                │
│ description              │ String   │ Marketing description of plan features     │
│ stripe_product_id        │ String   │ Stripe Product ID (prod_xxx format)        │
│ stripe_price_id          │ String   │ Stripe Price ID (price_xxx format)         │
│ amount                   │ Long     │ Price per billing period in cents          │
│                          │          │   (0 for FREE plan)                        │
│ currency                 │ String   │ ISO 4217 currency code (usd, vnd)          │
│ interval                 │ Enum     │ Billing frequency (day, week, month, year) │
│ interval_count           │ Integer  │ Multiplier (e.g., 3 = every 3 months)      │
│ trial_period_days        │ Integer  │ Free trial length in days (0 = no trial)   │
│ credits_per_period       │ Long     │ Credits included with this plan per period │
│ features                 │ Array    │ List of feature flags this plan unlocks    │
│                          │          │   (e.g., ["advanced_analytics", "api_access"])│
│ max_projects             │ Integer  │ Maximum projects allowed (-1 = unlimited)  │
│ max_team_members         │ Integer  │ Maximum team members (-1 = unlimited)      │
│ priority                 │ Integer  │ Sort order for display (lower = higher tier)│
│ is_active                │ Boolean  │ Whether this plan is available for purchase│
│ is_visible               │ Boolean  │ Whether this plan shows on pricing page    │
│                          │          │   (false = legacy/hidden plan)             │
│ metadata.stripe_recur    │ Boolean  │ True = recurring subscription              │
│ metadata.tax_code        │ String   │ Stripe tax code for this product           │
│ metadata.stripe_metadata│ Object   │ Additional Stripe product metadata         │
│ created_at               │ ISODate  │ When plan was created                      │
│ updated_at               │ ISODate  │ When plan details were last modified       │
│ deprecated_at            │ ISODate  │ When plan was deprecated (null = active)   │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(plan_code), UNIQUE(stripe_price_id), (is_active, priority),      │
│          (is_visible, is_active)                                                 │
│ Constraints: Cannot subscribe to deprecated/inactive plans (enforced in code)    │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                           tbl_stripe_customers                                   │
│  (Maps internal users to Stripe customers for payment processing)               │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ user_id                  │ String   │ Internal user UUID from Keycloak           │
│ stripe_customer_id       │ String   │ Stripe customer ID (cus_xxx format)        │
│ email                    │ String   │ User email address synced to Stripe        │
│ created_at               │ ISODate  │ When this mapping was first created        │
│ updated_at               │ ISODate  │ Last time customer data was synced         │
│ is_active                │ Boolean  │ Soft delete flag (true = active user)      │
│ metadata.display_name    │ String   │ User's display name for Stripe receipts    │
│ metadata.phone           │ String   │ User phone number for billing              │
│ metadata.address.*       │ Object   │ Billing address for tax/invoice purposes   │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(user_id), UNIQUE(stripe_customer_id), email, created_at DESC     │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                           tbl_payment_history                                    │
│  (Append-only log of all payment transactions through Stripe)                   │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ user_id                  │ String   │ User who made this payment                 │
│ stripe_payment_intent_id │ String   │ Stripe PaymentIntent ID (pi_xxx format)    │
│ stripe_charge_id         │ String   │ Stripe Charge ID (ch_xxx format)           │
│ amount                   │ Long     │ Payment amount in cents (e.g., 999 = $9.99)│
│ currency                 │ String   │ ISO 4217 currency code (usd, eur, vnd)     │
│ status                   │ Enum     │ Payment lifecycle state                    │
│                          │          │   PENDING: Created, awaiting confirmation  │
│                          │          │   SUCCEEDED: Payment completed successfully│
│                          │          │   FAILED: Payment declined or errored      │
│                          │          │   REFUNDED: Money returned to customer     │
│                          │          │   CANCELLED: PaymentIntent cancelled       │
│ payment_method.type      │ String   │ Payment method type (card, bank_transfer)  │
│ payment_method.last4     │ String   │ Last 4 digits of card for display          │
│ payment_method.brand     │ String   │ Card network (visa, mastercard, amex)      │
│ payment_method.exp_month │ Integer  │ Card expiration month (1-12)               │
│ payment_method.exp_year  │ Integer  │ Card expiration year                       │
│ description              │ String   │ Human-readable payment description         │
│ subscription_id          │ ObjectId │ FK to subscription if this is a renewal    │
│ credit_amount            │ Long     │ Credits granted from this payment (if any) │
│ metadata.stripe_receipt_url │ String│ URL to Stripe receipt for customer         │
│ metadata.stripe_invoice_id │ String │ Related Stripe invoice ID                  │
│ metadata.failure_code    │ String   │ Stripe error code if payment failed        │
│ metadata.failure_message │ String   │ Human-readable failure reason              │
│ metadata.refund_amount   │ Long     │ Amount refunded in cents                   │
│ metadata.refund_reason   │ String   │ Why this payment was refunded              │
│ created_at               │ ISODate  │ When payment was initiated                 │
│ updated_at               │ ISODate  │ When payment status last changed           │
│ processed_at             │ ISODate  │ When webhook confirmed this payment        │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(stripe_payment_intent_id), UNIQUE(stripe_charge_id),             │
│          (user_id, created_at DESC), (status, created_at DESC), subscription_id  │
│ Constraints: Append-only, amount/currency immutable after creation               │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                        tbl_processed_stripe_events                               │
│  (Webhook event deduplication and processing log with TTL)                      │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ event_id                 │ String   │ Stripe event ID (evt_xxx format)           │
│ event_type               │ String   │ Stripe event type (e.g., payment_intent.   │
│                          │          │   succeeded, invoice.payment_failed)       │
│ api_version              │ String   │ Stripe API version when event was created  │
│ created_at               │ ISODate  │ When Stripe generated this event           │
│ processed_at             │ ISODate  │ When our system finished processing it     │
│ status                   │ Enum     │ Processing state                           │
│                          │          │   PENDING: Received, not yet processed     │
│                          │          │   PROCESSING: Currently being handled      │
│                          │          │   COMPLETED: Successfully processed        │
│                          │          │   FAILED: Processing errored               │
│                          │          │   SKIPPED: Intentionally ignored (stub)    │
│ payload_hash             │ String   │ SHA-256 hash for payload integrity check   │
│ retry_count              │ Integer  │ How many times we tried to process this    │
│ error_message            │ String   │ Last error if processing failed            │
│ handler_name             │ String   │ Which handler class processed this event   │
│ idempotency_key          │ String   │ Generated key to prevent duplicate work    │
│ metadata.related_*       │ String   │ IDs of related Stripe objects              │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(event_id), (event_type, created_at DESC),                        │
│          (status, processed_at DESC), idempotency_key, payload_hash              │
│ TTL: Auto-deletes 30 days after processed_at                                     │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                      tbl_stripe_payment_methods                                  │
│  (Cached payment methods for quick checkout and recurring billing)              │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ user_id                  │ String   │ User who owns this payment method          │
│ stripe_customer_id       │ String   │ Stripe customer this method belongs to     │
│ stripe_payment_method_id │ String   │ Stripe PaymentMethod ID (pm_xxx format)    │
│ type                     │ String   │ Payment method type (card, sepa_debit)     │
│ card.brand               │ String   │ Card network (visa, mastercard, amex)      │
│ card.last4               │ String   │ Last 4 digits for UI display               │
│ card.exp_month           │ Integer  │ Card expiration month                      │
│ card.exp_year            │ Integer  │ Card expiration year                       │
│ card.fingerprint         │ String   │ Stripe's unique card fingerprint           │
│                          │          │   (detects same card across customers)     │
│ card.country             │ String   │ Country where card was issued              │
│ card.funding             │ String   │ Card funding type (credit, debit, prepaid) │
│ billing_details.*        │ Object   │ Customer billing address for this card     │
│ is_default               │ Boolean  │ True = this is the user's default card     │
│ is_active                │ Boolean  │ Soft delete (false = card removed by user) │
│ created_at               │ ISODate  │ When this payment method was saved         │
│ updated_at               │ ISODate  │ When billing details were last updated     │
│ last_used_at             │ ISODate  │ When this card was last charged            │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(stripe_payment_method_id), (user_id, is_active),                 │
│          (stripe_customer_id, is_default), card.fingerprint                      │
│ Constraints: One default card per user (partial unique on is_active=true)        │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                       tbl_stripe_subscriptions                                   │
│  (Detailed Stripe subscription records synced from webhook events)              │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ user_id                  │ String   │ User who owns this subscription            │
│ stripe_subscription_id   │ String   │ Stripe Subscription ID (sub_xxx format)    │
│ stripe_customer_id       │ String   │ Stripe customer ID                         │
│ plan_id                  │ String   │ Internal plan ID (maps to our plan catalog)│
│ stripe_price_id          │ String   │ Stripe Price ID (price_xxx format)         │
│ status                   │ Enum     │ Subscription lifecycle state               │
│                          │          │   TRIALING: In free trial period           │
│                          │          │   ACTIVE: Paid and in good standing        │
│                          │          │   PAST_DUE: Payment failed, grace period   │
│                          │          │   CANCELED: User canceled, may still active│
│                          │          │   EXPIRED: Fully ended, no access          │
│                          │          │   UNPAID: Never paid, access denied        │
│ current_period_start     │ ISODate  │ Start of current billing period            │
│ current_period_end       │ ISODate  │ End of current billing period (next charge)│
│ cancel_at_period_end     │ Boolean  │ True = user canceled, access until period_end│
│ canceled_at              │ ISODate  │ When user initiated cancellation           │
│ ended_at                 │ ISODate  │ When subscription fully terminated         │
│ trial_start              │ ISODate  │ When free trial began                      │
│ trial_end                │ ISODate  │ When free trial ends (first charge date)   │
│ quantity                 │ Integer  │ Number of seats/licenses purchased         │
│ amount                   │ Long     │ Price per billing period in cents          │
│ currency                 │ String   │ ISO 4217 currency code                     │
│ interval                 │ Enum     │ Billing frequency (day, week, month, year) │
│ interval_count           │ Integer  │ Multiplier (e.g., 3 = every 3 months)      │
│ payment_method_id        │ String   │ Stripe PaymentMethod used for this sub     │
│ latest_invoice_id        │ String   │ Most recent Stripe Invoice ID              │
│ latest_payment_intent_id │ String   │ Most recent PaymentIntent for this sub     │
│ metadata.stripe_invoice_url │ String│ URL to view invoice in Stripe              │
│ metadata.discount_code   │ String   │ Promo/discount code applied                │
│ metadata.discount_amount │ Long     │ Discount amount in cents                   │
│ metadata.tax_amount      │ Long     │ Tax amount in cents                        │
│ metadata.subscription_items│ Array   │ Multi-price subscription line items        │
│ created_at               │ ISODate  │ When subscription was first created        │
│ updated_at               │ ISODate  │ When subscription was last modified        │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(stripe_subscription_id), (user_id, status),                      │
│          (status, current_period_end), (cancel_at_period_end, current_period_end)│
│ Constraints: One active/trialing subscription per user (partial unique)          │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                         tbl_stripe_invoices                                      │
│  (Invoice records for billing history and reconciliation)                       │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ user_id                  │ String   │ User this invoice belongs to               │
│ stripe_invoice_id        │ String   │ Stripe Invoice ID (in_xxx format)          │
│ stripe_subscription_id   │ String   │ Related subscription (null for one-off)    │
│ stripe_payment_intent_id │ String   │ PaymentIntent for this invoice             │
│ stripe_customer_id       │ String   │ Stripe customer ID                         │
│ amount_due               │ Long     │ Total amount the customer owes (cents)     │
│ amount_paid              │ Long     │ Amount already paid (cents)                │
│ amount_remaining         │ Long     │ Still unpaid amount (amount_due - paid)    │
│ currency                 │ String   │ ISO 4217 currency code                     │
│ status                   │ Enum     │ Invoice state                              │
│                          │          │   DRAFT: Not yet finalized, editable       │
│                          │          │   OPEN: Finalized, awaiting payment        │
│                          │          │   PAID: Fully paid                         │
│                          │          │   UNCOLLECTIBLE: Written off as bad debt   │
│                          │          │   VOID: Canceled before payment            │
│ number                   │ String   │ Human-readable invoice number (e.g., INV-001)│
│ description              │ String   │ Invoice description for customer           │
│ period_start             │ ISODate  │ Start of billing period this invoice covers│
│ period_end               │ ISODate  │ End of billing period                      │
│ due_date                 │ ISODate  │ Payment deadline (for net terms)           │
│ paid_at                  │ ISODate  │ When invoice was fully paid                │
│ finalized_at             │ ISODate  │ When invoice became immutable              │
│ lines[].description      │ String   │ Line item description                      │
│ lines[].amount           │ Long     │ Line item amount in cents                  │
│ lines[].quantity         │ Integer  │ Line item quantity                         │
│ lines[].plan_id          │ String   │ Internal plan ID for this line item        │
│ lines[].period_start     │ ISODate  │ Billing period for this line item          │
│ lines[].period_end       │ ISODate  │ Billing period end for this line item      │
│ tax                      │ Long     │ Tax amount in cents                        │
│ discount                 │ Long     │ Discount amount in cents                   │
│ subtotal                 │ Long     │ Sum of line items before tax/discount      │
│ total                    │ Long     │ Final amount (subtotal + tax - discount)   │
│ pdf_url                  │ String   │ URL to download PDF invoice                │
│ hosted_invoice_url       │ String   │ URL for customer to view/pay invoice online│
│ created_at               │ ISODate  │ When invoice record was created locally    │
│ updated_at               │ ISODate  │ When invoice was last updated              │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(stripe_invoice_id), (user_id, created_at DESC),                  │
│          (stripe_subscription_id), (status, due_date), amount_remaining          │
└─────────────────────────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────────────────────────┐
│                       tbl_stripe_webhook_logs                                    │
│  (Audit log for all incoming webhook events - separate from processed events)   │
├──────────────────────────┬──────────┬───────────────────────────────────────────┤
│ Field                    │ Type     │ Meaning                                    │
├──────────────────────────┼──────────┼───────────────────────────────────────────┤
│ _id                      │ ObjectId │ MongoDB primary key                        │
│ event_id                 │ String   │ Stripe event ID from webhook payload       │
│ event_type               │ String   │ Type of Stripe event received              │
│ raw_payload              │ String   │ Full JSON payload from Stripe (for audit)  │
│ signature                │ String   │ Stripe-Signature header value              │
│                          │          │   (used to verify webhook authenticity)    │
│ ip_address               │ String   │ Source IP that sent the webhook            │
│ received_at              │ ISODate  │ When our server received the webhook       │
│ verified                 │ Boolean  │ True = Stripe signature validation passed  │
│ processed                │ Boolean  │ True = event was queued for processing     │
│ processing_status        │ Enum     │ Queue processing state                     │
│                          │          │   QUEUED: Waiting to be picked up          │
│                          │          │   PROCESSING: Being handled                │
│                          │          │   COMPLETED: Finished successfully         │
│                          │          │   FAILED: Error during processing          │
│ error_details.code       │ String   │ Application error code                     │
│ error_details.message    │ String   │ Human-readable error description           │
│ error_details.stack_trace│ String   │ Full stack trace for debugging             │
│ processing_duration_ms   │ Long     │ How long processing took (milliseconds)    │
│ handler_version          │ String   │ Version of handler that processed this     │
│                          │          │   (useful for debugging deployments)       │
├──────────────────────────┴──────────┴───────────────────────────────────────────┤
│ Indexes: UNIQUE(event_id), (event_type, received_at DESC),                       │
│          (verified, received_at DESC), (processing_status, received_at DESC)     │
│ TTL: Auto-deletes 90 days after received_at                                      │
└─────────────────────────────────────────────────────────────────────────────────┘
```

### Relationship Cardinality Summary

```
┌──────────────────────┐
│ tbl_stripe_customers │
└──────────┬───────────┘
           │
           │ 1:1
           │
    ┌──────┴──────┐
    │             │
    │ 1:N         │ 1:N
    ▼             ▼
┌─────────────┐ ┌────────────────────────┐
│tbl_payment_ │ │tbl_stripe_subscriptions│
│ history     │ └───────────┬────────────┘
└─────────────┘             │
                            │ 1:N
                            ▼
                  ┌─────────────────────┐
                  │ tbl_stripe_invoices │
                  └─────────────────────┘

┌───────────────────────────────┐
│ tbl_stripe_payment_methods    │
│ (1:N with tbl_stripe_customers)│
└───────────────────────────────┘

┌───────────────────────────────┐     ┌──────────────────────────┐
│ tbl_processed_stripe_events   │     │ tbl_stripe_webhook_logs  │
│ (TTL: 30 days)                │     │ (TTL: 90 days)           │
│ Deduplication                 │────▶│ Audit trail              │
└───────────────────────────────┘     └──────────────────────────┘
```

## Collections Schema

### 1. `tbl_stripe_customers`

Maps internal users to Stripe customers for payment processing.

```javascript
{
  _id: ObjectId,
  user_id: String,           // Internal user ID (Keycloak UUID)
  stripe_customer_id: String, // Stripe customer ID (cus_xxx)
  email: String,             // User email synced to Stripe
  created_at: ISODate,       // Record creation timestamp
  updated_at: ISODate,       // Last sync/update timestamp
  is_active: Boolean,        // Soft delete flag (default: true)
  metadata: {                // Additional Stripe metadata
    display_name: String,
    phone: String,
    address: {
      line1: String,
      line2: String,
      city: String,
      state: String,
      postal_code: String,
      country: String
    }
  }
}
```

**Indexes:**
```javascript
{ user_id: 1 },                          // Unique lookup by internal user
{ stripe_customer_id: 1 },               // Unique lookup by Stripe ID
{ email: 1 },                            // Search by email
{ created_at: -1 }                       // Audit queries
```

**Constraints:**
- Unique index on `user_id`
- Unique index on `stripe_customer_id`

---

### 2. `tbl_payment_history`

Append-only log of all payment transactions processed through Stripe.

```javascript
{
  _id: ObjectId,
  user_id: String,           // Internal user ID
  stripe_payment_intent_id: String, // Stripe PaymentIntent ID (pi_xxx)
  stripe_charge_id: String,  // Stripe Charge ID (ch_xxx)
  amount: Long,              // Amount in smallest currency unit (cents)
  currency: String,          // ISO 4217 currency code (e.g., "usd")
  status: String,            // Enum: PENDING, SUCCEEDED, FAILED, REFUNDED, CANCELLED
  payment_method: {
    type: String,            // card, bank_transfer, etc.
    last4: String,           // Last 4 digits of card
    brand: String,           // visa, mastercard, etc.
    exp_month: Integer,
    exp_year: Integer
  },
  description: String,       // Payment description
  subscription_id: ObjectId, // Reference to subscription (if applicable)
  credit_amount: Long,       // Credits granted (if this is a credit purchase)
  metadata: {
    stripe_receipt_url: String,
    stripe_invoice_id: String,
    failure_code: String,
    failure_message: String,
    refund_amount: Long,
    refund_reason: String
  },
  created_at: ISODate,       // Transaction timestamp
  updated_at: ISODate,       // Status update timestamp
  processed_at: ISODate      // Webhook processing timestamp
}
```

**Indexes:**
```javascript
{ user_id: 1, created_at: -1 },           // User payment history
{ stripe_payment_intent_id: 1 },          // Unique lookup
{ stripe_charge_id: 1 },                  // Unique lookup
{ status: 1, created_at: -1 },            // Status-based queries
{ subscription_id: 1 }                    // Subscription payments
```

**Constraints:**
- Append-only (no updates except status transitions)
- Immutable `amount` and `currency` fields

---

### 3. `tbl_processed_stripe_events`

Webhook event deduplication and processing log with TTL.

```javascript
{
  _id: ObjectId,
  event_id: String,          // Stripe event ID (evt_xxx)
  event_type: String,        // Stripe event type (e.g., payment_intent.succeeded)
  api_version: String,       // Stripe API version at event time
  created_at: ISODate,       // Stripe event timestamp
  processed_at: ISODate,     // Local processing timestamp
  status: String,            // Enum: PENDING, PROCESSING, COMPLETED, FAILED, SKIPPED
  payload_hash: String,      // SHA-256 hash for integrity verification
  retry_count: Integer,      // Number of processing attempts
  error_message: String,     // Last error message (if failed)
  handler_name: String,      // Which handler processed this event
  idempotency_key: String,   // Generated key for idempotent processing
  metadata: {
    related_payment_intent: String,
    related_invoice: String,
    related_subscription: String,
    related_customer: String
  }
}
```

**Indexes:**
```javascript
{ event_id: 1 },                            // Unique constraint
{ event_type: 1, created_at: -1 },          // Event type queries
{ status: 1, processed_at: -1 },            // Processing status monitoring
{ idempotency_key: 1 },                     // Idempotency checks
{ payload_hash: 1 }                         // Duplicate detection
```

**Constraints:**
- Unique index on `event_id`
- TTL index on `processed_at` (30 days): `{ processed_at: 1, expireAfterSeconds: 2592000 }`

---

### 4. `tbl_plans`

Plan catalog — defines available subscription tiers and pricing.

```javascript
{
  _id: ObjectId,
  plan_code: String,         // Unique code (e.g., "FREE", "BASIC", "PRO")
  name: String,              // Display name for users
  description: String,       // Marketing description of features
  stripe_product_id: String, // Stripe Product ID (prod_xxx)
  stripe_price_id: String,   // Stripe Price ID (price_xxx)
  amount: Long,              // Price per period in cents (0 for FREE)
  currency: String,          // ISO 4217 currency code
  interval: String,          // day, week, month, year
  interval_count: Integer,   // e.g., 3 for every 3 months
  trial_period_days: Integer, // Free trial length (0 = no trial)
  credits_per_period: Long,  // Credits included per billing period
  features: [String],        // Feature flags this plan unlocks
  max_projects: Integer,     // Max projects (-1 = unlimited)
  max_team_members: Integer, // Max team members (-1 = unlimited)
  priority: Integer,         // Sort order (lower = higher tier)
  is_active: Boolean,        // Available for new subscriptions
  is_visible: Boolean,       // Shows on pricing page
  metadata: {
    stripe_recurring: Boolean,
    tax_code: String,
    stripe_metadata: Object
  },
  created_at: ISODate,
  updated_at: ISODate,
  deprecated_at: ISODate     // When plan was deprecated (null = active)
}
```

**Indexes:**
```javascript
{ plan_code: 1 },               // Unique lookup by code
{ stripe_price_id: 1 },         // Unique lookup by Stripe price
{ is_active: 1, priority: 1 },  // Active plans sorted by tier
{ is_visible: 1, is_active: 1 } // Visible plans for pricing page
```

**Constraints:**
- Unique index on `plan_code`
- Unique index on `stripe_price_id`
- Cannot subscribe to deprecated/inactive plans (enforced in application code)

---

### 5. `tbl_stripe_payment_methods`

Cached payment methods for quick checkout and recurring billing.

```javascript
{
  _id: ObjectId,
  user_id: String,           // Internal user ID
  stripe_customer_id: String, // Stripe customer ID
  stripe_payment_method_id: String, // Stripe PaymentMethod ID (pm_xxx)
  type: String,              // card, sepa_debit, etc.
  card: {
    brand: String,           // visa, mastercard, amex, etc.
    last4: String,
    exp_month: Integer,
    exp_year: Integer,
    fingerprint: String,     // For duplicate detection
    country: String,         // Issuing country
    funding: String          // credit, debit, prepaid
  },
  billing_details: {
    name: String,
    email: String,
    phone: String,
    address: {
      line1: String,
      line2: String,
      city: String,
      state: String,
      postal_code: String,
      country: String
    }
  },
  is_default: Boolean,       // Default payment method flag
  is_active: Boolean,        // Soft delete flag (default: true)
  created_at: ISODate,
  updated_at: ISODate,
  last_used_at: ISODate      // Last successful payment timestamp
}
```

**Indexes:**
```javascript
{ user_id: 1, is_active: 1 },              // User's active payment methods
{ stripe_payment_method_id: 1 },           // Unique lookup
{ stripe_customer_id: 1, is_default: 1 },  // Default method lookup
{ card.fingerprint: 1 }                    // Duplicate card detection
```

**Constraints:**
- Unique index on `stripe_payment_method_id`
- Partial unique index: `{ user_id: 1, is_default: 1 }` where `is_active=true`

---

### 6. `tbl_stripe_subscriptions`

Detailed Stripe subscription records synced from webhook events.

**Business Rule**: One user can only have ONE active/trialing subscription at any moment.

```javascript
{
  _id: ObjectId,
  user_id: String,           // Internal user ID
  plan_id: ObjectId,         // FK → tbl_plans._id
  stripe_subscription_id: String, // Stripe Subscription ID (sub_xxx)
  stripe_customer_id: String, // Stripe customer ID
  stripe_price_id: String,   // Stripe Price ID (price_xxx) — denormalized from plan
  status: String,            // Enum: TRIALING, ACTIVE, PAST_DUE, CANCELED, EXPIRED, UNPAID
  current_period_start: ISODate,
  current_period_end: ISODate,
  cancel_at_period_end: Boolean,
  canceled_at: ISODate,
  ended_at: ISODate,
  trial_start: ISODate,
  trial_end: ISODate,
  quantity: Integer,         // Seat count or usage quantity
  amount: Long,              // Amount per period (cents) — snapshot from plan at subscribe time
  currency: String,          // ISO 4217 currency code
  interval: String,          // day, week, month, year
  interval_count: Integer,   // e.g., 3 for every 3 months
  payment_method_id: String, // Payment method used
  latest_invoice_id: String, // Stripe Invoice ID (in_xxx)
  latest_payment_intent_id: String, // Latest PaymentIntent
  metadata: {
    stripe_invoice_url: String,
    discount_code: String,
    discount_amount: Long,
    tax_amount: Long,
    subscription_items: [{
      stripe_price_id: String,
      quantity: Integer
    }],
    previous_plan_id: ObjectId // For plan switch tracking
  },
  created_at: ISODate,
  updated_at: ISODate
}
```

**Indexes:**
```javascript
{ user_id: 1, status: 1 },                   // Active subscription lookup
{ stripe_subscription_id: 1 },               // Unique lookup
{ stripe_customer_id: 1 },                   // Customer subscriptions
{ plan_id: 1 },                              // Subscriptions per plan
{ status: 1, current_period_end: 1 },        // Expiring subscriptions
{ cancel_at_period_end: 1, current_period_end: 1 } // Pending cancellations
```

**Constraints:**
- Unique index on `stripe_subscription_id`
- **Partial unique index**: `{ user_id: 1 }` where `status IN ['ACTIVE', 'TRIALING']`
  - This enforces: one user can only have ONE active/trialing subscription at a time
- FK constraint on `plan_id` → `tbl_plans._id` (enforced in application code)

---

### 7. `tbl_stripe_invoices`

Invoice records for billing history and reconciliation.

```javascript
{
  _id: ObjectId,
  user_id: String,
  stripe_invoice_id: String,   // Stripe Invoice ID (in_xxx)
  stripe_subscription_id: String,
  stripe_payment_intent_id: String,
  stripe_customer_id: String,
  amount_due: Long,
  amount_paid: Long,
  amount_remaining: Long,
  currency: String,
  status: String,              // Enum: DRAFT, OPEN, PAID, UNCOLLECTIBLE, VOID
  number: String,              // Invoice number (user-facing)
  description: String,
  period_start: ISODate,
  period_end: ISODate,
  due_date: ISODate,
  paid_at: ISODate,
  finalized_at: ISODate,
  lines: [{
    description: String,
    amount: Long,
    quantity: Integer,
    plan_id: String,
    period_start: ISODate,
    period_end: ISODate
  }],
  tax: Long,
  discount: Long,
  subtotal: Long,
  total: Long,
  pdf_url: String,
  hosted_invoice_url: String,
  created_at: ISODate,
  updated_at: ISODate
}
```

**Indexes:**
```javascript
{ user_id: 1, created_at: -1 },              // User invoice history
{ stripe_invoice_id: 1 },                    // Unique lookup
{ stripe_subscription_id: 1 },               // Subscription invoices
{ status: 1, due_date: 1 },                  // Overdue invoice tracking
{ amount_remaining: 1 }                      // Unpaid balance queries
```

**Constraints:**
- Unique index on `stripe_invoice_id`

---

### 8. `tbl_stripe_webhook_logs`

Audit log for all incoming webhook events (separate from processed events).

```javascript
{
  _id: ObjectId,
  event_id: String,
  event_type: String,
  raw_payload: String,       // JSON string of raw webhook payload
  signature: String,         // Stripe-Signature header value
  ip_address: String,        // Source IP address
  received_at: ISODate,
  verified: Boolean,         // Signature verification result
  processed: Boolean,        // Whether event was queued for processing
  processing_status: String, // QUEUED, PROCESSING, COMPLETED, FAILED
  error_details: {
    code: String,
    message: String,
    stack_trace: String
  },
  processing_duration_ms: Long,
  handler_version: String    // Handler version for debugging
}
```

**Indexes:**
```javascript
{ event_id: 1 },                            // Unique lookup
{ event_type: 1, received_at: -1 },         // Event type analysis
{ verified: 1, received_at: -1 },           // Failed verification monitoring
{ processing_status: 1, received_at: -1 },  // Processing queue status
{ ip_address: 1 }                           // IP-based analysis
```

**Constraints:**
- Unique index on `event_id`
- TTL index on `received_at` (90 days): `{ received_at: 1, expireAfterSeconds: 7776000 }`

---

## Data Flow Diagrams

### Customer Creation Flow

```
User Registration
    ↓
Check tbl_stripe_customers (exists?)
    ↓ (no)
Create Stripe Customer via API
    ↓
Save to tbl_stripe_customers
    ↓
Return stripe_customer_id
```

### Payment Processing Flow

```
Payment Request
    ↓
Retrieve stripe_customer_id from tbl_stripe_customers
    ↓
Create PaymentIntent via Stripe API
    ↓
Save pending record to tbl_payment_history
    ↓
Receive webhook (payment_intent.succeeded)
    ↓
Verify in tbl_processed_stripe_events (duplicate?)
    ↓ (no)
Update tbl_payment_history status
    ↓
Record in tbl_processed_stripe_events
    ↓
Publish domain event (credit.granted / subscription.created)
```

### Webhook Processing Flow

```
Stripe Webhook POST /webhooks/stripe
    ↓
Verify signature (Stripe-Signature header)
    ↓
Log to tbl_stripe_webhook_logs
    ↓
Check tbl_processed_stripe_events (event_id exists?)
    ↓ (no)
Route to appropriate handler
    ↓
Handler processes event
    ↓
Record in tbl_processed_stripe_events with status
    ↓
Update related collections (payment_history, subscriptions, etc.)
```

---

## Index Strategy

### Performance Indexes

| Collection | Index | Purpose |
|------------|-------|---------|
| All | `{ created_at: -1 }` | Time-range queries |
| All | `{ user_id: 1 }` | User-centric lookups |
| `tbl_payment_history` | `{ status: 1, created_at: -1 }` | Status dashboard queries |
| `tbl_stripe_subscriptions` | `{ status: 1, current_period_end: 1 }` | Expiring subscription alerts |
| `tbl_processed_stripe_events` | `{ event_type: 1, status: 1 }` | Processing monitoring |

### Unique Constraints

| Collection | Field | Purpose |
|------------|-------|---------|
| `tbl_stripe_customers` | `user_id` | One Stripe customer per user |
| `tbl_stripe_customers` | `stripe_customer_id` | Prevent duplicate mapping |
| `tbl_plans` | `plan_code` | Unique plan identifier |
| `tbl_plans` | `stripe_price_id` | One plan per Stripe price |
| `tbl_payment_history` | `stripe_payment_intent_id` | Idempotency |
| `tbl_processed_stripe_events` | `event_id` | Webhook deduplication |
| `tbl_stripe_subscriptions` | `stripe_subscription_id` | Unique Stripe subscription |
| `tbl_stripe_subscriptions` | `user_id` (partial) | One active/trialing sub per user |
| `tbl_stripe_invoices` | `stripe_invoice_id` | Unique Stripe invoice |
| `tbl_stripe_payment_methods` | `stripe_payment_method_id` | Unique payment method |

### TTL Indexes

| Collection | Field | Expiry | Purpose |
|------------|-------|--------|---------|
| `tbl_processed_stripe_events` | `processed_at` | 30 days | Cleanup processed events |
| `tbl_stripe_webhook_logs` | `received_at` | 90 days | Audit log retention |

---

## Data Retention Policy

| Collection | Retention | Strategy |
|------------|-----------|----------|
| `tbl_stripe_customers` | Indefinite | Soft delete only |
| `tbl_plans` | Indefinite | Soft delete (deprecated_at) only |
| `tbl_payment_history` | 7 years | Compliance requirement |
| `tbl_processed_stripe_events` | 30 days | TTL index |
| `tbl_stripe_subscriptions` | 2 years after cancellation | Archive then delete |
| `tbl_stripe_invoices` | 7 years | Compliance requirement |
| `tbl_stripe_payment_methods` | Indefinite | Soft delete on removal |
| `tbl_stripe_webhook_logs` | 90 days | TTL index |

---

## Migration Considerations

### Initial Schema Migration

```javascript
// Create all collections with indexes
db.createCollection("tbl_plans");
db.createCollection("tbl_stripe_customers");
db.createCollection("tbl_payment_history");
db.createCollection("tbl_processed_stripe_events");
db.createCollection("tbl_stripe_payment_methods");
db.createCollection("tbl_stripe_subscriptions");
db.createCollection("tbl_stripe_invoices");
db.createCollection("tbl_stripe_webhook_logs");

// Create unique indexes
db.tbl_plans.createIndex({ plan_code: 1 }, { unique: true });
db.tbl_plans.createIndex({ stripe_price_id: 1 }, { unique: true });
db.tbl_plans.createIndex({ is_active: 1, priority: 1 });

db.tbl_stripe_customers.createIndex({ user_id: 1 }, { unique: true });
db.tbl_stripe_customers.createIndex({ stripe_customer_id: 1 }, { unique: true });

db.tbl_payment_history.createIndex({ stripe_payment_intent_id: 1 }, { unique: true, sparse: true });
db.tbl_payment_history.createIndex({ user_id: 1, created_at: -1 });

db.tbl_processed_stripe_events.createIndex({ event_id: 1 }, { unique: true });
db.tbl_processed_stripe_events.createIndex({ processed_at: 1 }, { expireAfterSeconds: 2592000 });

db.tbl_stripe_subscriptions.createIndex({ stripe_subscription_id: 1 }, { unique: true });
db.tbl_stripe_subscriptions.createIndex({ user_id: 1, status: 1 });
db.tbl_stripe_subscriptions.createIndex({ plan_id: 1 });
db.tbl_stripe_subscriptions.createIndex(
  { user_id: 1 },
  {
    unique: true,
    partialFilterExpression: { status: { $in: ["ACTIVE", "TRIALING"] } }
  }
);
```

### Zero-Downtime Deployment Strategy

1. **Phase 1**: Deploy new collections alongside existing schema
2. **Phase 2**: Dual-write to both old and new collections
3. **Phase 3**: Backfill historical data
4. **Phase 4**: Switch reads to new collections
5. **Phase 5**: Remove old schema and dual-write logic

---

## Security Considerations

### Sensitive Data Handling

- **Never store** full credit card numbers, CVV, or Stripe secret keys
- **Only store** last4 digits, fingerprints, and public identifiers
- **Encrypt** `stripe_customer_id` at rest if PII regulations apply
- **Audit log** all access to payment method details

### Access Control

| Role | Read Access | Write Access |
|------|-------------|--------------|
| Payment Service | All collections | All collections |
| Analytics Service | Payment history, invoices (read-only) | None |
| Support Portal | Customer mapping, payment status (masked) | None |
| Admin | All (audit logged) | Manual corrections only |

---

## Monitoring & Alerts

### Key Metrics

| Metric | Threshold | Alert |
|--------|-----------|-------|
| Webhook processing latency | > 5 seconds | P2 - Investigate |
| Failed webhook rate | > 5% in 5 min | P1 - Immediate |
| Duplicate event detection rate | > 1% | P2 - Investigate |
| Payment status stuck in PENDING | > 10 minutes | P1 - Immediate |
| Stripe API error rate | > 2% | P2 - Investigate |

### Dashboard Queries

```javascript
// Pending payments older than 10 minutes
db.tbl_payment_history.find({
  status: "PENDING",
  created_at: { $lt: new Date(Date.now() - 600000) }
});

// Failed webhooks in last hour
db.tbl_processed_stripe_events.find({
  status: "FAILED",
  processed_at: { $gt: new Date(Date.now() - 3600000) }
});

// Active subscriptions expiring in 7 days
db.tbl_stripe_subscriptions.find({
  status: "ACTIVE",
  current_period_end: {
    $lt: new Date(Date.now() + 604800000),
    $gt: new Date()
  }
});
```

---

## Relationships to Existing Collections

### Integration with Credit System

```
tbl_stripe_customers (1) ←→ (1) tbl_credit_accounts
tbl_payment_history (N) ←→ (1) tbl_credit_accounts (via credit_amount)
```

### Integration with Subscription System

```
tbl_stripe_customers (1) ←→ (1) tbl_subscriptions
tbl_stripe_subscriptions (1) ←→ (1) tbl_subscriptions (sync target)
tbl_payment_history (N) ←→ (1) tbl_subscriptions
```

### Event Publishing

```
tbl_processed_stripe_events → PaymentEventPublisher → RabbitMQ
  - credit.granted
  - credit.consumed
  - subscription.created
  - subscription.cancelled
```

---

## Testing Strategy

### Test Data Generation

```javascript
// Create test customer
db.tbl_stripe_customers.insertOne({
  user_id: "test-user-001",
  stripe_customer_id: "cus_test_123",
  email: "test@example.com",
  created_at: new Date(),
  updated_at: new Date(),
  is_active: true
});

// Create test payment
db.tbl_payment_history.insertOne({
  user_id: "test-user-001",
  stripe_payment_intent_id: "pi_test_456",
  amount: 999,
  currency: "usd",
  status: "SUCCEEDED",
  created_at: new Date(),
  processed_at: new Date()
});

// Create test webhook event
db.tbl_processed_stripe_events.insertOne({
  event_id: "evt_test_789",
  event_type: "payment_intent.succeeded",
  processed_at: new Date(),
  status: "COMPLETED"
});
```

### Test Scenarios

1. **Idempotency**: Process same webhook event twice, verify single payment record
2. **TTL Cleanup**: Verify processed events expire after 30 days
3. **Status Transitions**: Test all valid payment status transitions
4. **Concurrent Webhooks**: Simulate duplicate webhook delivery
5. **Signature Verification**: Test invalid signature rejection
