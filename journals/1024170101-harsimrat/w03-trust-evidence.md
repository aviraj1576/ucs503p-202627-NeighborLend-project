# Week 3 : What Evidence a Rental Needs to Be Trusted

## Goal

Lender and borrower are usually strangers. Before designing any feature,
list what can go wrong in a rental and what record would settle each case.

## Relevant Context

The rental flow from the pitch is: listing → negotiation → booking and
payment → handover → rental period → return → settlement and rating.

## Key Observation

Almost every dispute is about one question: *what was the item's condition
at handover, and what was it at return?* A record made by only one side, or
made after the dispute has started, cannot answer that question fairly.

## Solution

| Risk | Record that settles it | Made by |
|---|---|---|
| "The item was already damaged when I got it" | condition photos at handover, confirmed by the borrower | lender, then borrower |
| "The item came back damaged" | return photos of the same views, reviewed by the lender | borrower, then lender |
| Disagreement about a deduction | the claim, the reply and both photo sets, decided by a third party | administrator |
| Payment taken but item never handed over | handover allowed only after the payment is verified | system |
| One side never reacts | a deadline after which the system takes the default action | system |

Requirements that followed for the design:

1. Each step is confirmed by the **other** party (no one confirms their own
   claim).
2. Evidence is stored with the step it belongs to and cannot be replaced
   after that step is confirmed.
3. A deduction from the deposit is never automatic; a person decides it.

**Because**

Writing the risks first gave a test for every later feature: does it create
or protect one of these records? The handover state machine (Week 15) and the
damage comparison (Weeks 5–8 and 16) both come from this table.
