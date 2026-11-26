# Week 12 : Ratings After a Rental, and Notifications for Every Step

## Goal

After a rental, each side rates the other once. And at every step of a
rental, the other party should learn that it is their turn to act.

## Relevant Context

A rental is a `BorrowRequest`. The ratings are stored on it as
`lender_rating` and `borrower_rating` (1–5) with an optional comment, so each
rental can hold at most one rating in each direction.

## Key Observation

A rating is only meaningful once the rental is really over. If a rating were
allowed while a return was still being reviewed, it could be used as a
threat during a disagreement.

## Solution

**Ratings.** `POST /{id}/rate-lender` (borrower only) and
`POST /{id}/rate-borrower` (lender only) apply three checks in order:

``` python
if current_user.id != req.borrower_id:
    raise HTTPException(403, "Only the borrower can rate the lender")
if not rating_eligible(db, req):
    raise HTTPException(409, "Ratings require confirmed return and completed financial settlement")
if req.lender_rating is not None:
    raise HTTPException(409, "Lender already rated for this rental")
```

`rating_eligible` requires a recorded return with photos, a captured
payment, a completed booking and a fully settled deposit (refund processed,
and refund plus deduction equal to the deposit). After saving, the rated
user's aggregate rating and trust score (Week 14) are recalculated.

**Notifications.** One helper creates a notification with a type, a title, a
message and a link to the rental:

``` python
def create_notification(db, user_id, notification_type, title, message,
                        link_url, borrow_request_id=None):
    notification = Notification(user_id=user_id, notification_type=notification_type,
                                title=title, message=message, link_url=link_url,
                                borrow_request_id=borrow_request_id)
    db.add(notification)
    return notification
```

The helper only *adds* the row; the endpoint commits it together with the
action itself, for example
`"{name} rated you 4/5 stars for {title}."` with a link to
`/borrow-requests/{id}`.

**Because**

Because the notification is committed in the same transaction as the action,
a user can never be told about an action that failed and was rolled back.
The link takes them straight to the screen where they must act.
