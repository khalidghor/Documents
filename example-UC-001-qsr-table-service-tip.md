# Order use case — `UC-001`: QSR table service with tip added after authorization

> A filled-in example of [order-use-case-template.md](order-use-case-template.md). It is deliberately short and shows the level of detail expected.

## 1. Summary

| Field | Value |
|---|---|
| Use case ID | `UC-001` |
| Name | Table service, card dipped at open, tip on receipt |
| Industry profile | `qsr` |
| Channel(s) | `pos` with `terminal` |
| Customer present? | CIT (card present) |
| Markets and currencies | US, USD |
| Priority | Must |
| Status | Accepted |

**One-sentence story.** As a restaurant, I want to authorize the card when the check opens and capture the final amount including the tip when it closes, so that I never lose a tip or hold the wrong amount.

**Why it matters.** Table service is a large share of restaurant volume. Losing tips or making a second charge for the tip are common sources of complaints.

## 2. Parties

Buyer: a guest. Merchant: the restaurant (single merchant). Merchant of record: the restaurant, with `fund_flow: single_merchant`. Staff: the server, who receives the tip.

## 3. What is being sold

| Line | `type` | SKU | Qty | Modifiers | Fulfillment group | Tax code |
|---|---|---|---|---|---|---|
| li_1 | product | BURGER | 2 | No onions (one of the two) | fg_table | food_prepared |
| li_2 | product | IPA-PINT | 2 | — | fg_table | alcohol |
| li_3 | product | BROWNIE | 1 | added later | fg_table | food_prepared |

- **Tip:** added on the receipt (`added_at: receipt`), paid to the server (`recipient: staff`).
- **Tax:** exclusive, US-XX; the merchant is liable.

## 4. Fulfillment

`fg_table` · `dine_in` · fulfilled by the merchant · `capture_trigger: on_close`.

## 5. How it is paid

| Field | Value |
|---|---|
| Strategy | `authorize_then_capture_on_close` |
| Methods | Card (EMV/contactless) |
| Payment Intents | One per check; one per guest if the bill is split |
| `capture_method` / authorization type | `manual` / `estimated` |
| `partial_authorization` | `accept_and_collect_remainder` |
| Expiry | n/a — the check closes the same day |
| Overcapture tolerance | 20% for tips under MCC 5812 (to be confirmed per network) |

## 6. Happy path

| # | Actor | Action | Order | payment_status | fulfillment | Intent | Operation | Money |
|---|---|---|---|---|---|---|---|---|
| 1 | Server | Open check, dip card: `POST /v1/orders` (place) | confirmed | authorized | unfulfilled | authorized | authorize $64.20 | — |
| 2 | Server | Add brownie: `POST /amend` | confirmed | authorized | unfulfilled | authorized | incremental $8.50 (if needed) | — |
| 3 | Guest | Writes $14.00 tip; server closes: `POST /close` | completed | collected | fulfilled | captured | capture $86.70 | Tip credited to `tips:payable:staff` |

## 7. Exceptions (selected)

- ☑ **Tip above tolerance.** Do an incremental authorization first. If that is declined, capture within tolerance and collect the rest by a second tender.
- ☑ **Partial authorization** on a prepaid card. The remainder is collected by a second tender on the same order.
- ☑ **Guest walks out before close.** Capture the authorized amount without a tip after the configured hold, then `close`.
- ☑ **Dispute "not as described".** The merchant is liable. Auto-assembled evidence: itemised receipt and card-present EMV data.

## 10. Fit and gaps

| Need | Covered by | Gap? |
|---|---|---|
| Tip after authorization | `tips[].added_at = receipt`, overcapture tolerance in `payment-lifecycle-policy.yaml` | No |
| Server tip payout | `tips:payable:staff` in the commerce ledger | **Yes.** There is no tip-pool distribution report yet. Proposed: `GET /v1/reports/tips` by staff and shift |

## 11. Acceptance criteria

| # | Given / when / then | Sandbox |
|---|---|---|
| AC1 | Given an authorization of $64.20, when the check closes with a $14.00 tip and a $8.50 add-on, then one capture of $86.70 succeeds and no second charge exists | Scenario card `approve` |
| AC2 | Given a tip of 30%, when the check closes, then an incremental authorization is attempted before capture | Scenario card `incremental_declines` |
