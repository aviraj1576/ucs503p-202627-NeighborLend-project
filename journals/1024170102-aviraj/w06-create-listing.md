# Week 6 : The Create Listing Form with AI Prefill and a Price Suggestion

## Goal

Make listing an item quick: the lender uploads a photo, the category is filled
in from the photo, and a daily rent and deposit are suggested. The lender can
still change everything before publishing.

## Relevant Context

The backend offers two endpoints for this:

- `POST /api/ai/identify` runs the item-detection model on the photo and
  returns predicted labels with confidences;
- `POST /api/listings/suggest-price` takes the purchase price and the age of
  the item, and returns a daily rent and a security deposit.

## Key Observation

Both features are only suggestions. If either endpoint is slow or fails, the
lender must still be able to finish the form. A failed AI call should leave
the fields empty, not block the page.

## Solution

1. The category is prefilled from the best prediction, and the confidence is
   shown as a bar so that the lender can judge it:

    ``` jsx
    <div className="confidence-track">
      <div className="confidence-bar" style={{ width: `${aiData.confidence}%` }} />
    </div>
    <span className="t-small muted">{aiData.confidence}% match</span>
    ```

2. The price suggestion calls the backend. If the call fails, the page applies
   the same rule locally: the value falls by 15 % per year of age, the daily
   rent is 4 % and the deposit 20 % of that value, rounded to the nearest ₹5.

    ``` js
    try {
      const res = await api.post('/listings/suggest-price',
                                 { purchase_price: rawPrice, age_years: rawAge });
      setSuggestion(res.data);
    } catch (err) {
      const currentVal = Math.round(rawPrice * Math.pow(0.85, rawAge) * 100) / 100;
      const dailyRent = Math.round((currentVal * 0.04) / 5) * 5;
      const deposit = Math.round((currentVal * 0.20) / 5) * 5;
      setSuggestion({ current_value: currentVal, daily_rent: dailyRent,
                      security_deposit: deposit });
    }
    ```

3. The inputs are validated before the call: the purchase price must be
   greater than zero, and the age cannot be negative.

**Because**

Showing the confidence, instead of silently filling the field, keeps the
lender in control of what is published. The local fallback must use exactly
the same formula as the backend (`services/pricing.py`); otherwise the
suggestion would change depending on whether the server answered.
