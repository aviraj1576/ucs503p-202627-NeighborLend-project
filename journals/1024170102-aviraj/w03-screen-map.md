# Week 3 : From Use Cases to a Screen Map

## Goal

Before writing interface code, list every screen the application needs and
give each one a route (URL). The frontend and the backend can then be planned
around the same set of user actions.

## Relevant Context

The use cases from the pitch have four kinds of actor:

- a **guest**, who can look at listings but cannot act on them;
- a **lender**, who lists items and confirms handover and return;
- a **borrower**, who searches, negotiates, pays and returns the item;
- an **administrator**, who reviews disputes.

Lender and borrower are roles of the same registered user, not separate
account types.

## Key Observation

Many use cases act on the same object. "Send offer", "counter offer",
"accept" and "reject" all belong to one conversation about one listing, so
they need one screen, not four. Grouping the use cases by the object they act
on (a listing, a conversation, a borrow request) gives a small set of screens.

## Solution

| Screen | Route | Use cases | Login |
|---|---|---|---|
| Landing | `/` | explain the service | no |
| Register, Verify email, Login | `/register`, `/verify-email`, `/login` | create account, verify OTP, log in | no |
| Browse | `/browse` | search nearby items | no |
| Listing detail | `/listing/:id` | view item, start a chat | no |
| Create listing | `/create-listing` | publish item, upload photos, AI identification, price suggestion | yes |
| My listings | `/my-listings` | edit or remove own items | yes |
| Messages | `/messages` | chat, offer, counter, accept, reject | yes |
| Borrow request | `/borrow-requests/:id` | handover, return, rating | yes |
| My borrow requests | `/my-borrow-requests` | track rentals | yes |
| Notifications | `/inbox` | read notifications | yes |
| Profile | `/profile`, `/users/:userId` | update profile, view trust score | yes |

The administrator screens (`/admin/disputes`) and the damage-check page were
added later, once those workflows existed.

**Because**

One route per object keeps links shareable: a notification can point straight
to `/borrow-requests/<id>`. Deciding early which routes need a login also
fixed where the frontend has to check authentication (Week 5).
