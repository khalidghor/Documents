# Payment Attempt, Payment Intent, and Order API Assessment

## Executive Summary

For a modern PSP, **Payment Attempt**, **Payment Intent**, and **Order** should be treated as three levels of abstraction rather than competing APIs:

- **Payment Attempt** — “Try to move this money using this payment credential.”
- **Payment Intent** — “I intend to collect this amount from this customer.”
- **Order** — “This is the commercial transaction/customer purchase, and one or more payments may fulfill it.”

Recommended architecture:

```text
                    ORDER
                      │
             1..N Payment Intents
                      │
             1..N Payment Attempts
                      │
             Execution ID
                      │
                ORCHESTRATOR
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
     Risk        Credential        Routing
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                  Processor
```

The key principle is:

> **Order should be the commerce abstraction, Payment Intent the payment-domain abstraction, and Payment Attempt the execution abstraction.**

---

# 1. Payment Attempt API

A Payment Attempt is the most transaction-centric abstraction.

Conceptually:

```http
POST /payment_attempts
```

```json
{
  "amount": 12500,
  "currency": "USD",
  "payment_method": "pm_123",
  "merchant_reference": "INV-123"
}
```

It represents an **individual effort to obtain authorization or complete a payment**.

## Client Benefits

| Benefit | Why it matters |
|---|---|
| Simple mental model | Client says: “Take $125.” |
| Maximum control | Client controls when and how each attempt happens |
| Excellent for simple checkout | One customer → one payment → one authorization |
| Natural retry model | A new attempt can be created after a decline |
| Clean transaction analytics | Each authorization attempt has its own identity |
| Works well with orchestration | PSP can independently optimize each attempt |
| Lower integration complexity | Client does not need to understand complex payment lifecycle |

The architectural advantage is that the attempt becomes the **atomic execution unit**.

For a modular PSP:

```text
Payment Attempt
       │
       ▼
Execution ID
       │
       ▼
Orchestrator
       │
 ┌─────┼─────┬─────┐
 ▼     ▼     ▼     ▼
Risk  Auth  Route  Credential
              │
              ▼
           Acquirer
```

This fits well with an architecture based on an **execution identifier and independently deployable modules**.

---

# 2. Payment Intent API

Payment Intent moves one level above the individual attempt.

Instead of:

> “Authorize $125.”

the merchant says:

> “I want to collect $125 from this customer.”

The PSP then manages the execution of that intent.

Example:

```text
Payment Intent
    │
    ├── Attempt #1 → Declined
    │
    ├── Attempt #2 → 3DS
    │
    └── Attempt #3 → Authorized
```

## Client Benefits

### 2.1 Persistent Payment Lifecycle

The merchant receives a stable payment object whose state can evolve:

```text
Created
   ↓
Requires Payment Method
   ↓
Requires Authentication
   ↓
Processing
   ↓
Succeeded
```

This is much easier than reconstructing the payment state from individual transactions.

It is particularly valuable for asynchronous and multi-step payment methods.

---

### 2.2 Authentication Becomes Part of the Payment Lifecycle

A Payment Intent provides a natural place to manage authentication:

```text
Payment Intent
       │
       ▼
Risk evaluation
       │
       ├── Low risk → Authorization
       │
       └── Authentication required
                    │
                    ▼
                   3DS
                    │
                    ▼
                Authorization
```

The merchant does not have to create a new conceptual payment whenever authentication is required.

This is especially valuable when the PSP owns the authentication and exemption decisioning layer.

---

### 2.3 Retry Without Changing the Customer's Intent

Example:

```text
Payment Intent = $100 Nike purchase

Attempt 1
Visa → Declined

Attempt 2
Mastercard → Declined

Attempt 3
Network Token → Approved
```

The merchant still has:

```text
PaymentIntent PI_123 = $100
```

The attempts are execution details underneath the intent.

This creates a powerful separation:

> **Intent = customer/business objective**  
> **Attempt = PSP execution**

That separation is highly valuable for a Decision Platform.

---

# 3. The Biggest Payment Intent Benefit

The core benefit can be summarized as:

> **The client delegates payment-state management to the PSP.**

Without Payment Intent:

```text
Merchant application

Create payment
     ↓
Declined?
     ↓
Try again
     ↓
3DS?
     ↓
Redirect
     ↓
Come back
     ↓
Try authorization
     ↓
Check status
```

With Payment Intent:

```text
Merchant

Create Payment Intent
        ↓
       PSP
        ↓
 ┌──────┴─────────┐
 │ Risk           │
 │ Authentication │
 │ Credential     │
 │ Routing        │
 │ Retry          │
 └──────┬─────────┘
        ↓
    Final state
```

This substantially reduces merchant-side payment orchestration complexity.

---

