# Week 4 : The Data Model for the Proposal

## Goal

Define the main tables and their relationships for the proposal, so that the
rental can be traced from listing to return and settlement.

## Relevant Context

The proposal names eight entities: User, Item, Booking, Offer, Payment,
Evidence, Dispute and Review. The backend uses SQLAlchemy models with UUID
primary keys.

## Key Observation

Some of the proposed entities are better modelled as *roles* or *columns*
than as separate tables:

- borrower and lender are both `User` rows; the role depends on the rental;
- an "Item" exists only as a listing, so `Listing` is the item;
- a review is one rating in each direction per rental, so it fits as columns
  on the rental instead of a separate table.

## Solution

| Table | Purpose | Key relations |
|---|---|---|
| `users` | account, verification flag, trust score | |
| `listings` | the item, price per day, deposit, location, AI label | `owner_id → users` |
| `listing_photos` | photo URLs, one marked as cover | `listing_id → listings` |
| `conversations` | one chat per borrower, lender and listing | `listing_id → listings` |
| `messages` | chat messages, including system messages | `conversation_id` |
| `negotiation_offers` | offer and counter-offer terms | `conversation_id` |
| `borrow_requests` | the rental: status, handover state, photos, ratings | `listing_id`, `borrower_id` |
| `notifications` | messages to a user, with a link | `user_id` |

Money fields in the later payment tables are stored as integer **paise**
(`BigInteger`), never as floating point.

**Because**

Fewer tables means fewer joins in every rental screen, and the class diagram
in the final report keeps the same scope note: no separate Item, Category or
Escrow table, and AI labels stored on the listing.
