# Week 12 : An SVM Intent Classifier for the Negotiation Assistant

## Goal

While negotiating in chat, a user can ask the assistant questions such as
"is ₹200 a day fair for this?", "what should I counter with?" or "when do I
get my deposit back?". The assistant should recognise what is being asked and
answer using the real numbers of that listing and offer.

## Relevant Context

For every conversation the backend already knows the listing's daily price,
the rule-based price suggestion (Week 6), the security deposit, and the
offers exchanged so far. The assistant only has to choose *which* answer to
give and fill in these numbers.

## Key Observation

Choosing the answer is a short-text classification problem. The messages are
short and informal, and often misspelt. Word features alone miss spelling
variants, while character n-grams still match them. A linear SVM on sparse
TF-IDF features is a strong, fast classifier for this kind of input, and it
needs relatively little training data.

## Solution

1. **Intents**: `price_fairness`, `suggest_counter`, `deposit_info`,
   `duration_cost`, `how_it_works` and `other`.

2. **Model**: word and character TF-IDF features feeding a `LinearSVC`:

    ``` python
    from sklearn.pipeline import make_pipeline, make_union
    from sklearn.feature_extraction.text import TfidfVectorizer
    from sklearn.svm import LinearSVC

    model = make_pipeline(
        make_union(
            TfidfVectorizer(ngram_range=(1, 2), sublinear_tf=True),
            TfidfVectorizer(analyzer="char_wb", ngram_range=(3, 5), sublinear_tf=True),
        ),
        LinearSVC(C=1.0),
    )
    model.fit(train_texts, train_intents)
    ```

3. **Low confidence**: `LinearSVC` gives no probabilities, so the largest
   value of `decision_function` is used as the confidence. Below a threshold,
   the message is treated as `other`, and the assistant shows a short menu of
   the topics it can help with instead of guessing.

4. **Numbers come from rules, not from the model.** For `suggest_counter`,
   the reply proposes a value between the last two offers, never below the
   platform minimum (`MIN_DAILY_PRICE_RUPEES = 20` in `constants.py`). The
   assistant never sends an offer itself; the user still has to press
   "Make offer".

**Because**

Separating "understand the question" (the SVM) from "compute the answer"
(deterministic rules) means the classifier can never invent a price. As the
final report states, the assistant supports the user during negotiation but
never makes the rental decision.
