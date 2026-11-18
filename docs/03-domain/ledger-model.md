# Ledger Model

> Status: v1.0. Virtual-currency wallets (D7) on a double-entry ledger (D54). Booking states are defined in the [booking lifecycle](booking-lifecycle.md). Decisions Dxx live in [Product Vision §9](../01-product/vision-and-purpose.md#9-decisions-log).

## 1. Principles

- **Double-entry:** every LedgerTransaction has ≥2 entries whose amounts sum to zero (NFR-105).
- **Immutable:** entries are never edited or deleted; corrections are compensating transactions (NFR-902).
- **Balances are derived:** an account balance is the sum of its entries; a stored balance is only a cache that must always equal that sum (NFR-105, NFR-110).
- **One business operation = one ledger transaction**, committed atomically with the booking state change it belongs to (NFR-103, NFR-104).
- **Refunds are always funded:** upfront payments stay in the merchant's Pending account until no full refund is possible (D55).
- **Provider boundary:** all movement in or out of the system goes through the Virtual-funding account, which a real payment provider's clearing account can replace later (FR-907, NFR-1302).

## 2. Wallets and accounts

| Wallet | Account | Meaning | May go negative |
|---|---|---|---|
| Customer | Available | Spendable funds | No |
| Customer | Reserved | Frozen for no-show fees and wallet-settled remainders of active bookings (D14, D57) | No |
| Merchant | Available | Withdrawable funds | No |
| Merchant | Pending | Upfront payments of bookings that may still be fully refunded (D55) | No |
| System | Virtual funding | Counterpart of top-ups and withdrawals; balance = −(total money in the system) | Yes (only this one) |
| System | Platform fees | Reserved for a future platform fee (D15, FR-906); unused in the MVP | No |

Displayed values (FR-903):

| Wallet | balance | reserved / pending | available |
|---|---|---|---|
| Customer | Available + Reserved | Reserved | Available |
| Merchant | Available + Pending | Pending | Available |

## 3. Ledger entities

| Entity | Attributes |
|---|---|
| Wallet | id, owner (customer_id or merchant_id), currency, created_at |
| LedgerAccount | id, wallet_id, kind (available · reserved · pending · funding · fees) |
| LedgerTransaction | id, type, booking_id?, idempotency_key (unique), actor, reason, created_at (UTC) |
| LedgerEntry | id, transaction_id, account_id, amount (signed, minor units), purpose (upfront · no_show_fee · remainder · funds) |

The `purpose` tag lets each booking's money state be derived from the ledger alone: what was paid, reserved, refunded and captured, per component (NFR-904).

## 4. Money flows per booking operation

Notation: U = upfront amount, F = no-show fee, R = wallet-settled remainder (0 if settled at venue), k = late-cancellation kept %. All values come from the booking's terms snapshot. CA/CR = customer Available/Reserved, MA/MP = merchant Available/Pending, VF = Virtual funding.

| Op | Type | Trigger | Entries | Ref |
|---|---|---|---|---|
| W1 | top_up | Customer deposits X | VF −X; CA +X | FR-902 |
| W2 | withdrawal | Customer or Owner withdraws X ≤ available | CA (or MA) −X; VF +X | FR-902, FR-904 |
| M1 | checkout | Held → Confirmed / PendingApproval | CA −U; MP +U; CA −(F+R); CR +(F+R) | FR-504, D11 |
| M2 | full_refund | Reject, auto-reject, cancel in window, merchant cancel, PendingApproval cancel (D60) | MP −U; CA +U; CR −(F+R); CA +(F+R) | FR-509, FR-603, FR-604, FR-801 |
| M3 | late_cancel | Customer cancels after free window | MP −U; MA +⌊k·U⌋; CA +(U − ⌊k·U⌋); CR −(F+R); CA +(F+R) | FR-509 |
| M4 | complete | Completed (manual or auto) | MP −U; MA +U; CR −R; MA +R; CR −F; CA +F | FR-605, FR-608, D57 |
| M5 | no_show | NoShow | MP −U; MA +U; CR −F; MA +F; CR −R; CA +R | FR-605, D58 |

- Booking operations with no money effect (Provisional, verification, approval, reschedule, move/extend, Expired, guest and merchant-created bookings) create no ledger transaction.
- A booking with U = F = R = 0 never touches the ledger.
- **Rounding:** % amounts are computed once at booking time, in minor units, rounded half-up, and stored in the terms. The kept share of a late cancellation is rounded **down** (in the customer's favour).
- Checkout fails with no effect if CA < U + F + R; the hold remains until expiry (FR-504).

## 5. Worked example

Service price 100.00; deposit 20%; no-show fee 15.00; remainder via wallet. Customer's available balance 200.00. U = 20.00, F = 15.00, R = 80.00.

| Step | CA | CR | MP | MA |
|---|---|---|---|---|
| Start | 200.00 | 0 | 0 | 0 |
| M1 checkout | 85.00 | 95.00 | 20.00 | 0 |
| M4 complete | 100.00 | 0 | 0 | 100.00 |
| (alternative) M5 no-show | 165.00 | 0 | 0 | 35.00 |
| (alternative) M3 late cancel, k = 50% | 190.00 | 0 | 0 | 10.00 |

Each row conserves money: CA + CR + MP + MA = 200.00 throughout.

## 6. Invariants (checked continuously and by nightly reconciliation, NFR-110)

1. Every transaction sums to zero.
2. No account except Virtual funding is ever negative.
3. Cached balances equal the sum of entries.
4. Customer Reserved = Σ (F + R) of that customer's bookings in PendingApproval or Confirmed.
5. Merchant Pending = Σ U of that merchant's bookings in PendingApproval or Confirmed.
6. Every booking in a terminal state has a net zero position in CR and MP.
7. Σ of all non-system accounts = −(Virtual funding balance).

## 7. Concurrency and idempotency

- Each wallet is a linearizable unit (NFR-102): debits check the available balance in the same transaction that writes the entries (no lost update, no overdraft).
- Every ledger transaction has a unique idempotency key derived from the command (e.g. booking id + operation); a retry returns the existing transaction (NFR-106).
- Booking money operations write the booking state change and the ledger transaction in one atomic unit; the cross-aggregate atomicity mechanism is an architecture decision for [05-architecture](../05-architecture/).

## 8. Extensibility

- **Platform fee:** add entries crediting Platform fees in M4/M5/M3 (e.g. MA +U·(1−fee), PF +U·fee) — no model change (FR-906).
- **Real payments:** Virtual funding becomes the PSP clearing account; top-ups/withdrawals become provider-backed with their own pending states (FR-907).
- **Multi-currency:** wallets already carry a currency; a transaction then must balance per currency.

## 9. Change log

| Version | Date | Change |
|---|---|---|
| v0.9 | 2026-11-12 | First draft (D54–D58) |
| v1.0 | 2026-11-12 | D60 applied |
