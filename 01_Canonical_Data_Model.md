# Canonical Data Model — Order, Obligation, PaymentRequest, Session, Attempt, Capture & Refund

**Purpose:** Consolidated domain model reflecting the later API simplification review. This is the recommended external resource model; earlier PaymentIntent terminology is retained in the historical assessment documents for comparison.

## 1. Core principles

1. **Order is the commercial contract**: what was purchased, by whom, and the commercial/fulfilment lifecycle.
2. **Obligation is the party-level claim**: who is owed what, on what basis, including marketplace sellers, franchisees, platform fees and tax authorities.
3. **PaymentRequest (`pr_`) is the durable collection aggregate**: the amount and currency to collect, the collection lifecycle, and the aggregate of attempts, captures and refunds. It is the external successor to the earlier PaymentIntent concept; do not expose both as competing durable objects.
4. **Session (`cs_`, `ts_`, `as_`, `link_`) is a typed interaction binding**: checkout, terminal, agent, or payment-link interaction. It is not a money object.
5. **Attempt (`att_`) is one execution/network try**: internal and read-only to merchants. A retry creates another attempt under the same PaymentRequest when it is still the same collection objective.
6. **Capture and Refund are money-operation resources** tied to the collection/authorized execution; they are not states masquerading as attempts.
7. **Order, collection, session and fulfilment states are independent**. Derive summaries; never combine them into one mega-state machine.
8. **Merchant-owned OMS remains authoritative**. If a merchant already owns its Order, pass `order_ref`; do not require a duplicate PSP Order.

## 2. Entity relationship model

```text
Merchant / Customer
       │
       ▼
     Order (ord_)  ─────────────── Commercial & fulfilment record
       │
       ├── 1..N LineItems (optional thin representation)
       │
       ├── 1..N Obligations (obl_) ─ Party claim / allocation basis
       │       ├── beneficiary (seller, franchisee, platform, tax authority)
       │       ├── amount / currency / calculation basis
       │       └── release, reserve, clawback conditions
       │
       └── 0..N PaymentRequests (pr_) ─ Collection aggregates
               │
               ├── 0..N Sessions (cs_ / ts_ / as_ / link_)
               │       └── one active IN_PROGRESS binding per pr_ at a time
               │
               ├── 0..N Attempts (att_) [read-only execution trace]
               │       └── execution_id → orchestrator / decision platform
               │
               ├── 0..N Captures (cap_)
               │
               └── 0..N Refunds (rfnd_)

Disputes and settlement/reconciliation records reference the relevant
PaymentRequest, Attempt, Capture and Order keys; they are separate domains.
```

A PaymentRequest can exist without a PSP Order when the merchant owns its OMS. An Order can exist without a PaymentRequest (quote, unpaid order, cash/external settlement). An Order can have multiple obligations and multiple collection requests. Avoid assuming a strict one-order/one-payment relationship.

## 3. Resource definitions

### Order — `ord_`

**Authority:** PSP only when the merchant has no OMS or explicitly chooses PSP-managed commerce. Otherwise the merchant's `order_ref` is a correlation key and the PSP must not mirror the order.

| Attribute | Meaning |
|---|---|
| `id` | PSP order identifier (`ord_`) |
| `merchant_id` | Owning merchant / platform tenant |
| `customer_id` | Customer reference, if known |
| `reference` | Merchant's external order reference |
| `currency` | Order currency |
| `total_amount` | Commercial total, represented in minor units |
| `line_items[]` | Optional thin items for hosted checkout / order display; not a full OMS schema |
| `obligations[]` | Claims that explain allocation of value to beneficiaries |
| `fulfillment_status` | `DRAFT`, `OPEN`, `FULFILLING`, `FULFILLED`, `CLOSED`; cancellation/return side transitions |
| `created_at`, `updated_at` | Timestamps |
| `metadata` | Merchant-defined non-authoritative metadata |

Order must not carry authorization state. Fulfilment state is not inferred from payment state.

### Obligation — `obl_`

Represents **who is owed what and why**. Needed for multi-party commerce and money allocation, not merely as a fixed fee deduction.

| Attribute | Meaning |
|---|---|
| `id`, `order_id` | Identity and parent order |
| `beneficiary` | Party reference/type; may be seller, franchisee, platform, tax authority |
| `amount`, `currency` | Claim amount |
| `basis` | Fixed amount, percentage, gross basis, tax rule, contract rule, etc. |
| `conditions` | Release, reserve, fulfilment, dispute or clawback conditions |
| `status` | Claim lifecycle; distinct from payment collection state |

### PaymentRequest — `pr_`

The externally addressable **collection aggregate**. Earlier design material calls this Payment Intent; the later review resolves the naming collision by retaining `paymentRequest`, which already exists in the production API as the durable aggregate.

| Attribute | Meaning |
|---|---|
| `id` | PaymentRequest identifier (`pr_`) |
| `merchant_id`, `customer_id` | Tenant and customer context |
| `order_id` / `order_ref` | Optional PSP order link or merchant-owned order correlation |
| `obligation_ids[]` | Optional claims being fulfilled |
| `amount`, `currency` | Collection amount and currency |
| `amount_details` | Optional tax, tip, shipping, surcharge and breakdown data |
| `capture_method` | Immediate or manual/delayed capture policy |
| `payment_method` | Selected or reusable instrument reference; sensitive credential material remains in Vault |
| `state` | Collection state machine below |
| `authorization[]` | Authorization outcomes / references, with attempt linkage |
| `captures[]`, `refunds[]` | Money-operation references and aggregate totals |
| `client_secret` | Scoped confirmation secret where applicable; never a PAN or reusable credential |
| `idempotency_key` | Request deduplication key (normally enforced by request ledger) |
| `expires_at`, `created_at`, `updated_at` | Lifecycle timestamps |
| `metadata` | Merchant-defined metadata |

