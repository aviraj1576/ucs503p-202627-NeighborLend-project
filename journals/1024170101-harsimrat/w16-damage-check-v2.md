# Week 16 : Damage Check, Second Version: Six Guided Views and an Advisory AI Comparison

## Problem

The classical pipeline (Weeks 5–8) was mostly inconclusive on real photos,
and too heavy for the 512 MB server. The second prototype needed a damage
check that works inside the real rental flow, with photos taken by ordinary
users on their phones.

## Relevant Context

The damage check is a separate module, `backend/damage_check/`, mounted into
the backend. A check belongs to one rental and collects photos in two phases,
`before` (lender, at handover) and `after` (borrower, at return), each with
the same six views: front, back, left, right, top and bottom.

## Key Observation

Most failures of the first version came from the photos, not the algorithm:
different sides of the object, different distances, other things in the
frame. **Guided capture of the same six views** solves most of this before
any comparison runs. Then a vision-language model can compare each
before/after pair of the *same* view, as long as it is told clearly what does
**not** count as damage.

## Solution

1. **Guided six-view capture** in the frontend (`components/damage/`), with a
   client-side quality check before upload. By default all six views are
   required in both phases.

2. **One comparison per view**, at most two at a time, each with a 60 s
   timeout and up to 4 retries on rate-limit or server errors. Photos are
   scaled down to at most 1024 px on the longer side before they are sent.

3. **A conservative prompt.** The model is told that its main job is to avoid
   false alarms, and to report damage only if the defect is clearly visible
   in AFTER, the same spot is clearly intact in BEFORE, and it is physical
   damage rather than a change of light. It answers in JSON:

    ``` json
    {"comparable": true, "damage_detected": false, "confidence": "low",
     "findings": [], "note": "..."}
    ```

    Text inside the photos is treated as image content, never as an
    instruction.

4. **A cautious overall result.** An error never counts as "clear":

    ``` python
    def aggregate_overall(results):
        for r in results:
            if r.comparable and r.damage_detected and r.confidence in ("medium", "high"):
                return "possible_damage"
        for r in results:
            if r.error or not r.comparable or r.damage_detected:
                return "needs_review"
        return "clear"
    ```

    If the AI provider cannot start (for example, no API key), the app still
    runs and every view becomes `needs_review`.

5. **People decide.** The lender reviews the result and approves the return
   or disputes it; only an administrator can approve a deduction. If the
   lender does not review a return within 72 hours, it is approved
   automatically. This deadline is checked whenever the rental is read, with
   the row locked so that two requests cannot both apply it.

**Because**

The AI's output is advice attached to the evidence, never a decision. That
matches the final report's position: the damage comparison gives moderate,
inconsistent results across photographs and conditions, so it must not
authorise any financial deduction on its own.
