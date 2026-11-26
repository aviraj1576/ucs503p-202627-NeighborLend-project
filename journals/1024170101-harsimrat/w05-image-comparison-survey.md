# Week 5 : Why a Simple Pixel Difference Cannot Detect Damage

## Goal

Find a way to compare a handover photo with a return photo of the same item
and point out possible new damage, as an aid for the lender and the
administrator.

## Relevant Context

The two photos are taken by different people, on different phones, days
apart, in different places. Nothing about the camera position, the lighting
or the background is controlled.

## Key Observation

Subtracting the two images (`|before − after|`) marks almost everything as
changed, for reasons that have nothing to do with damage:

| Cause | Effect on a pixel difference |
|---|---|
| a slightly different angle or distance | every edge moves, so every edge is "changed" |
| different lighting or white balance | the whole object changes brightness |
| a different background | large changed areas around the object |
| glare on glossy surfaces | bright patches that look like scratches |
| blur or low resolution | detail missing in one photo only |

A useful comparison therefore needs, in this order: photos good enough to
compare, the object separated from the background, the two views aligned
geometrically, and only then a comparison that tolerates lighting changes.

## Solution

The pipeline chosen for the next weeks, built with OpenCV and NumPy:

1. **Quality gate**: reject blurred, dark, over-exposed or low-resolution
   photos and ask for a recapture (Week 6).
2. **Alignment**: match local features between the two photos and estimate a
   geometric transform with RANSAC (Week 7).
3. **Comparison**: structural similarity (SSIM) over the aligned object
   area, which compares local structure rather than raw brightness (Week 7).
4. **Output**: *candidate* regions for a person to look at, never a verdict.

**Because**

Each stage removes one cause of false alarms before the next stage runs. If
a stage fails, the result is "inconclusive", which is honest, instead of a
difference map full of false damage.
