# Week 14 : A Balanced, Idempotent Ledger for the Simulated Deposit

## Problem

One Razorpay Test Mode payment collects both the rent and the refundable
deposit. Later the deposit is refunded in full, or partly after an approved
deduction. The system must always know where every rupee is, even when a
webhook or an API call is delivered twice.

## Relevant Context

There is no real escrow account. The deposit is tracked as a *simulated*
liability in the application's own tables (`payment_models.py`):

- `RentalBooking`: rent and deposit for one accepted offer;
- `PaymentTransaction`: payments and refunds reported by the gateway;
- `JournalTransaction` and `EscrowLedgerEntry`: double-entry journal lines;
- `RefundIntent`, `RefundAttempt`, `RentalDispute`, `ProcessedWebhook`,
  `FinancialAudit`.

## Key Observation

Two classic payment bugs had to be impossible by construction:

1. **Rounding drift.** Floats such as `0.1 + 0.2` cannot represent money
   exactly.
2. **Double posting.** A retried request or a repeated webhook must not
   record the same money movement twice.

## Solution

**Money as integer paise.** Amounts are converted once, with `Decimal` and
explicit rounding, and stored as integers:

``` python
def paise(value):
    amount = Decimal(str(value))
    if not amount.is_finite() or amount < 0 or amount > Decimal("10000000"):
        raise ValueError("Invalid currency value")
    return int((amount * 100).quantize(Decimal("1"), rounding=ROUND_HALF_UP))
```

**Every journal must balance.** Lines are `(account, side, amount)`, and
total debits must equal total credits. When a payment is captured:

``` python
post(db, booking, "funding:" + booking.id, [
    ("gateway_clearing",    "DEBIT",  booking.total_amount_paise),
    ("rental_fee_clearing", "CREDIT", booking.rental_fee_paise),
    ("escrow_deposit_held", "CREDIT", booking.security_deposit_paise),
], "TEST MODE: payment funding; simulated accounting")
```

The database enforces the same rule with a check constraint:
`CHECK (debit_paise = credit_paise AND debit_paise > 0)`.

**Idempotency through a reference.** Each posting has a unique `reference`
(`"funding:<booking>"`, `"settlement:<booking>"`). If it has already been
posted, `post()` returns the existing journal, but only if the stored lines
are identical; otherwise it raises an error instead of silently recording a
different movement under the same reference:

``` python
# simplified from escrow_service.post()
existing = db.scalar(select(JournalTransaction)
                     .where(JournalTransaction.reference == reference))
if existing:
    if (existing.booking_id != booking.id or existing.debit_paise != total
            or sorted(stored_lines) != sorted(lines)):
        raise ValueError("Journal idempotency reference conflicts with its immutable posting")
    return existing
```

**Because**

The journal is append-only: corrections are new balanced postings, never
edits. The balance of any account can be recomputed from the journal lines,
and a retried operation is harmless.