# 4. Why Introduce Order?

An **Order is not fundamentally a payment object**.

It represents the **commercial obligation or purchase**.

For example:

```text
Order
ORD-123
$1,000
```

could contain:

```text
Airline ticket       $700
Hotel                 $250
Baggage                $50
---------------------------
Total                $1,000
```

The customer might pay:

```text
$400 Visa
$300 Mastercard
$300 PayPal
```

So:

```text
                 ORDER
                   │
       ┌───────────┼───────────┐
       ▼           ▼           ▼
 Payment Intent  Payment Intent Payment Intent
     $400            $300          $300
       │              │             │
   Attempts        Attempts       Attempts
```

Order is therefore a natural parent for multiple payment obligations.

---

# 5. Benefits of Order Over Payment Intent

## 5.1 Separates Commerce from Money Movement

Payment Intent says:

> “Collect $1,000.”

Order says:

> “Customer purchased these things for $1,000.”

This distinction matters because the payment amount can change.

Example:

```text
Order
$100

Payment Intent
$100
```

Customer adds shipping:

```text
Order
$115

Payment Intent
$115
```

Or the customer makes multiple payments:

```text
Order = $115

Payment #1 = $50
Payment #2 = $65
```

The **Order remains the commercial anchor**.

---

## 5.2 Enables Multiple Payments

Example marketplace or travel transaction:

```text
Order $1,000

Customer pays:
   Visa       $600
   PayPal     $200
   Gift Card  $200
```

Or:

```text
Order
├── Airfare       $700
├── Hotel         $300
└── Insurance     $100

Payments
├── Card          $500
├── BNPL          $300
└── Gift card     $300
```

Payment Intent alone can become awkward because the payment object is being asked to represent the commercial transaction.

Order provides the natural parent.

---

## 5.3 Supports Partial Fulfillment

Example:

```text
Order = $500

Product A = $200
Product B = $300
```

Product A ships today and Product B next week.

The payment lifecycle and fulfillment lifecycle are different:

```text
                  ORDER
                   │
       ┌───────────┴───────────┐
       ▼                       ▼
 Fulfillment A            Fulfillment B
    $200                      $300
       │                       │
       ▼                       ▼
 Payment                  Payment
```

This is important for:

- Marketplaces
- Retail
- Travel
- Lodging
- Subscriptions
- Restaurants/QSR
- Endless aisle
- Backorders

---

## 5.4 Natural Place for Refund Aggregation

Without Order:

```text
Payment Intent
     ↓
Payment
     ↓
Refund
```

With Order:

```text
Order
 │
 ├── Payment Intent
 │      └── Payment Attempt
 │
 ├── Payment Intent
 │      └── Payment Attempt
 │
 ├── Refund
 │
 └── Dispute
```

The merchant can answer:

> “How much of this customer's order has been paid, refunded, disputed, and remains outstanding?”

That is much harder if Payment Intent is the highest-level object.

---

## 5.5 Enables Payment Allocation

This is particularly valuable for multi-pay-in collection.

Example:

```text
ORDER-123
$1,000

Payment collection strategy:

1. Stored card → $500
2. Gift card → $200
3. ACH → $300
```

The Order can maintain:

```text
Order Total           $1,000
Collected                $700
Remaining                $300
```

while the PSP independently creates Payment Intents and Attempts.

---

# 6. Order for Marketplaces

Marketplace transactions demonstrate why Order can be a higher-level abstraction.

Example:

```text
Order
$250

Customer
   │
   ▼
Payment
   │
   ├── Merchant A $180
   ├── Merchant B  $50
   └── Platform    $20
```

The payment object alone does not adequately describe all of the commercial relationships.

Order can represent:

```text
Commercial transaction
        │
        ├── Customer
        ├── Merchant(s)
        ├── Platform
        ├── Fulfillment
        ├── Payment(s)
        ├── Refund(s)
        └── Dispute(s)
```

---

# 7. Side-by-Side Comparison

| Capability | Payment Attempt | Payment Intent | Order |
|---|---:|---:|---:|
| Execute a payment | **Excellent** | Excellent | Indirect |
| Simple checkout | **Excellent** | Excellent | More than needed |
| Stable payment lifecycle | Limited | **Excellent** | Excellent |
| Authentication lifecycle | Limited | **Excellent** | Through Intent |
| Retry management | Manual/client | **PSP-managed** | Through Intent |
| Smart routing | Excellent | **Excellent** | Through Intent |
| Multiple attempts | Client-managed | **Native concept** | Through Intent |
| Multiple payments | Awkward | Possible | **Excellent** |
| Partial payments | Awkward | Possible | **Excellent** |
| Partial fulfillment | Poor | Poor | **Excellent** |
| Line items | Poor | Limited | **Excellent** |
| Refund aggregation | Payment-level | Intent-level | **Order-level** |
| Marketplace | Limited | Possible | **Excellent** |
| Split tender | Poor | Possible | **Excellent** |
| Subscription | Possible | **Excellent** | Useful |
| Invoice collection | Possible | **Excellent** | **Excellent** |
| Merchant integration simplicity | **High** | **High** | Medium |
| Commerce abstraction | Low | Medium | **High** |

