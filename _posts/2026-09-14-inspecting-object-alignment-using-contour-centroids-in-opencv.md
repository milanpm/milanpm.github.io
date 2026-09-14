---
layout: post
title: "Inspecting Object Alignment Using Contour Centroids in OpenCV"
date: 2026-09-14 12:00:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Contours, Centroid, Machine Vision, Alignment Inspection]
description: "Measure object position offsets using contour moments in OpenCV and inspect alignment with a pixel-distance tolerance and PASS/FAIL results."
---

<!--
File: 2026-09-14-inspecting-object-alignment-using-contour-centroids-in-opencv.md
Date: 2026-09-14
Author: Alex
Description:
    Documents Day 48 contour-centroid alignment inspection,
    including position offsets, distance tolerance, visualization,
    and measured PASS/FAIL results.
-->

# Inspecting Object Alignment Using Contour Centroids in OpenCV

In Day 47 of my image-processing practice, I compared the contour centroid, bounding-box center, and minimum-enclosing-circle center.

In Day 48, I used the contour centroid to answer a practical inspection question:

> Is the object center close enough to its reference position?

This example places the same rectangular object at two different positions. It measures each centroid, calculates the position offset, and reports PASS or FAIL using a distance tolerance.

---

## 1. Inspection Conditions

The experiment uses the following settings:

| Parameter | Value |
|---|---|
| Image size per case | 400 × 400 pixels |
| Object | Filled white rectangle |
| Reference center | (200, 220) |
| Distance tolerance | 20 px |
| Normal object center | (210, 215) |
| Shifted object center | (245, 245) |
| Measurement method | Contour centroid |

The object shape stays the same. Only its position changes.

The 20 px tolerance is an example value for this experiment, rather than a production specification.

---

## 2. Preparing the Object Image

Each case starts with a blank image.

```python
image = np.zeros((400, 400, 3), dtype=np.uint8)

cx, cy = object_center

cv2.rectangle(
    image,
    (cx - 60, cy - 40),
    (cx + 60, cy + 40),
    (255, 255, 255),
    -1,
)
```

The rectangle is centered on the specified object position.

However, the inspection does not simply reuse that position. It measures the centroid from the generated image.

---

## 3. Extracting the Contour

The image is converted to grayscale and thresholded.

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY,
)

contours, _ = cv2.findContours(
    binary,
    cv2.RETR_EXTERNAL,
    cv2.CHAIN_APPROX_SIMPLE,
)

if not contours:
    raise ValueError("No contours were found.")

contour = max(contours, key=cv2.contourArea)
```

This example selects the largest external contour.

That selection works for the single-object test image. A scene containing several objects would require a suitable inspection ROI or another target-selection rule.

Inspection annotations are drawn only after extracting the contour. Otherwise, the reference marker and tolerance circle could become part of the measurement image.

---

## 4. Calculating the Contour Centroid

The centroid is calculated from contour moments.

```python
moments = cv2.moments(contour)

if moments["m00"] == 0:
    raise ValueError(
        "Cannot calculate centroid: contour area is zero."
    )

centroid = (
    moments["m10"] / moments["m00"],
    moments["m01"] / moments["m00"],
)
```

The equations are:

```text
cx = m10 / m00
cy = m01 / m00
```

For contour moments, `m00` represents the contour area.

The zero-area check prevents division by zero.

The measured centroid retains its floating-point coordinates. Rounding is performed only when creating coordinates for drawing.

```python
centroid_draw = tuple(map(round, centroid))
```

---

## 5. Measuring Position Offsets

The measured center is compared with the reference center.

```python
REFERENCE_CENTER = (200, 220)
TOLERANCE_PX = 20.0

dx = centroid[0] - REFERENCE_CENTER[0]
dy = centroid[1] - REFERENCE_CENTER[1]
```

Image coordinates use the following directions:

| Offset | Meaning |
|---|---|
| Positive dx | Right of the reference |
| Negative dx | Left of the reference |
| Positive dy | Below the reference |
| Negative dy | Above the reference |

For the normal-position case:

```text
dx = 210 - 200 = +10 px
dy = 215 - 220 = -5 px
```

The object center is 10 px to the right and 5 px above the reference.

---

## 6. Calculating the Center Distance

The Euclidean distance combines the horizontal and vertical offsets.

```python
distance = math.hypot(dx, dy)
```

This is equivalent to:

```text
distance = sqrt(dx² + dy²)
```

For the normal-position case:

```text
sqrt(10² + (-5)²) = 11.18 px
```

For the shifted-position case:

```text
sqrt(45² + 25²) = 51.48 px
```

The distance gives the magnitude of the offset. The signed dx and dy values retain its direction.

---

## 7. Making the PASS / FAIL Decision

The measured distance is compared with the tolerance.

```python
passed = distance <= TOLERANCE_PX
status = "PASS" if passed else "FAIL"
```

The decision rule is:

```text
distance <= 20 px → PASS
distance > 20 px  → FAIL
```

A distance of exactly 20 px is included in PASS.

This rule creates a circular acceptance region around the reference center.

Checking dx and dy against separate limits would create a different acceptance region.

---

## 8. Running the Example

Run the following command from the image-processing repository root:

```bash
python examples/08_Contours/src/35_center_alignment_inspection.py
```

The program prints the measurements and displays both cases side by side.

Press any key while the result window is active to close it.

---

## 9. Measured Results

| Case | Measured centroid | dx | dy | Distance | Tolerance | Result |
|---|---|---:|---:|---:|---:|---|
| Normal position | (210.00, 215.00) | +10.00 px | -5.00 px | 11.18 px | 20.00 px | PASS |
| Shifted position | (245.00, 245.00) | +45.00 px | +25.00 px | 51.48 px | 20.00 px | FAIL |

The actual console output was:

```text
[Normal position]
Reference center: (200, 220)
Measured centroid: (210.00, 215.00)
Offset: dx=+10.00 px, dy=-5.00 px
Distance: 11.18 px
Tolerance: 20.00 px
Result: PASS

