# Week 14 : A Trust Score that New Users Cannot Game

## Problem

A user with a single 5-star rating should not look more trustworthy than a
user with forty ratings averaging 4.8. Disputes that a user lost should lower
trust, and a verified email should count for something.

## Relevant Context

For each user the database knows their ratings, the number of completed
rentals, the disputes they lost, disputes settled without a clear loser
("neutral"), and whether their email is verified.

## Key Observation

A plain average is unreliable for small counts. A **Bayesian average** pulls
the score towards a prior until enough ratings exist: with a prior of 3.5
worth 5 ratings, one 5-star rating gives (5 + 5 × 3.5) / 6 = 3.75, not 5.

## Solution

Four components, each normalised to 0–5, and a weighted total:

``` python
RATING_PRIOR = 3.5
RATING_PRIOR_COUNT = 5

def calculate_trust_components(*, ratings, transactions, losses, neutral, verified):
    rating, rating_count, average = _rating_component(ratings)   # Bayesian average, scaled to 0–5
    reliability = _clamp(5.0 * (1.0 - (losses + 0.25 * neutral) / (transactions + 5.0)))
    experience = _clamp(5.0 * (1.0 - math.exp(-transactions / 20.0)))
    verification = 5.0 if verified else 0.0
    total = round(0.45 * rating + 0.30 * reliability + 0.15 * experience + 0.10 * verification, 2)
```

| Component | Weight | Behaviour |
|---|---|---|
| rating | 45 % | Bayesian average of 1–5 stars |
| reliability | 30 % | falls with lost disputes; a neutral one counts a quarter |
| experience | 15 % | rises with completed rentals, levelling off (about 3.2 at 20 rentals) |
| verification | 10 % | full marks for a verified email |

The score is recalculated whenever a rating is given or a dispute is decided,
and the profile page shows each component, not only the total.

**Because**

The "+5" in the reliability denominator works like the rating prior: one lost
dispute in a user's first rental does not drop them to zero. Showing the
components explains a low score instead of just displaying it.
