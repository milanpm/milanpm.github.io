---
layout: post
title: "Inspecting Object Rotation Alignment Using OpenCV"
date: 2026-09-21 16:00:00 +0900
categories: [Image Processing, OpenCV]
tags: [opencv, computer-vision, contour, image-moments, rotation, alignment, inspection]
description: "Learn how to measure contour orientation with image moments, calculate angular error, and classify rotated objects using a reference angle and tolerance."
---

In the previous post, I inspected object alignment by comparing contour centroids with a reference position.

However, position alone is not enough for many machine vision applications. An object may be located at the correct position while still being rotated beyond the acceptable orientation.

In this post, I use OpenCV image moments to measure the orientation of rectangular objects, compare each measurement with a reference angle, and classify the object as PASS or FAIL according to an angular tolerance.

## Table of Contents

1. [Learning Objectives](#learning-objectives)
2. [Why Rotation Alignment Matters](#why-rotation-alignment-matters)
3. [Inspection Criteria](#inspection-criteria)
4. [Creating Rotated Test Objects](#creating-rotated-test-objects)
5. [Measuring Orientation with Image Moments](#measuring-orientation-with-image-moments)
6. [Calculating the Smallest Angular Error](#calculating-the-smallest-angular-error)
7. [PASS/FAIL Decision](#passfail-decision)
8. [Visualizing the Inspection](#visualizing-the-inspection)
9. [Experimental Results](#experimental-results)
10. [Why the Measured Angle Differs Slightly](#why-the-measured-angle-differs-slightly)
11. [Practical Applications](#practical-applications)
12. [Limitations and Improvements](#limitations-and-improvements)
13. [Conclusion](#conclusion)
14. [Source Code](#source-code)
15. [Next Step](#next-step)

## Learning Objectives

In this post, I will:

- create rotated rectangular contours with `cv2.boxPoints()`;
- calculate contour orientation from central image moments;
- convert the orientation from radians to degrees;
- calculate the smallest signed angular error for an axis orientation;
- apply an angular tolerance;
- visualize PASS and FAIL inspection results.

## Why Rotation Alignment Matters

A centroid tells us where an object is located, but it does not tell us whether the object is facing the correct direction.

For example, two parts may have the same center position while one is rotated incorrectly. A position-only inspection would accept both parts, but a rotation inspection can distinguish the correctly oriented part from the defective one.

Rotation inspection is useful for:

- checking components on a production line;
- verifying PCB and electronic-part orientation;
- inspecting labels and packages;
- aligning parts before robotic pick-and-place operations;
- confirming object pose before measurement or assembly.

## Inspection Criteria

The rotation inspection uses the following parameters:

| Parameter | Value |
|---|---:|
| Reference angle | 10.00° |
| Angular tolerance | ±8.00° |
| Acceptable range | 2.00° to 18.00° |

The signed angular error is calculated as:

$$
e_\theta = \theta_{\text{measured}} - \theta_{\text{reference}}
$$

The inspection decision is:

$$
\text{PASS if } |e_\theta| \leq T_\theta
$$

where \(T_\theta\) is the angular tolerance.

Because an orientation axis has the same direction every 180 degrees, the implementation also normalizes the error to the range from −90 to +90 degrees.

## Creating Rotated Test Objects

I created two rectangular contours with `cv2.boxPoints()`:

```python
def create_rotated_rectangle(center, size, angle):
    """Create contour points for a rotated rectangle."""
    rectangle = (center, size, angle)
    box = cv2.boxPoints(rectangle)
    return box.astype(np.int32)
```

The normal object was created with an input angle of 14 degrees, while the rotated object was created with an input angle of 35 degrees.

```python
normal_contour = create_rotated_rectangle(
    center=(180, 220),
    size=(180, 80),
    angle=14
)

rotated_contour = create_rotated_rectangle(
    center=(530, 220),
    size=(180, 80),
    angle=35
)
```

The two objects have different center positions for visualization, but the inspection in this example evaluates only their rotation.

## Measuring Orientation with Image Moments

OpenCV's `cv2.moments()` function calculates spatial and central moments from a contour.

```python
moments = cv2.moments(contour)
```

The orientation calculation uses the second-order central moments:

- `mu20`: spread along the x-axis;
- `mu02`: spread along the y-axis;
- `mu11`: correlation between the x and y coordinates.

The principal-axis orientation is calculated as:

$$
\theta = \frac{1}{2}\operatorname{atan2}
\left(2\mu_{11},\mu_{20}-\mu_{02}\right)
$$

The implementation is:

```python
def calculate_orientation(contour):
    """Calculate contour orientation in degrees using image moments."""
    moments = cv2.moments(contour)

    if moments["m00"] == 0:
        raise ValueError("Contour area is zero.")

    mu20 = moments["mu20"]
    mu02 = moments["mu02"]
    mu11 = moments["mu11"]

    angle_radians = 0.5 * np.arctan2(
        2.0 * mu11,
        mu20 - mu02
    )

    return np.degrees(angle_radians)
```

The factor of one-half is required because the second-order moments describe an axis rather than a directed vector. The axis repeats after 180 degrees, not 360 degrees.

In OpenCV image coordinates, the y-coordinate increases downward. Therefore, a positive measured angle appears clockwise on the displayed image.

## Calculating the Smallest Angular Error

A direct subtraction is not sufficient near the orientation boundary. For example, +89 degrees and −89 degrees describe axes that differ by only 2 degrees, not 178 degrees.

The following function calculates the smallest signed error for a 180-degree axis orientation:

```python
def calculate_angle_error(measured_angle, reference_angle):
    """Calculate the smallest angle error for an axis orientation."""
    error = measured_angle - reference_angle
    return (error + 90.0) % 180.0 - 90.0
```

This expression normalizes the result to:

$$
-90^\circ \leq e_\theta < 90^\circ
$$

The sign indicates the direction of the deviation, while the absolute value determines whether it remains within tolerance.

## PASS/FAIL Decision

The measured angle is compared with the 10-degree reference angle. The object passes when the absolute angular error does not exceed 8 degrees.

```python
measured_angle = calculate_orientation(contour)
angle_error = calculate_angle_error(
    measured_angle,
    REFERENCE_ANGLE
)

passed = abs(angle_error) <= ANGLE_TOLERANCE
result = "PASS" if passed else "FAIL"
```

Using `<=` means that an object exactly on the tolerance boundary is accepted.

## Visualizing the Inspection

The program draws:

- a white contour around each object;
- a colored line along the measured principal orientation;
- a red point at the contour centroid;
- the measured angle and signed error;
- the final PASS or FAIL result.

The orientation line is green for a passing object and red for a failing object.

```python
dx = int(length * np.cos(angle_radians))
dy = int(length * np.sin(angle_radians))

start_point = (center[0] - dx, center[1] - dy)
end_point = (center[0] + dx, center[1] + dy)

cv2.line(image, start_point, end_point, color, 3)
cv2.circle(image, center, 5, (0, 0, 255), -1)
```

![Rotation alignment inspection showing PASS and FAIL results](/assets/images/posts/post-19-rotation-alignment/rotation_alignment_inspection.png)

*Rotation alignment inspection using a 10-degree reference angle and an ±8-degree tolerance. The normal object passes with a +3.66-degree error, while the rotated object fails with a +25.05-degree error.*

## Experimental Results

| Object | Input Angle | Measured Angle | Signed Error | Absolute Error | Result |
|---|---:|---:|---:|---:|---|
| Normal Object | 14.00° | 13.66° | +3.66° | 3.66° | PASS |
| Rotated Object | 35.00° | 35.05° | +25.05° | 25.05° | FAIL |

The normal object was measured at 13.66 degrees. Its absolute angular error was 3.66 degrees, which was smaller than the allowed tolerance of 8.00 degrees. It was therefore classified as PASS.

The rotated object was measured at 35.05 degrees. Its absolute angular error was 25.05 degrees, which exceeded the allowed tolerance. It was therefore classified as FAIL.

## Why the Measured Angle Differs Slightly

The measured orientations are close to, but not exactly equal to, the input angles. The normal object's input angle was 14.00 degrees, but its measured angle was 13.66 degrees. The rotated object's input angle was 35.00 degrees, but its measured angle was 35.05 degrees.

This small difference is expected because:

- `cv2.boxPoints()` initially returns floating-point coordinates;
- the coordinates are converted to integer pixel positions;
- a rotated edge becomes a discrete staircase of pixels;
- the moments are calculated from the resulting integer contour.

In a real inspection system, image resolution, blur, threshold selection, noise, lens distortion, and contour quality can introduce additional measurement variation. The tolerance should therefore be selected from repeated measurements of real acceptable and defective samples.

## Practical Applications

This inspection method can be adapted to:

- detect rotated components on a conveyor;
- verify label or barcode orientation;
- check the pose of rectangular manufactured parts;
- align an object before robotic handling;
- inspect PCB component placement;
- measure the dominant orientation of elongated regions.

## Limitations and Improvements

This example uses clean synthetic rectangular contours. A real machine vision system should also consider:

1. **Contour preprocessing**
   Apply grayscale conversion, thresholding, morphology, and noise removal before measuring orientation.

2. **Contour selection**
   Filter candidates by area, position, aspect ratio, or expected shape instead of assuming that every contour is a valid part.

3. **Nearly symmetric objects**
   Orientation becomes unstable when an object is close to circular or square because it has no strongly dominant principal axis.

4. **Directional ambiguity**
   Image moments measure an axis, so the directions \(\theta\) and \(\theta + 180^\circ\) are equivalent. An asymmetric feature is required when the front and back directions must be distinguished.

5. **Calibration and repeatability**
   Determine the production tolerance from repeated measurements under actual lighting, camera, and part-placement conditions.

6. **Combined inspection**
   Evaluate position and rotation together because an object can pass one criterion while failing the other.

## Conclusion

In this post, I measured object rotation using second-order contour moments. I calculated the principal-axis orientation, normalized the angular error to account for the 180-degree axis period, and applied a tolerance-based PASS/FAIL decision.

The normal object passed with a +3.66-degree error, while the rotated object failed with a +25.05-degree error. This experiment demonstrates how contour geometry can be converted into a simple and interpretable machine vision inspection rule.

## Source Code

[View `36_rotation_alignment_inspection.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/08_Contours/src/36_rotation_alignment_inspection.py)

## Next Step

In the next post, I will combine centroid-based position inspection with moment-based rotation inspection. An object will pass only when both its position error and angular error remain within their respective tolerances.