### Session — typed interaction binding

| Type | Prefix | Purpose |
|---|---|---|
| Checkout Session | `cs_` | Web checkout / hosted checkout |
| Terminal Session | `ts_` | Reader, terminal or softPOS interaction |
| Agent Session | `as_` | Agent-delegated interaction under a signed mandate |
| Payment Link | `link_` | Reusable or single-use link surface |

Common fields: `id`, `payment_request_id`, `type`, `state`, `presented_at`, `expires_at`, `return_url`, `surface_context`, `created_at`. Session completion does not imply payment authorization or capture. A single PaymentRequest may be rebound to sequential sessions; only one session may be `IN_PROGRESS` for a given PaymentRequest at a time.

### Attempt — `att_`

One actual execution try. Keep it internal/read-only; merchant APIs may expose a sanitized decision trace.

| Attribute | Meaning |
|---|---|
| `id`, `payment_request_id` | Attempt identity and parent collection |
| `execution_id` | Unique orchestration execution/correlation ID |
| `sequence` | Attempt ordinal within collection |
| `status` | `CREATED`, `IN_FLIGHT`, `AUTHORIZED`, `DECLINED`, `ERROR`, `UNKNOWN`, `CANCELLED` (connector-specific details mapped to canonical outcome) |
| `payment_method_ref` | Credential/vault reference used for this attempt |
| `credential_type` | PAN/token/network token/wallet credential classification, as permitted |
| `acquirer_id`, `processor_id`, `route_id` | Chosen execution route |
| `authentication_ref`, `authentication_outcome` | 3DS/SCA reference and result |
| `request_message`, `response_code`, `network_transaction_id` | Sanitized network/processor trace |
| `started_at`, `completed_at` | Execution timestamps |
| `retry_of_attempt_id` | Optional predecessor link |
| `decision_record_id` | Decision Platform audit link |

Attempt does not own the merchant's collection objective or the collection's final lifecycle. Do not let clients create arbitrary attempts when using the managed collection API.

### Capture — `cap_` and Refund — `rfnd_`

Capture fields: `id`, `payment_request_id`, `attempt_id` (where applicable), `amount`, `currency`, `status`, `processor_reference`, `created_at`. Multiple captures may fulfil one authorization; cumulative captures cannot exceed the permitted authorization amount except where network/connector rules explicitly support incremental authorization.

Refund fields: `id`, `payment_request_id`, `capture_id` (where applicable), `amount`, `currency`, `reason`, `status`, `processor_reference`, `created_at`. Partial and multiple refunds are supported subject to captured/refundable balance.

## 4. State machines and ownership

### PaymentRequest collection state

```text
REQUIRES_INSTRUMENT
       │ instrument supplied
       ▼
REQUIRES_ACTION  ── SCA / redirect / customer action
       │ action completed
       ▼
PROCESSING       ── network in flight; attempts/replans hidden
       │ approved
       ▼
AUTHORIZED       ── approved; funds NOT yet captured
       │ capture(s)
       ▼
PARTIALLY_CAPTURED
       │ final capture / full amount captured
       ▼
CAPTURED
       │ settlement confirmation (if surfaced here)
       ▼
SETTLED

Terminal/side outcomes: FAILED, CANCELLED, EXPIRED
```

`AUTHORIZED` is deliberately distinct from `CAPTURED`. Avoid ambiguous `SUCCEEDED` where it could mean either authorization or money movement. Asynchronous payment methods may require additional method-specific pending states, mapped into the canonical collection lifecycle without collapsing authorization and capture semantics.

### Session state

```text
CREATED → ACTIVE → IN_PROGRESS → COMPLETED
                       ├──────→ ABANDONED (may be rebound via a new session)
                       └──────→ EXPIRED
```

### Order fulfilment state

`DRAFT → OPEN → FULFILLING → FULFILLED → CLOSED`, with cancellation and return as side transitions. No authorization state belongs on Order.

### Three independent dimensions

- **Commercial / fulfilment:** Order state.
- **Collection / money:** PaymentRequest, Attempt, Capture and Refund state.
- **Interaction:** Session state.

A completed Session with an `AUTHORIZED` PaymentRequest is valid. A fulfilled Order may have multiple collection requests. Derive read models rather than forcing cross-domain state transitions.

## 5. Invariants

- One idempotent API request resolves to one canonical resource/result; duplicate suppression is not a decline.
- One PaymentRequest represents one collection objective and currency. If the objective changes materially, amend under explicit rules or create a new collection request; never silently mutate completed history.
- Attempts are append-only execution records; retries create new attempts and preserve prior outcomes.
- Captures and refunds are immutable money-operation records with explicit status transitions and references.
- No raw PAN, cryptogram or sensitive authentication data is stored on Order, PaymentRequest or Attempt; use Credential Vault references and scoped tokens.
- `order_ref` is a correlation key for merchant-owned orders; no mirrored Order and no sync job.
- Ledger/owed-vs-collected view is derived and read-only, keyed by `order_ref` or `order_id`; never a second writable order system.
- Every event and financial row carries stable join keys: merchant, order/order_ref, payment_request, attempt, capture/refund, execution_id and processor/network references as applicable.