---

# 8. Recommended PSP Object Model

For the PSP architecture, support all three rather than forcing clients into one abstraction.

```text
                    ORDER
                      │
             1..N Payment Intents
                      │
             1..N Payment Attempts
                      │
             Execution ID
                      │
                ORCHESTRATOR
                      │
       ┌──────────────┼──────────────┐
       ▼              ▼              ▼
     Risk        Credential        Routing
       │              │              │
       └──────────────┼──────────────┘
                      ▼
                  Processor
```

Clients can enter at the abstraction level that fits their needs.

### Simple merchant

```http
POST /payment_attempts
```

Use when the merchant knows exactly what payment it wants to execute.

### Sophisticated ecommerce merchant

```http
POST /payment_intents
```

Use when the merchant wants the PSP to manage authentication, retries, asynchronous states, and optimization.

### Commerce / marketplace / travel client

```http
POST /orders
```

Then:

```http
POST /payment_intents
```

Use Order for the commercial transaction and Payment Intent for money collection.

---

# 9. Relationship to the Decision Platform

The model maps particularly well to a Decision Platform.

The **Payment Intent** is the object the PSP optimizes.

```text
                PAYMENT INTENT
                     │
             "Collect $250"
                     │
                     ▼
              DECISION PLATFORM
                     │
       ┌─────────────┼─────────────┐
       ▼             ▼             ▼
   Credential      Risk          Routing
   decision        decision       decision
       │             │             │
       └─────────────┼─────────────┘
                     ▼
              PAYMENT ATTEMPT
                     │
                     ▼
                PROCESSOR
```

Each attempt generates experimental and outcome data:

```text
Intent
  │
  ├── Attempt #1
  │    Route A
  │    Declined
  │
  ├── Attempt #2
  │    Route B
  │    3DS
  │
  └── Attempt #3
       Network Token
       Approved
```

The Decision Platform can then learn which execution strategy performs best for a given context:

- Merchant
- Customer
- BIN
- Issuer
- Credential type
- Amount
- Geography
- Device
- Risk profile
- Payment method
- Processor
- Network
- Authentication state

This creates a clean separation between **business intent** and **execution optimization**.

---

# 10. Recommended API Philosophy

### Payment Attempt

> **“I know exactly what payment I want to execute. Just execute it.”**

Best for:

- Simple merchants
- Direct authorization
- Low-complexity integrations
- Server-to-server payments
- Specialized payment flows

### Payment Intent

> **“I want to collect this money. You manage the payment lifecycle.”**

Best for:

- Ecommerce
- Mobile
- 3DS
- Retries
- Asynchronous methods
- Tokenized credentials
- Optimization
- Sophisticated payment orchestration

### Order

> **“I am selling something. Manage the commercial transaction and let payments fulfill it.”**

Best for:

- Marketplaces
- Retail
- Travel
- Lodging
- Split tender
- Partial payments
- Partial fulfillment
- Invoices
- Multi-pay-in
- Complex commerce

---

# 11. Final Architectural Principle

The strongest architecture is:

```text
┌───────────────────────────────────────────┐
│                  ORDER                    │
│                                           │
│  Commercial transaction / purchase        │
│                                           │
│      ┌─────────────────────────────┐      │
│      │      PAYMENT INTENT         │      │
│      │                             │      │
│      │  Payment objective/lifecycle│      │
│      │                             │      │
│      │    ┌──────────────────┐     │      │
│      │    │ PAYMENT ATTEMPT  │     │      │
│      │    │                  │     │      │
│      │    │ Execution        │     │      │
│      │    └────────┬─────────┘     │      │
│      └─────────────┼───────────────┘      │
└────────────────────┼──────────────────────┘
                     ▼
              DECISION PLATFORM
                     │
              ORCHESTRATOR
                     │
        ┌────────────┼────────────┐
        ▼            ▼            ▼
      Risk       Credential     Routing
        │            │            │
        └────────────┼────────────┘
                     ▼
                 Processor
```

**In one sentence:**

> **Order represents what the customer bought; Payment Intent represents the money the merchant intends to collect; Payment Attempt represents how the PSP actually tries to collect it.**

That separation gives your PSP a clean domain model while allowing simple merchants to use a lightweight Attempt API and sophisticated merchants to progressively adopt Intent and Order capabilities.
