# Order use case — `<UC-ID>`: `<short name>`

> Copy this file to `use-cases/<UC-ID>-<slug>.md` and fill every section. Write "n/a" rather than deleting a section, so reviewers can tell "not applicable" from "not considered".
>
> The field names in `code` match the specification: the Order is in [docs/02](../docs/02-order-resource.md), the Payment Intent in [docs/03](../docs/03-payment-intent.md), the lifecycle in [docs/05](../docs/05-payment-lifecycle.md) and marketplaces in [docs/08](../docs/08-marketplace-and-ledgers.md). This makes section 10 (fit and gaps) a mechanical check.

## 1. Summary

| Field | Value |
|---|---|
| Use case ID | `UC-___` |
| Name | |
| Industry profile | `qsr` · `e_retail` · `retail` · `grocery` · `marketplace` · `payfac` · `hospitality` · `services` · `digital` · `b2b` |
| Channel(s) | `web` · `app` · `pos` · `kiosk` · `drive_thru` · `call_center` · `agent` · `marketplace_partner` · `invoice` · `terminal` |
| Customer present? | Customer-initiated (CIT) · merchant-initiated (MIT) · agent-initiated |
| Markets and currencies | |
| Requested by / owner | |
| Priority | Must · Should · Could |
| Status | Draft · In review · Accepted · Built |
| Reference merchant(s) | |

**One-sentence story.** As a `<buyer/merchant/seller/agent>`, I want to `<goal>` so that `<outcome>`.

**Why it matters.** What is the volume, revenue, or competitive gap? Which competitor already does this, and how?

## 2. Parties

| Party | Who | Notes |
|---|---|---|
| Buyer | Guest, known customer, B2B buyer, or agent acting under a mandate | |
| Merchant / platform | | |
| Sellers / sub-merchants | Count and examples | |
| Merchant of record | Platform or seller | |
| Fund-flow model | `platform_mor_separate_transfers` · `platform_mor_destination` · `seller_mor_on_behalf_of` · `seller_mor_direct` · `split_at_authorization` · `payfac_submerchant` · `single_merchant` | |
| Staff / third parties | Server, cashier, courier, aggregator, 3PL | |

## 3. What is being sold

| Line | `type` | Catalog ref / SKU | Qty (decimal?) | Modifiers / bundle | Seller | Fulfillment group | Tax code | Attributes (age-restricted, perishable…) |
|---|---|---|---|---|---|---|---|---|
| li_1 | | | | | | | | |

- **Discounts:** each with `funded_by` (`merchant` · `marketplace` · `seller` · `manufacturer` · `split`), and line-level or prorated.
- **Fees:** each with its `type` and `payee`.
- **Tips:** when added (`checkout` · `post_authorization` · `receipt`) and who receives them.
- **Shipping:** who charges it and who keeps it.
- **Tax:** inclusive or exclusive, which jurisdictions, and who is liable (`seller` · `marketplace_facilitator` · `deemed_supplier` · `buyer_reverse_charge`).

## 4. Fulfillment

| Group | `method` | `fulfilled_by` | Location | Promised time/window | `capture_trigger` |
|---|---|---|---|---|---|
| fg_1 | | | | | |

Fulfillment events this use case depends on (shipped, delivered, handed over, closed):

## 5. How it is paid — collection plan

| Field | Value |
|---|---|
| `collection_plan.strategy` | `pay_now` · `authorize_then_capture_on_fulfillment` · `authorize_then_capture_on_close` · `deposit_then_balance` · `installments` · `bnpl` · `split_tender` · `pay_at_pickup` · `pay_on_delivery` · `verify_now_charge_later` · `invoice` · `externally_collected` |
| Payment methods offered | Card (credit/debit/prepaid) · wallet · ACH · pay by bank · RTP/FedNow · BNPL · gift card · store credit · cash/other tender · agent mandate |
| Number of Payment Intents per order | One, or many (and why) |
| `capture_method` | `automatic` · `manual` · `multi` |
| Authorization type | `final` · `estimated` · `pre_authorization` · `incremental_eligible` |
| `partial_authorization` | `reject` · `accept_and_collect_remainder` · `accept_and_reduce` |
| `authorization_expiry_policy` | `reauthorize` · `notify` · `cancel_order_lines` · `collect_new_payment` · `capture_available` |
| `reauthorization_mode` | `auto` · `reauth_then_reverse` · `reverse_then_reauth` |
| Stored-credential consent collected? | Yes / no — needed for reauthorization and MIT |
| Mandate needed? | Type and scope (recurring, installment, ACH, delegated agent) |
| Overcapture / tip tolerance | |
| Authentication expectations | 3DS / SCA / exemptions / PIN / none |
| Expected time from authorization to last capture | Compare it to the authorization validity in `config/authorization-validity.yaml` |

