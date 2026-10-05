---
title: "Inspecting Object Position and Rotation Alignment Using OpenCV"
date: 2026-10-06
categories: [Image Processing, OpenCV]
tags: [OpenCV, Python, Contours, Object Inspection, Position Alignment, Rotation Alignment]
description: "Learn how to inspect both object position and rotation alignment using contour-based measurements in OpenCV."
---

In the previous post, I inspected an object's **rotation alignment** by comparing its measured angle with a reference angle.

In this post, I combine **position inspection** and **rotation inspection** to determine whether an object is correctly positioned and oriented.

## Inspection Criteria

The inspection uses the following reference values:

```python
REFERENCE_CENTER = (200, 220)
POSITION_TOLERANCE = 20

REFERENCE_ANGLE = 10.0
ANGLE_TOLERANCE = 8.0
```

An object passes the position inspection when its center is within **20 pixels** of the reference center.

It passes the rotation inspection when its angle differs from the reference angle by no more than **8 degrees**.

## Position Inspection

The distance between the measured center and the reference center is calculated as:

```python
position_error = math.sqrt(
    (cx - ref_x) ** 2 +
    (cy - ref_y) ** 2
)

position_pass = position_error <= POSITION_TOLERANCE
```

This measures the actual two-dimensional displacement of the object.

## Rotation Inspection

The rotation error is calculated from the difference between the measured and reference angles:

```python
angle_error = abs(measured_angle - REFERENCE_ANGLE)

angle_pass = angle_error <= ANGLE_TOLERANCE
```

## Combined Inspection

Both conditions must pass for the object to be accepted.

```python
overall_pass = position_pass and angle_pass
```

Therefore:

| Position | Rotation | Overall |
|---|---|---|
| PASS | PASS | PASS |
| PASS | FAIL | FAIL |
| FAIL | PASS | FAIL |
| FAIL | FAIL | FAIL |

## Experimental Results

### Normal Object

```text
Measured Center : (209.5, 214.5)
Position Error  : 10.98 px
Position        : PASS

Measured Angle  : 14.04°
Rotation        : PASS

Overall         : PASS
```

The object's position and rotation are both within their allowed tolerances.

### Misaligned Object

```text
Measured Center : (244.5, 244.5)
Position Error  : 50.80 px
Position        : FAIL

Measured Angle  : 34.99°
Rotation        : FAIL

Overall         : FAIL
```

The object exceeds both the position and rotation tolerances.

## What I Learned

This experiment extends a single-condition inspection into a simple multi-condition inspection system.

The important idea is:

```python
overall_pass = position_pass and angle_pass
```

A part should be accepted only when **all required inspection conditions are satisfied**.

This structure can also be extended with additional inspection criteria such as object size:

```python
overall_pass = (
    position_pass
    and angle_pass
    and size_pass
)
```

## Source Code

The source code for this experiment is available in the `01_ImageProcessing` GitHub repository.

## Next Step

In the next post, I will inspect **object size** and compare the measured width and height with reference dimensions and tolerances.
