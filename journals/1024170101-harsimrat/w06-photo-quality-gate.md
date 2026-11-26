# Week 6 : A Quality Gate for Condition Photos

## Problem

A blurred or badly lit photo makes any later comparison meaningless: a
blurred return photo hides scratches, and an over-exposed one looks like
damage. Such photos must be caught when they are uploaded, while the user can
still take another one.

## Relevant Context

Each photo arrives as an RGB image. An optional mask marks the pixels that
belong to the object, so the background does not affect the measurements.

## Key Observation

Each common problem has a simple, cheap measurement:

| Problem | Measurement |
|---|---|
| blur, or no texture | variance of the Laplacian (few strong edges → low variance) |
| low contrast | standard deviation of the grey values |
| under- or over-exposure | fraction of pixels below 10 or above 245 |
| too small | shorter side below a minimum |
| object cut off | the object mask touches the image border |

## Solution

``` python
def assess_quality(evidence_id, rgb, config, mask=None):
    gray = cv2.cvtColor(rgb, cv2.COLOR_RGB2GRAY)
    values = gray[mask] if mask is not None and mask.any() else gray.ravel()
    laplacian = cv2.Laplacian(gray, cv2.CV_64F)
    focus = float(laplacian[mask].var() if mask is not None and mask.any() else laplacian.var())
    dark, bright = float((values < 10).mean()), float((values > 245).mean())
    reasons = []
    if min(gray.shape) < config.min_side:
        reasons.append("low_resolution")
    if focus < config.min_focus:
        reasons.append("blur_or_no_texture")
    if values.std() < config.min_contrast:
        reasons.append("low_contrast")
    if dark > config.max_clipped_fraction:
        reasons.append("underexposed")
    if bright > config.max_clipped_fraction:
        reasons.append("overexposed")
    ...
```

Each reason comes with a concrete instruction for the user, for example
*"Reduce glare and exposure; avoid direct flash on reflective surfaces"* for
`overexposed`. A photo with any reason is marked `needs_recapture`.

**Because**

The Laplacian responds to edges, so a sharp photo has many large values and a
high variance, while blur spreads edges out and lowers it. Telling the user
*why* a photo was rejected, and how to retake it, turns a rejection into a
better second photo. The instruction text also warns against sharpening or
upscaling a photo just to pass the check.
