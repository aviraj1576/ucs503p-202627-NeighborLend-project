# Week 12 : Webhooks: Verifying Them and Processing Each One Exactly Once

## Problem

If the borrower closes the browser right after paying, the verification call
from Week 11 never arrives, and the booking stays unpaid although the money
was taken. Razorpay solves this with **webhooks**: it calls our backend
directly when a payment or refund changes state. But webhooks can be
retried, delivered more than once, or arrive before our own API call
returns.

## Relevant Context

Each webhook is a POST to `/api/payments/webhook` with a JSON body and two
headers: `X-Razorpay-Signature` (HMAC-SHA256 of the raw body with the
webhook secret) and `X-Razorpay-Event-Id`.

## Key Observation

Two properties are needed:

1. **authenticity**: the signature must be checked on the *raw* bytes, before
   the JSON is parsed;
2. **idempotency**: the same event id must never be applied twice, and must
   never be reused for a different payload.

## Solution

1. Read the body with a size limit, then check the event id and the
   signature:

    ``` python
    raw = bytearray()
    async for chunk in request.stream():
        if len(raw) + len(chunk) > WEBHOOK_MAX_BODY_BYTES:
            raise HTTPException(413, "Webhook too large")
        raw.extend(chunk)
    if (not event_id or len(event_id) > 128
            or not provider.verify_webhook(bytes(raw), request.headers.get("x-razorpay-signature"))):
        raise HTTPException(400, "Invalid webhook signature or identifier")
    ```

2. Accept only known event types (`payment.captured`, `payment.failed`,
   `order.paid`, `refund.processed`, `refund.failed`, …).

3. Record a **receipt** keyed by the event id, with the SHA-256 of the body,
   and commit it *before* acting on the event:

    ``` python
    digest = hashlib.sha256(raw).hexdigest()
    previous = db.get(ProcessedWebhook, event_id)
    if previous and previous.payload_sha256 != digest:
        raise HTTPException(409, "Webhook identifier reused with another payload")
    if previous and previous.status == "processed":
        return {"received": True, "duplicate": True}
    ```

4. Never trust amounts in the webhook. For a refund the backend does not
   know yet, it fetches the refund from Razorpay and uses that record.

`refund.created` is only recorded: a refund being *created* is not evidence
that the money was returned.

**Because**

The signature is computed over the exact bytes Razorpay sent; parsing and
re-serialising the JSON could change them. Storing the payload hash with the
event id turns a replay into a harmless "duplicate", and a forged reuse of
an old id with different contents into a 409.
