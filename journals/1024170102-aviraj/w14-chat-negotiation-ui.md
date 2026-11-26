# Week 14 : The Chat Page: Messages and Rental Offers in One Place

## Goal

Let a borrower and a lender talk about one listing and agree on terms: make
an offer, counter it, accept it or reject it, without leaving the
conversation.

## Relevant Context

The backend endpoints used by the page:

| Action | Endpoint |
|---|---|
| send a message | `POST /api/conversations/{id}/messages` |
| make an offer | `POST /api/conversations/{id}/offers` |
| counter an offer | `POST /api/offers/{offer_id}/counter` |
| accept / reject | `POST /api/offers/{offer_id}/accept`, `/reject` |

An offer contains the listing, a daily price, start and end dates, and an
optional note.

## Key Observation

There are no WebSockets in the backend, so a new message or offer from the
other user only appears when the page asks for it. Polling every few seconds
works, but polling a tab that nobody is looking at wastes requests.

## Solution

1. **Visible-only polling**: refresh every 5 seconds, and also immediately
   when the tab becomes visible or gets focus again:

    ``` jsx
    const refreshWhenVisible = () => {
      if (document.visibilityState === 'visible') fetchActiveConversation(conversationId, true);
    };
    const interval = window.setInterval(refreshWhenVisible, 5000);
    window.addEventListener('focus', refreshWhenVisible);
    document.addEventListener('visibilitychange', refreshWhenVisible);
    ```

2. **One form for offer and counter.** The same form is used for both; only
   the endpoint changes:

    ``` jsx
    if (negotiationMode === 'counter' && counterTargetOffer) {
      await api.post(`/offers/${counterTargetOffer.id}/counter`, payload);
    } else {
      await api.post(`/conversations/${activeConv.id}/offers`, payload);
    }
    ```

3. **Accept is confirmed and then navigates.** Accepting an offer creates the
   borrow request, so the page asks for confirmation first, then opens the new
   request with a notice that the handover is pending.

4. **Server messages are shown as they are.** If the backend rejects an offer
   (for example, the dates overlap an existing booking), its `detail` text is
   shown under the form instead of a generic error.

**Because**

The backend, not the page, decides whether an offer is valid. Showing its
message directly means the user sees the real reason ("the item is already
booked for these dates") and the frontend does not repeat the rules.
