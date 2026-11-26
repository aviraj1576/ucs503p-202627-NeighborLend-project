# Week 15 : The Handover and Return Workflow as a Guarded State Machine

## Problem

After an offer is accepted, the item still has to change hands and come
back. Both sides need a record of the item's condition at each step (Week 3),
and neither side should be able to skip a step or act for the other.

## Relevant Context

A rental is a `BorrowRequest` row. Its progress is stored in
`handover_status`, whose allowed values come from an enum:

``` python
class HandoverStatus(str, enum.Enum):
    not_applicable = "not_applicable"
    pending = "pending"
    photos_uploaded = "photos_uploaded"                # lender uploaded item photos
    borrower_confirmed = "borrower_confirmed"          # borrower received the item
    lender_confirmed = "lender_confirmed"              # handover complete, rental active
    issue_reported = "issue_reported"
    return_photos_uploaded = "return_photos_uploaded"  # borrower uploaded return photos
    return_confirmed = "return_confirmed"              # lender approved the return
    disputed = "disputed"                              # lender disputed the return
    dispute_resolved = "dispute_resolved"              # administrator decided
```

## Key Observation

Each transition is allowed for exactly **one role** and only from a fixed set
of **previous states**. If every endpoint checks both before changing
anything, the whole workflow is a state machine enforced by the server, not
by the user interface.

``` text
pending ─lender uploads photos→ photos_uploaded ─borrower confirms→ lender_confirmed (rental active)
        ─borrower uploads return photos→ return_photos_uploaded
        ─lender approves→ return_confirmed      or   ─lender disputes→ disputed → dispute_resolved
```

## Solution

Each endpoint checks the role first (HTTP 403), then the state (HTTP 409),
and only then updates the row and notifies the other party:

``` python
if current_user.id != lender_id:
    raise HTTPException(403, "Only the lender can upload handover photos")
if req.handover_status not in (HandoverStatus.pending.value, HandoverStatus.photos_uploaded.value):
    raise HTTPException(409, "Handover is not in a state to upload photos")
```

| Endpoint | Who | From state |
|---|---|---|
| `POST /handover/upload-photos` | lender | `pending`, `photos_uploaded` |
| `POST /handover/borrower-confirm` | borrower | `photos_uploaded` |
| `POST /handover/report-issue` | borrower | `photos_uploaded`, `borrower_confirmed` |
| `POST /return/upload-photos` | borrower | `lender_confirmed`, `return_photos_uploaded` |
| `POST /return/confirm` | lender | `return_photos_uploaded` |

Once the lender has shared the condition photos, the borrower's confirmation
of receipt completes the handover and starts the rental, so the lender does
not need a separate confirmation step. Handover photos can be uploaded only
after the payment has been verified.

**Because**

403 ("not your action") and 409 ("not now") give the frontend two different
errors to show. Keeping the photos on the same row as the state means the
evidence for each step is stored together with the step itself. The handover
tests in `tests/test_handover.py` exercise each guard.