## 6. Happy path, step by step

One row per step. Fill all five status columns, so you can see how the order moves as the payment moves.

| # | Actor | Action / API call | Order `status` | `payment_status` | `fulfillment_status` | Intent `status` | Operation (type, amount) | Money / ledger effect |
|---|---|---|---|---|---|---|---|---|
| 1 | Buyer | `POST /v1/orders` (place) | | | | | | |
| 2 | | | | | | | | |
| 3 | | | | | | | | |

Webhooks the merchant must handle on this path:

## 7. Exceptions and variations

Tick each one that applies and describe the expected behaviour. Each ticked row needs an acceptance test in section 11.

| ✓ | Scenario | Expected behaviour (order + payment + money) |
|---|---|---|
| ☐ | Soft decline / step-up (3DS) | |
| ☐ | Hard decline, customer retries with another method | |
| ☐ | Partial authorization | |
| ☐ | Timeout / unknown outcome | |
| ☐ | Amount increases (add items, tip, substitution) — incremental or overcapture | |
| ☐ | Line canceled before fulfillment — partial reversal | |
| ☐ | Whole order canceled | |
| ☐ | Split shipment / several captures | |
| ☐ | Authorization expires before fulfillment — reauthorize, notify or cancel | |
| ☐ | Reauthorization declined | |
| ☐ | Capture fails | |
| ☐ | Full refund / one partial refund / several partial refunds | |
| ☐ | Return with exchange (even or uneven) | |
| ☐ | Refund to the original method fails (closed card) — fallback | |
| ☐ | Pre-dispute alert (Ethoca / CDRN / RDR / Order Insight) | |
| ☐ | Dispute: fraud / not received / not as described / credit not processed | |
| ☐ | Dispute after a partial refund | |
| ☐ | ACH return — R01 NSF (re-present) or R10 unauthorized | |
| ☐ | NSF on a merchant-initiated charge — managed retry | |
| ☐ | Offline / store-and-forward authorization | |
| ☐ | Agent acting outside its mandate — step-up or refuse | |
| ☐ | Other: | |

## 8. Money

- **Who earns what on each capture.** Fill one row per party.

  | Party | Gross | Commission / fees | Tax | Net |
  |---|---|---|---|---|
  | | | | | |

- **On a refund.** Who is debited? Is commission refunded? Is there an admin fee? Is tax reversed?
- **On a lost dispute.** Who is liable, and which allocation rule covers it? Who responds? Who pays the fee? Is the seller's money held while the dispute is open?
- **Payouts.** When are funds available, and when are they paid out? Are any reserves or holds involved?
- **Ledger postings that must exist.** Cover both ledgers if it is a marketplace. Use the worked example in docs/08 §4 as the pattern.

## 9. Constraints

| Area | Requirement |
|---|---|
| Volume / peak TPS | |
| Latency (checkout, POS, kitchen) | |
| Regulation (SCA, Reg II, Nacha, tax regimes, age verification) | |
| Network rules (validity, tolerance, retry limits) | |
| PCI scope (where card data is entered) | |
| Reporting / reconciliation needs | |
| Integration (OMS, POS, KDS, ERP, tax engine, 3PL) | |

## 10. Fit and gaps against the model

| Need | Covered by (resource.field / endpoint / config) | Gap? | Proposed change |
|---|---|---|---|
| | | Yes / No | |

- **Decisions needed.** Name each decision, its options and who owns it.
- **Open questions.**

## 11. Acceptance criteria and sandbox tests

| # | Given / when / then | Sandbox tools (test clock, scenario card, simulator) |
|---|---|---|
| AC1 | | |
| AC2 | | |

## 12. Sign-off

| Role | Name | Date |
|---|---|---|
| Product | | |
| Payments engineering | | |
| Risk / disputes | | |
| Finance / tax (if money or tax changes) | | |
