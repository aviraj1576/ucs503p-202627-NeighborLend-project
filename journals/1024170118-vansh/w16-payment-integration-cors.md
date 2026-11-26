# Week 16 : Integrating the Payment Module and Fixing CORS for Vercel

## Part 1 : Merging the Payment Subsystem into the Main Branch

### Problem

The payment and deposit subsystem (Tikshan's journal) was built on its own
branch, `integration/payment-main`. The hardest part of merging it into
`main` was one file, `pages/BorrowRequestDetail.jsx`, which by then rendered
every stage of a rental: handover photos, receipt, return, inspection,
payment and dispute.

### Key Observation

Both branches needed to change this page, each for different stages, but in
the same long file. As long as one page owns every stage, every change to
any stage touches it.

### Solution

The page was split into one component per stage, and the page now only
decides which one to show:

| Component | Stage |
|---|---|
| `PaymentCheckout`, `PaymentBreakdown`, `PaymentStatus` | paying rent and deposit |
| `HandoverConfirmation` | borrower confirms receipt |
| `ReturnSubmission` | borrower uploads return photos |
| `ReturnInspection` | lender reviews the return |
| `EscrowStatusCard`, `DepositRefundStatus` | where the deposit is |
| `DisputeSection` | claims and responses |

This made `BorrowRequestDetail.jsx` several hundred lines shorter, and
`components/payments.test.js` was added to test the payment display logic.

**Because**

Small components with one job each can be changed in parallel without
conflicts, and each can be tested on its own.

## Part 2 : Requests from Vercel Blocked by CORS

### Error

The frontend on Vercel preview deployments could not call the backend. The
browser reported a CORS error, and the backend logs showed no failing
request.

### Relevant Context

The allowed origins were a fixed list (`CORS_ORIGINS`). Every Vercel preview
deployment gets its own sub-domain of `vercel.app`, so the list could never
contain all of them.

### Solution

1. Allow a pattern as well as the fixed list: localhost on any port for
   development, and any sub-domain of `vercel.app`:

    ``` python
    allow_origin_regex=r"^https?://(localhost|127\.0\.0\.1)(:\d+)?$|^https://([a-zA-Z0-9_-]+\.)?vercel\.app$"
    ```

2. Wrap the **whole** application in `CORSMiddleware`, not only the FastAPI
   app's middleware stack:

    ``` python
    _fastapi_app = app
    app = CORSMiddleware(_fastapi_app, allow_origins=settings.cors_origin_list,
                         allow_origin_regex=..., allow_credentials=True,
                         allow_methods=["*"], allow_headers=["*"])
    app.dependency_overrides = _fastapi_app.dependency_overrides
    ```

**Because**

An unhandled exception produces a 500 response outside the normal middleware
stack, without CORS headers. The browser then reports a CORS error and hides
the real 500, which is what made this hard to diagnose. With the outer
wrapper, errors also carry the headers and appear in the console as what
they are. The last line keeps the tests' dependency overrides working
through the wrapper.
