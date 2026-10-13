---
layout: post
title: "Inspecting Object Size Using OpenCV"
date: 2026-10-13 14:15:00 +0900
categories: [image-processing, opencv]
tags: [opencv, contours, minAreaRect, size-inspection, quality-inspection]
description: "Measure the long and short sides of a rotated rectangle with OpenCV and detect oversized or undersized objects using a pixel tolerance."
---

<a id="post-top"></a>

In Post #20, I checked whether an object was in the correct position and within the allowed rotation range. An object can pass both checks and still have the wrong size.

In this post, I measure the long and short sides of a rotated rectangular object and compare them with reference dimensions. The example checks one normal object, one object with an oversized long side, and one object with an undersized short side.

## Table of Contents

- [Learning Objectives](#learning-objectives)
- [Inspection Criteria](#inspection-criteria)
- [Measuring a Rotated Rectangle](#measuring-a-rotated-rectangle)
- [Calculating Size Errors](#calculating-size-errors)
- [Full Source Code](#full-source-code)
- [Running the Example](#running-the-example)
- [Results and Interpretation](#results-and-interpretation)
- [Practical Considerations](#practical-considerations)
- [Summary](#summary)
- [Source Code](#source-code)
- [Next Step](#next-step)

## Learning Objectives

- Measure a rotated rectangular object using `cv2.minAreaRect()`.
- Separate the measured dimensions into a long side and a short side.
- Calculate signed errors against reference dimensions.
- Apply an inclusive tolerance to each side.
- Combine both checks into a final PASS or FAIL result.

## Inspection Criteria

The reference object is **150 × 80 px**, and each side has a tolerance of **±8 px**.

| Dimension | Reference | Tolerance | Accepted range |
|---|---:|---:|---:|
| Long side | 150 px | ±8 px | 142–158 px |
| Short side | 80 px | ±8 px | 72–88 px |

Both dimensions must pass. A correct short side cannot compensate for an oversized long side.

The test images contain white rectangles on a black background. All rectangles have the same center and a rotation of 20 degrees, so this example focuses on size.

## Measuring a Rotated Rectangle

`cv2.boundingRect()` returns an axis-aligned rectangle. Its width and height can change when the object rotates, even if the object's own dimensions stay the same.

`cv2.minAreaRect()` finds a minimum-area rectangle that can rotate with the contour:

```python
(_, _), (width, height), _ = cv2.minAreaRect(contour)
```

Its return structure is:

```text
((center_x, center_y), (width, height), angle)
```

Only the dimensions are needed here. The center and angle are assigned to `_` to indicate that they are not used.

The returned width is not necessarily the object's long side. Normalize the dimensions before comparing them with the reference:

```python
long_side = max(width, height)
short_side = min(width, height)
```

## Calculating Size Errors

Calculate the error by subtracting the reference from the measured length:

```python
long_error = long_side - REFERENCE_LONG_SIDE
short_error = short_side - REFERENCE_SHORT_SIDE
```

- A positive error means the measured side is too large relative to the reference.
- A negative error means it is too small relative to the reference.
- The absolute error determines whether it is within tolerance.

```python
long_pass = abs(long_error) <= SIZE_TOLERANCE
short_pass = abs(short_error) <= SIZE_TOLERANCE
overall_pass = long_pass and short_pass
```

The tolerance is **not subtracted from the measurement**. For a measured long side of 154 px, the error is `154 - 150 = +4 px`, which passes.

The comparison uses `<=`, so errors of exactly −8 px and +8 px are accepted.

## Full Source Code

File: `examples/08_Contours/src/38_size_inspection.py`

```python
"""
File: 38_size_inspection.py
Date: 2026-09-28
Author: Alex
Description:
    Measures the long and short sides of a rotated rectangular object
    and compares them with reference dimensions and a size tolerance.
"""

import cv2
import numpy as np


REFERENCE_LONG_SIDE = 150.0
REFERENCE_SHORT_SIDE = 80.0
SIZE_TOLERANCE = 8.0

IMAGE_WIDTH = 480
IMAGE_HEIGHT = 420
OBJECT_CENTER = (240, 210)


def create_test_image(size, angle=20.0):
    """Draw one rotated rectangle on a binary image."""
    image = np.zeros((IMAGE_HEIGHT, IMAGE_WIDTH), dtype=np.uint8)
    rotated_rect = (OBJECT_CENTER, size, angle)
    box = np.int32(cv2.boxPoints(rotated_rect))
    cv2.fillConvexPoly(image, box, 255)
    return image


def inspect_size(binary_image, object_name):
    """Measure both sides and check them against the reference size."""
    contours, _ = cv2.findContours(
        binary_image,
        cv2.RETR_EXTERNAL,
        cv2.CHAIN_APPROX_SIMPLE,
    )
    if not contours:
        raise ValueError(f"No contour found for {object_name}")

    contour = max(contours, key=cv2.contourArea)
    (_, _), (width, height), _ = cv2.minAreaRect(contour)

    long_side = max(width, height)
    short_side = min(width, height)
    long_error = long_side - REFERENCE_LONG_SIDE
    short_error = short_side - REFERENCE_SHORT_SIDE

    long_pass = abs(long_error) <= SIZE_TOLERANCE
    short_pass = abs(short_error) <= SIZE_TOLERANCE

    return {
        "name": object_name,
        "long_side": long_side,
        "short_side": short_side,
        "long_error": long_error,
        "short_error": short_error,
        "long_pass": long_pass,
        "short_pass": short_pass,
        "overall_pass": long_pass and short_pass,
    }


def main():
    cases = [
        ("Normal", (150, 80)),
        ("Long side too large", (175, 80)),
        ("Short side too small", (150, 60)),
    ]

    print(
        f"Reference: {REFERENCE_LONG_SIDE:.0f} x "
        f"{REFERENCE_SHORT_SIDE:.0f} px"
    )
    print(f"Tolerance: ±{SIZE_TOLERANCE:.0f} px per side\n")

    for name, size in cases:
        image = create_test_image(size)
        result = inspect_size(image, name)
        status = "PASS" if result["overall_pass"] else "FAIL"

        print(f"{name}: {status}")
        print(
            f"  Long side:  {result['long_side']:.2f} px "
            f"(error {result['long_error']:+.2f}) — "
            f"{'PASS' if result['long_pass'] else 'FAIL'}"
        )
        print(
            f"  Short side: {result['short_side']:.2f} px "
            f"(error {result['short_error']:+.2f}) — "
            f"{'PASS' if result['short_pass'] else 'FAIL'}"
        )


if __name__ == "__main__":
    main()
```

## Running the Example

Run the command from the root of the `01_ImageProcessing` repository:

```bash
python examples/08_Contours/src/38_size_inspection.py
```

The script requires OpenCV and NumPy. It generates its own test images and prints the results; no input image files are required, and no image window is opened.

The following output was recorded during this practice:

```text
Reference: 150 x 80 px
Tolerance: ±8 px per side

Normal: PASS
  Long side:  151.47 px (error +1.47) — PASS
  Short side: 80.98 px (error +0.98) — PASS
Long side too large: FAIL
  Long side:  175.87 px (error +25.87) — FAIL
  Short side: 80.63 px (error +0.63) — PASS
Short side too small: FAIL
  Long side:  151.06 px (error +1.06) — PASS
  Short side: 60.38 px (error -19.62) — FAIL
```

## Results and Interpretation

| Test case | Measured long side | Long-side error | Measured short side | Short-side error | Final result |
|---|---:|---:|---:|---:|---|
| Normal | 151.47 px | +1.47 px | 80.98 px | +0.98 px | PASS |
| Long side too large | 175.87 px | +25.87 px | 80.63 px | +0.63 px | FAIL |
| Short side too small | 151.06 px | +1.06 px | 60.38 px | −19.62 px | FAIL |

### Normal Object

Both absolute errors are smaller than 8 px, so the object passes.

The measured dimensions are slightly different from the requested 150 × 80 px. `boxPoints()` produces corner coordinates, which are converted to integers before the rectangle is drawn. Measuring the contour of this pixel representation introduces small differences from the ideal rectangle.

### Long Side Too Large

The short side passes with an error of +0.63 px. However, the long-side error is +25.87 px, which exceeds the tolerance. The final result is FAIL.

### Short Side Too Small

The long side passes with an error of +1.06 px. The short-side error is −19.62 px, whose absolute value exceeds 8 px. The final result is FAIL.

Keeping the signed errors helps explain whether a failed dimension is oversized or undersized.

## Practical Considerations

This is a controlled example using synthetic rectangular objects. The measurements are in pixels, not millimeters.

- Keep camera distance, resolution, and imaging geometry consistent when using fixed pixel references.
- For physical dimensions, calibrate the image scale and account for perspective and lens distortion as needed.
- The largest-contour selection assumes the target is the largest foreground object. Real images may require an inspection region or additional contour filtering.
- Segmentation noise can affect the fitted rectangle. Choose tolerances based on actual imaging repeatability and product requirements.
- For an irregular object, the minimum-area rectangle measures an enclosing box rather than every local feature of the object.

## Summary

This example detects size defects by measuring a rotated rectangle, normalizing its long and short sides, and comparing both dimensions with reference values.

The normal object passed. The oversized long side and undersized short side failed for the expected reasons. A final PASS requires both dimensional checks to pass.

## Source Code

[View `38_size_inspection.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/08_Contours/src/38_size_inspection.py)

## Next Step

In Post #22, I will combine position, rotation, and size inspection into one final result using `39_position_rotation_size_inspection.py`.

The combined inspection will accept an object only when all required checks pass.

---

[← Previous: Position and Rotation Inspection]({% post_url 2026-10-06-inspecting-object-position-and-rotation-alignment-using-opencv %}) · [↑ Back to Top](#post-top) · **Next: Post #22 — Coming Soon**
