# Week 11 : Preventing Double Booking of the Same Item

## Problem

Two borrowers negotiating for the same item at the same time could both get
an accepted offer for overlapping dates. The item can only be in one place.

## Relevant Context

An accepted offer becomes a borrow request with `needed_from` and
`needed_until` dates. While the item is out, the request may also have an
actual start (`borrowing_started_at`) and a due date (`return_due_at`). The
check runs whenever an offer is made, countered or accepted.

## Key Observation

Two date intervals [S1, E1] and [S2, E2] overlap exactly when
**S1 ≤ E2 and E1 ≥ S2**. This one condition covers every case (one inside the
other, partial overlap at either end, identical ranges), so no case-by-case
logic is needed.

## Solution

`services/negotiation.py` checks both kinds of existing booking with that
condition, directly in SQL:

``` python
conflict_offer = (db.query(NegotiationOffer)
    .filter(NegotiationOffer.listing_id == listing_id,
            NegotiationOffer.status == OfferStatus.accepted,
            NegotiationOffer.start_date <= end_date,
            NegotiationOffer.end_date >= start_date)
    .first())

conflict_borrow = (db.query(BorrowRequest)
    .filter(BorrowRequest.listing_id == listing_id,
            BorrowRequest.status == BorrowRequestStatus.fulfilled,
            BorrowRequest.returned_at.is_(None),
            func.coalesce(BorrowRequest.borrowing_started_at, BorrowRequest.needed_from) <= end_date,
            func.coalesce(BorrowRequest.return_due_at, BorrowRequest.needed_until) >= start_date)
    .first())
```

When a conflict exists, the user gets the reason with the dates, for example
*"Item is already booked from Nov 20, 2026 to Nov 22, 2026."* The offer being
countered is excluded from the check, so a counter-offer does not conflict
with itself.

The rental cost uses one rule everywhere: the number of days is
`end − start`, a same-day rental counts as one day, and an end date before
the start date is rejected.

**Because**

`coalesce` uses the real dates once a rental has started and the planned ones
before that, so a late return keeps blocking new bookings until the item is
actually back (`returned_at` is set).