[Shifted position]
Reference center: (200, 220)
Measured centroid: (245.00, 245.00)
Offset: dx=+45.00 px, dy=+25.00 px
Distance: 51.48 px
Tolerance: 20.00 px
Result: FAIL
```

Both cases produced the expected measurements and decisions.

---

## 10. Visualizing the Inspection

![Contour-centroid alignment inspection showing a normal position with PASS and a shifted position with FAIL]({{ '/assets/images/posts/post-18-center-alignment/center_alignment_inspection.png' | relative_url }})

The display uses these annotations:

| Annotation | Meaning |
|---|---|
| Blue cross | Reference center |
| Orange dot | Measured centroid |
| Yellow circle | Allowed centroid positions |
| Magenta line | Connection between reference and measured centers |
| Green contour | PASS |
| Red contour | FAIL |

The normal centroid is inside the yellow circle. The shifted centroid is outside it.

**The entire object does not need to fit inside the yellow circle.** Only its centroid must satisfy the distance condition.

Each panel has its own original image coordinates. The printed coordinates for the right panel do not include the horizontal offset introduced when joining the two images.

---

## 11. Practical Machine Vision Applications

### Object position inspection

A camera can measure how far a part is displaced from an expected position.

The measured offsets provide more information than a PASS / FAIL label alone.

### Assembly alignment

A center-distance condition can provide a basic position check before an assembly or inspection step.

The acceptance rule should reflect the actual process requirements.

### Position correction

In image coordinates, the translation required to move the measured center to the reference is:

```text
correction_x = -dx
correction_y = -dy
```

These values are pixel translations.

Converting them into physical motion requires camera calibration and a transformation into the equipment coordinate system. This example does not generate robot or PLC commands.

---

## 12. What This Inspection Does and Does Not Measure

A contour centroid depends on the object's area distribution.

A damaged or incomplete object can have a shifted centroid even when its outer placement has not changed.

Therefore, a centroid offset can reflect position changes, shape changes, or both.

PASS in this example means that the center-distance condition is satisfied. It does not establish that the object's size, orientation, or shape is acceptable.

A practical inspection may combine center position with area, dimensions, orientation, or shape measurements.

---

## 13. What I Learned

Through this example, I learned that:

- contour moments provide an area-based object center
- subtracting the reference center gives signed position offsets
- positive y points downward in image coordinates
- `math.hypot()` calculates the Euclidean center distance
- a distance tolerance creates a circular acceptance region
- the boundary is accepted when the condition uses `<=`
- measurement coordinates should retain floating-point precision
- drawing annotations should follow contour extraction
- a center-position check is only one part of an inspection

The main lesson is:

> An object center becomes useful for alignment inspection when it is compared with a defined reference position and a measurable tolerance.

---

## Source Code

The complete Python example is available on GitHub:

[View 35_center_alignment_inspection.py](https://github.com/milanpm/01_ImageProcessing/blob/980bc87/examples/08_Contours/src/35_center_alignment_inspection.py)

The learning repository is available here:

[01_ImageProcessing](https://github.com/milanpm/01_ImageProcessing)

Day 48 was recorded in commit `980bc87`:

```text
Add Day 48 center alignment inspection example
```

The example and README were pushed to GitHub, and the final working tree was clean.

---

## Related Post

The latest published image-processing experiment compares morphological kernel sizes:

[Comparing Morphological Kernel Sizes in OpenCV]({{ '/image-processing/opencv/2026/09/07/comparing-morphological-kernel-sizes-in-opencv.html' | relative_url }})

It explores a different stage of image processing: how kernel size changes foreground cleanup.

---

## Next Step

A useful extension is to compare the circular distance condition with separate horizontal and vertical tolerance limits.

This would show how the acceptance region changes when a process permits different amounts of displacement along each axis.
