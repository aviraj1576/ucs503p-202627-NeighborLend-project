# Week 15 : Refunds: Authorising a Settlement Before Any Money Moves

## Problem

At the end of a rental the deposit goes back to the borrower, in full or
minus an approved deduction. A refund is the one operation that sends money
out, so a bug or a double click must never refund the wrong amount, refund
twice, or deduct money that nobody approved.

## Relevant Context

The ledger (Week 14) records where the deposit is. A refund is made through
Razorpay Test Mode against the original payment. Deductions come from a
damage claim, which the two users may dispute.

## Key Observation

"Deciding how much to refund" and "sending the refund" are two different
steps, done by different people, at different times. Keeping them apart, with
a stored record between them, makes each one checkable on its own.

## Solution

1. **A settlement intent** records the decision. There is at most one per
   booking, the deduction must be a whole number of paise between zero and
   the deposit, and repeating the same decision returns the existing intent:

    ``` python
    existing = db.scalar(select(RefundIntent).where(RefundIntent.booking_id == booking.id))
    if existing:
        if existing.kind != kind or existing.deduction_paise != deduction:
            conflict("Settlement is already authorized with different terms")
        return existing
    if (booking.payment_status != "captured" or type(deduction) is not int
            or not 0 <= deduction <= booking.security_deposit_paise):
        conflict("Invalid settlement allocation")
    amount = booking.security_deposit_paise - deduction
    ```

2. **Deductions in a dispute need an independent administrator**, a written
   reason, and recorded return evidence:

    ``` python
    if not admin or actor in {booking.borrower_id, booking.lender_id}:
        raise HTTPException(403, "An independent human administrator must review deductions")
    if not reason.strip():
        raise HTTPException(422, "Administrator resolution requires a reason")
    ```

3. **Executing the refund** recomputes the expected amount from the booking
   and refuses to continue if the stored intent does not match it
   (`"Refund allocation integrity failed"`). A refund already `dispatching`
   must be reconciled with Razorpay before a retry.

4. **An allow-list of gateway operations.** The Razorpay adapter can only
   create orders, read orders, payments and refunds, and create a normal
   refund. Any other path is rejected even when called from Python code:

    ``` python
    # This adapter has no capture, payout, transfer, account or settlement
    # endpoint. Reject unsupported paths even when called from Python code.
    raise ProviderError("unsupported_provider_operation")
    ```

Every step runs with the booking row locked and writes a `FinancialAudit`
record with the actor.

**Because**

The intent is the single source of truth for the amount: the execution step
does not take an amount from anyone, it only checks and carries out the
recorded decision. As the final report states, AI damage results are only
advice; a deduction always needs a person who is not one of the two parties.
