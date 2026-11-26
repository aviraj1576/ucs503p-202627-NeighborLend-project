# Week 7 : Prototype: Aligning Two Photos and Comparing Them with SSIM

## Problem

For the prototype, compare a handover photo and a return photo of the same
item and mark regions that may have changed, without marking the normal
differences listed in Week 5 (angle, lighting, background).

## Relevant Context

Both photos have passed the quality gate (Week 6), and each has a mask of the
object. The two photos are never taken from exactly the same position.

## Key Observation

Before two photos can be compared pixel by pixel, the "before" photo must be
warped so that each point of the object lands where it is in the "after"
photo. This needs reliable point correspondences, and a single wrong match
can spoil the transform unless the estimate is robust to outliers.

## Solution

**1. Local features (SIFT) on the object only**, with contrast-limited
histogram equalisation (CLAHE) to reduce lighting differences, and RootSIFT
normalisation of the descriptors:

``` python
detector = cv2.SIFT_create(nfeatures=1200, enable_precise_upscale=True)
gray = cv2.createCLAHE(clipLimit=2.0, tileGridSize=(8, 8)).apply(gray)
keypoints, descriptors = detector.detectAndCompute(gray, mask.astype(np.uint8) * 255)
descriptors = np.sqrt(descriptors / np.maximum(descriptors.sum(axis=1, keepdims=True), 1e-8))
```

**2. Mutual matching with Lowe's ratio test**: a match is kept only if it
passes the 0.75 ratio test in *both* directions (before → after and
after → before).

**3. RANSAC homography**, with a fallback to a simpler similarity transform
under the same checks. At least 12 inliers and an inlier ratio of 0.45 are
required; otherwise the pair is reported as
`weak_geometric_support_or_view_change`.

``` python
matrix, inliers = cv2.findHomography(src, dst, cv2.RANSAC, 3.0)
```

**4. SSIM over the aligned overlap.** A global gain and offset correct the
overall brightness first; then a Gaussian-window SSIM map is computed, and a
pixel is a change candidate only if both the SSIM difference and the
absolute grey difference are large. Connected regions are kept if their area
is within limits, and the 20 largest are returned:

``` python
difference = np.clip((1 - ssim_map(adjusted, gray2)) / 2, 0, 1)
proposals = (difference > config.difference_threshold) & (np.abs(adjusted - gray2) > 0.06) & overlap
```

A morphological *closing* (not an opening) joins broken pieces, because an
opening would erase the one- or two-pixel-wide lines that thin scratches and
cracks produce.

**Because**

RANSAC fits the transform to random small subsets and keeps the one most
points agree with, so wrong matches do not pull it away. SSIM compares local
mean, contrast and structure, so a uniform lighting change affects it far
less than a scratch does. Every step can stop the pipeline with a named
reason, and the output is a set of *candidates* for a person to review.
