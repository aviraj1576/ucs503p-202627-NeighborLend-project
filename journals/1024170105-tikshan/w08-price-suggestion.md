# Week 8 : Improvement Plan: A Rule-Based Rental Price Suggestion

## Problem

The proposal planned a regression model (scikit-learn) to suggest a rental
price. When the work started, there was no dataset of rental prices for
items like ours to train it on.

## Relevant Context

The lender already enters two numbers that strongly affect a fair price: the
purchase price and the age of the item in years. The suggestion is only a
starting point that the lender can change.

## Key Observation

A regression model trained on a handful of made-up prices would only learn
those guesses, while looking more authoritative than it is. A transparent
rule, with its reasoning shown to the lender, is more honest until real
rental data exists.

## Solution

`backend/app/services/pricing.py`:

``` python
def calculate_rental_suggestion(purchase_price: float, age_years: float) -> dict:
    """
    - Current value = purchase_price * (0.85 ** age_years)
    - Daily rent = 4% of current value, rounded to the nearest ₹5
    - Refundable security deposit = 20% of current value, rounded to the nearest ₹5
    """
    current_value = round(purchase_price * (0.85 ** age_years), 2)
    daily_rent = float(round_to_nearest_5(current_value * 0.04))
    security_deposit = float(round_to_nearest_5(current_value * 0.20))
    return {"current_value": current_value, "daily_rent": daily_rent,
            "security_deposit": security_deposit,
            "explanation": f"Estimated current value: ₹{current_value:,.2f} ..."}
```

Example: a drill bought for ₹5,000, two years old, has a current value of
5000 × 0.85² = ₹3,612.50, a daily rent of ₹145 and a deposit of ₹725.

Negative inputs are clamped to zero, and the response includes an
`explanation` string that the form shows next to the numbers. The test file
`tests/test_suggest_price.py` checks the rounding and the edge cases.

**Because**

The formula is easy to explain and to check by hand, and the bookings made
on the platform become the data a regression model could later be trained
on. The use-case and data-flow diagrams in the final report describe the
suggestion as rule-based for this reason.
