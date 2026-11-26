# Week 11 : Razorpay Test Mode: Creating an Order and Verifying the Checkout

## Goal

Let the borrower pay the rent and the refundable deposit together through
Razorpay, in **Test Mode** only, and mark the booking as funded only when the
payment is really captured.

## Relevant Context

The Razorpay flow has three steps:

1. the backend creates an **order** for the exact amount;
2. the browser opens Razorpay Checkout for that order;
3. Checkout returns `razorpay_order_id`, `razorpay_payment_id` and
   `razorpay_signature` to the browser, which sends them to the backend.

## Key Observation

Everything returned to the browser can be changed by the user. The backend
must check the signature with its own secret, **and** then ask Razorpay
directly for the order and the payment, before trusting either.

## Solution

1. **Order.** Only the borrower can create it, only for an accepted offer,
   and only while the booking is `awaiting_payment`. The amount is the
   booking's own total in paise, never a value from the request.

2. **Signature.** Razorpay signs `order_id|payment_id` with HMAC-SHA256 using
   the key secret:

    ``` python
    def verify_checkout(self, order_id, payment_id, signature):
        expected = hmac.new(self._secret.encode(),
                            f"{order_id}|{payment_id}".encode(),
                            hashlib.sha256).hexdigest()
        return (isinstance(signature, str)
                and bool(re.fullmatch(r"[a-f0-9]{64}", signature))
                and hmac.compare_digest(expected, signature))
    ```

3. **Authoritative check.** After the signature, the order and the payment
   are fetched from Razorpay, and every field must match the booking:

    ``` python
    if (order.get("id") != order_id
            or order.get("receipt") != reference
            or order.get("amount") != amount
            or order.get("currency") != "INR"
            or order.get("livemode") is True
            or payment.get("id") != payment_id):
        conflict("Authoritative provider data does not match booking")
    ```

4. **Test keys only.** The provider refuses to start unless the key id
   matches `rzp_test_…`, so a live key can never be used by mistake.

All of this runs with the booking row locked (`SELECT … FOR UPDATE`), so two
verifications of the same booking cannot run at the same time.

**Because**

A valid signature proves that Razorpay issued these ids, but not that they
belong to *this* booking with *this* amount. Fetching the order and comparing
its `receipt` (our own reference) and amount closes that gap.
