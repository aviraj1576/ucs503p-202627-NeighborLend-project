# Week 7 : Prototype: How the Detection Model Is Evaluated

## Goal

For the prototype report, define how the detection model will be judged, and
what each number means for the listing form.

## Relevant Context

A detection counts as correct when its class is right **and** its box
overlaps the labelled box enough. The overlap is measured by
*Intersection over Union*:

``` text
IoU = area(predicted ∩ true) / area(predicted ∪ true)
```

## Key Observation

The four standard metrics answer different questions:

| Metric | Question it answers | Why it matters here |
|---|---|---|
| Precision | Of the items the model reported, how many were right? | a wrong suggestion has to be corrected by the lender |
| Recall | Of the items in the photos, how many were found? | a missed item only means typing it in |
| mAP@50 | Average precision over all classes, at IoU ≥ 0.5 | overall detection quality |
| mAP@50–95 | The same, averaged over IoU 0.5 to 0.95 | how tight the boxes are |

For the listing form, **precision matters more than recall**: no suggestion
is a small inconvenience, but a wrong suggestion can be published by mistake.

## Solution

- Evaluate on the validation split with `model.val()`, which reports all four
  metrics per class and overall.
- Choose the confidence threshold for the app from the precision–recall
  trade-off, not the trainer's default: a higher threshold gives fewer but
  more reliable suggestions.
- Report per-class results, because a good average can hide one class the
  model hardly ever gets right.

At the prototype stage, the model had been trained and checked on sample
images. As written in the prototype report, the final numerical values were
to be recorded after the final model evaluation.

**Because**

The box quality (mAP@50–95) matters less for this use than the label,
because the app shows the label and not the box. Choosing the threshold by
precision follows directly from how the result is used.
