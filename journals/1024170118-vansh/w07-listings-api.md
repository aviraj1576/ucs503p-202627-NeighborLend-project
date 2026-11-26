# Week 7 : The Listings API for the Prototype

## Goal

For the prototype, connect the Create Listing and Browse screens to real
endpoints: create a listing with photos, edit it, and keep other users from
changing it.

## Relevant Context

A listing has a name, category, description, daily price, deposit, location,
an optional AI label with its confidence, and one or more photo URLs. Its
status is one of `draft`, `published`, `paused` or `removed`.

## Key Observation

Three rules must hold no matter what the frontend sends:

1. the owner is the logged-in user, never a value from the request body;
2. only the owner can edit a listing;
3. a listing that is currently being rented must not change its terms.

## Solution

**Create.** The owner comes from the token, and the first photo becomes the
cover:

``` python
listing = Listing(owner_id=current_user.id, name=data.name, ...,
                  status=ListingStatus.published if data.publish else ListingStatus.draft)
db.add(listing)
db.flush()                                   # assigns listing.id
for i, url in enumerate(data.photo_urls):
    db.add(ListingPhoto(listing_id=listing.id, url=url, is_cover=(i == 0)))
db.commit()
```

**Update.** The owner check is part of the query, and status changes follow
an explicit table:

``` python
listing = (db.query(Listing)
             .filter(Listing.id == listing_id, Listing.owner_id == current_user.id)
             .with_for_update().first())
if not listing:
    raise HTTPException(404, "Listing not found or unauthorized")
if has_content_changes and _listing_is_in_use(db, listing.id):
    raise HTTPException(409, "This listing cannot be edited while it is being rented.")

allowed_transitions = {
    ListingStatus.draft: {ListingStatus.published},
    ListingStatus.published: {ListingStatus.paused},
    ListingStatus.paused: {ListingStatus.published},
}
```

**Because**

Filtering by `owner_id` in the query answers "not found" for someone else's
listing, which does not reveal whether it exists. `with_for_update()` locks
the row, so two simultaneous edits cannot both pass the "in use" check. The
explicit transition table makes an invalid status change (for example,
`removed` back to `published`) a clear 409 instead of a silent update.
