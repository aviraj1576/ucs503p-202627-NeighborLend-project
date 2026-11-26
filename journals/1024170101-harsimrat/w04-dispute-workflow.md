# Week 4 : The Damage Claim and Deposit Settlement Workflow for the Proposal

## Goal

Describe, for the proposal, exactly what happens to the security deposit
after an item is returned, including the case where the two users disagree.

## Relevant Context

The borrower pays a refundable deposit together with the rent. After the
return, the lender compares the return photos with the handover photos
(Week 3).

## Key Observation

There are only three outcomes, and each one must end with the whole deposit
accounted for:

1. no damage: the full deposit goes back to the borrower;
2. damage, and the borrower accepts the claim: the agreed amount is
   deducted and the rest is refunded;
3. damage, and the borrower rejects the claim: nothing can be deducted until
   someone independent decides.

## Solution

``` text
            return photos uploaded
                     │
          lender reviews return
           │                  │
      no damage          damage claim (amount + photos)
           │                  │
   refund full deposit   borrower responds
                          │            │
                       accepts      rejects
                          │            │
              deduct agreed amount   dispute: deposit stays held
              refund the rest        until an administrator decides
```

Rules written into the proposal:

- the claimed amount can never be more than the deposit;
- if the borrower rejects the claim, the booking is marked *disputed* and the
  disputed amount stays unresolved until a platform-level decision;
- every decision is recorded with who made it and when.

**Because**

Making "rejected" lead to a held deposit, rather than to either side
winning, removes any reason to reject or claim falsely just to win by
default. In the final system the third outcome is decided by an
administrator who is neither of the two parties.
