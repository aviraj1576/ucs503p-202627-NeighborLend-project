# Week 11 : An SVD-Based Recommender for Listings

## Goal

Show each user a short list of listings they are likely to want, based on
what users with similar activity have chatted about and borrowed.

## Relevant Context

NeighborLend has no "like" button, so there are no explicit ratings of
listings. What the database does record is **implicit feedback**:

- a borrower opened a conversation about a listing;
- a borrower sent an offer on it;
- a booking was created for it.

## Key Observation

The user × listing matrix built from these events is very sparse: most users
have interacted with only a few of the listings. Matrix factorisation suits
this case. It describes every user and every listing by a small number of
latent factors, and the product of the factors gives a score even for pairs
that have never interacted. Truncated SVD computes such a factorisation
directly on a sparse matrix.

## Solution

1. Build the matrix *R* with a weight per kind of event. A stronger signal
   gets a larger weight, and repeated events on the same pair add up:

    ``` python
    from scipy.sparse import csr_matrix
    from sklearn.decomposition import TruncatedSVD

    WEIGHTS = {"chat": 1.0, "offer": 2.0, "booking": 3.0}

    def build_matrix(events, users, listings):
        rows = [users[u] for u, _, _ in events]
        cols = [listings[l] for _, l, _ in events]
        vals = [WEIGHTS[kind] for _, _, kind in events]
        return csr_matrix((vals, (rows, cols)), shape=(len(users), len(listings)))
    ```

2. Factorise with *k* components (*k* is kept small because there are few
   users) and reconstruct the scores:

    ``` python
    svd = TruncatedSVD(n_components=k, random_state=0)
    user_factors = svd.fit_transform(R)          # U·Σ
    scores = user_factors @ svd.components_      # ≈ R, zeros filled in
    ```

3. For a user, rank the listings by score, then remove their own listings,
   listings they have already booked, and listings outside their search
   radius. Return the top *N*.

4. **Cold start.** A new user has an empty row, so every score is zero. For
   them, the recommender falls back to the most-interacted listings near
   their location.

**Because**

The reconstruction *U·Σ·Vᵀ* is the best rank-*k* approximation of *R*, so
the filled-in values reflect patterns shared by many users rather than one
user's history. The planned check is to hide one interaction per user and
measure how often it appears in the top *N* (hit rate); as the final report
notes, the relevance of the recommendations still needs a proper evaluation
with more users.
