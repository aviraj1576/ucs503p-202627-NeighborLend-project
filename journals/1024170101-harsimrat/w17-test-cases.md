# Week 17 : Designing Test Cases for the Main Rental Workflow

## Goal

Check the main rental workflow end to end on the deployed system
(neighborlend.vercel.app), not only through unit tests, and record the
results in the course's test case format.

## Test Case Format

Each test case follows the course's Test Case Template (Doc:T_01):

- a header: test case number and name, system, subsystem, designer, design
  date and a short description;
- **pre-conditions**, for example "the email is not registered" and "the
  tester can read its inbox";
- numbered **steps**, each with an action, the expected system response, and
  Pass/Fail and Comment columns;
- **post-conditions** to check in the database afterwards, for example "the
  user is saved with a verified email" and "no account is created for the
  second email".

Every case tests the happy path and also failure conditions, such as a wrong
OTP, a wrong CAPTCHA answer, a wrong password or a conflicting booking.

## Test Cases

| # | Test case | Subsystem | Steps |
|---|---|---|---|
| 1.1 | Register with Email OTP Verification | Authentication | 7 |
| 1.2 | Login and Protected Pages | Authentication | 5 |
| 2.1 | Create Listing with AI Photo Detection | Listings | 8 |
| 2.2 | Browse and Search Listings | Listings | 7 |
| 3.1 | In-Chat Negotiation and Booking Conflict | Negotiation | 10 |
| 4.1 | Handover State Machine | Rental | 7 |
| 4.2 | Return and Dispute | Rental | 5 |

## Execution and Results

The cases were run on the live deployment, with Razorpay in Test Mode and a
separate browser session for each user (lender and borrower). Public
Mailinator inboxes were used so that the OTP emails could be read.

On the first run, four cases failed one step each:

| Test case | Failed step | What was fixed |
|---|---|---|
| 2.1 | 7 | Create Listing now shows an error when a non-image file is chosen |
| 2.2 | 6 | Browse says when location is blocked or unavailable, and shows all listings without a distance filter |
| 3.1 | 6 | Accepting an offer notifies both parties and shows a success message |
| 4.1 | 3 | Once the lender has shared the condition photos, the borrower's receipt completes the handover and starts the rental |

After the fixes were deployed, the four cases were re-run in full. All seven
cases passed (49 of 49 steps).

**Because**

Writing the expected response *before* running a step is what made these
failures visible. Three of the four were missing feedback to the user, which
unit tests of the API would not have caught.
