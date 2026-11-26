# Week 8 : Improvement Plan: Object Masks, an Identity Check, and Honest Results

## Problem

The prototype comparison (Week 7) assumed that both photos show the same
object and that the object mask is right. With real photos, the background,
hands holding the item, or a different item could produce many false
candidates.

## Relevant Context

Two small pretrained models were added in front of the geometric stage, both
used only for inference (no training):

- **EfficientSAM-Ti** segments the object inside a target box, giving the
  object mask;
- **DINOv2-S/14** turns the masked object into an embedding vector. Two
  photos of the same object should have similar embeddings (cosine
  similarity close to 1).

The pipeline is: quality gate → masks → embeddings → SIFT/RANSAC alignment →
masked SSIM candidates.

## Key Observation

An identity check placed *before* alignment stops the pipeline early when the
two photos may not show the same thing. A pair whose cosine similarity is
below **0.65** is reported as `embedding_inconsistent`, and no difference map
is produced for it.

## Results on Real Photos

Three real photo pairs were run through the full pipeline on a GPU
(two drill pairs and one headphone pair):

| Pair | Outcome | Cosine similarity |
|---|---|---|
| drill 1 → drill 2 | inconclusive | 0.493 |
| drill 1 → drill 3 | inconclusive | 0.462 |
| headphones 1 → 2 | inconclusive | 0.453 |

All three stopped at the identity check, so alignment and SSIM were never
reached. Looking at the masks explained part of this: one drill mask also
included the fingers holding the tool, and one headphone mask contained
detached pieces of background. Each run took about 13–16 seconds and around
1.5 GB of memory in the worker process.

## Conclusion for the Improvement Plan

- The classical pipeline is careful (it does not invent damage), but with
  uncontrolled photos it is mostly **inconclusive**.
- Results depend strongly on how the photos are taken. Guided capture of
  fixed views, against a plain background, would help more than any
  threshold change.
- Its run time and memory are too high for the free 512 MB server.

These findings led to the second version in Week 16: guided capture of six
fixed views, and a vision-language model used as an *advisory* comparison.

**Because**

Reporting "inconclusive" with the reason is more useful than a confident
wrong answer, because a deposit deduction depends on it. Lowering the 0.65
threshold until the pairs "pass" would have hidden the real problem, which
was the photos and the masks.
