---
layout: post
title: "Morphological Gradient in OpenCV: Highlighting Object Boundaries"
date: 2026-09-01 14:35:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Morphological Gradient, Boundary Detection]
description: "Learn how the morphological gradient highlights object boundaries by calculating the difference between dilation and erosion in OpenCV."
---

# Morphological Gradient in OpenCV: Highlighting Object Boundaries

In the previous lessons, we studied erosion, dilation, opening, and closing.

These operations can also be combined to highlight the boundaries of foreground objects.

The **morphological gradient** calculates the difference between a dilated image and an eroded image:

```text
Morphological Gradient = Dilation − Erosion
```

Dilation expands the white foreground, while erosion shrinks it. The difference between these two results produces a band around each object boundary.

In this post, we will learn how to:

- prepare a binary image using thresholding
- create a `5 × 5` morphological kernel
- calculate dilation and erosion
- apply `cv2.MORPH_GRADIENT`
- verify the mathematical definition
- measure the highlighted boundary pixels
- understand practical applications and limitations

---

## 1. What Is the Morphological Gradient?

The morphological gradient is the difference between the dilation and erosion of an image.

It can be expressed as:

$$
G = (A \oplus B) - (A \ominus B)
$$

Here:

- $A$ represents the foreground set
- $B$ represents the structuring element
- $\oplus$ represents dilation
- $\ominus$ represents erosion
- $G$ represents the morphological gradient

The operation consists of three conceptual steps:

1. dilate the foreground
2. erode the foreground
3. subtract the eroded result from the dilated result

The complete process is:

```text
                 ┌→ Dilation ─┐
Binary image ────┤             ├→ Dilation − Erosion → Gradient
                 └→ Erosion ──┘
```

The dilated boundary lies outside the original foreground, while the eroded boundary lies inside it.

Subtracting them produces a boundary band covering both sides of the original object boundary.

---

## 2. Why Does the Gradient Highlight Boundaries?

In this example:

- white pixels (`255`) represent foreground objects
- black pixels (`0`) represent the background

Dilation expands white regions:

- outer boundaries move into the background
- objects become thicker
- narrow gaps become smaller

Erosion shrinks white regions:

- boundaries move inward
- objects become thinner
- small foreground details may disappear

Pixels far inside a large foreground object remain white in both results. Their difference is zero.

Pixels far inside the background remain black in both results. Their difference is also zero.

The main differences occur near object boundaries. These pixels become white in the gradient image.

---

## 3. Preparing the Binary Image

The source image is loaded with `cv2.imread()`:

```python
image = cv2.imread(str(image_path))
```

It is converted from BGR color to grayscale:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

Thresholding creates the binary input:

```python
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)
```

The thresholding rule is:

$$
dst(x,y) =
\begin{cases}
255, & \text{if } src(x,y) > 127 \\
0, & \text{otherwise}
\end{cases}
$$

Pixels brighter than `127` become white, while the others become black.

Using a binary image makes the foreground boundaries and pixel measurements easy to interpret.

---

## 4. Creating a `5 × 5` Kernel

The example uses a rectangular NumPy kernel:

```python
kernel = np.ones((5, 5), dtype=np.uint8)
```

The kernel contains 25 active elements:

```text
[[1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]]
```

This kernel extends two pixels from its center in every direction.

The kernel size influences:

- boundary thickness
- sensitivity to small structures
- visibility of fine details
- amount of highlighted area

A larger kernel produces a wider boundary band. Therefore, the morphological gradient should not automatically be interpreted as a one-pixel edge map.

---

## 5. Calculating Dilation and Erosion

Dilation is calculated with:

```python
dilated = cv2.dilate(
    binary,
    kernel,
    iterations=1
)
```

Erosion is calculated with:

```python
eroded = cv2.erode(
    binary,
    kernel,
    iterations=1
)
```

These images are retained to verify the definition of the morphological gradient.

The expected result is calculated explicitly:

```python
expected_gradient = cv2.subtract(dilated, eroded)
```

`cv2.subtract()` performs saturated subtraction. For an 8-bit image, results stay within the valid range from `0` to `255`.

---

## 6. Applying the Morphological Gradient

OpenCV provides `cv2.MORPH_GRADIENT` through `cv2.morphologyEx()`:

```python
gradient = cv2.morphologyEx(
    binary,
    cv2.MORPH_GRADIENT,
    kernel
)
```

The arguments are:

- `binary`: input binary image
- `cv2.MORPH_GRADIENT`: requested operation
- `kernel`: structuring element

The result is then compared with the manually calculated difference:

```python
results_match = np.array_equal(
    gradient,
    expected_gradient
)
```

The program returned:

```text
Gradient equals dilation minus erosion: True
```

This confirms that the OpenCV result exactly matches:

```text
Dilation − Erosion
```

---

## 7. Complete Python Example

```python
"""
File: 05_morphological_gradient.py
Author: Alex
Created: 2026-09-01
Last Updated: 2026-09-01

Description:
    Demonstrates how to calculate the morphological gradient of a binary
    image using OpenCV.

    The morphological gradient is the difference between dilation and
    erosion:

        Morphological Gradient = Dilation - Erosion

    Dilation expands the white foreground, while erosion shrinks it.
    Subtracting the eroded image from the dilated image highlights the
    boundaries of foreground objects.

Processing Steps:
    1. Load the source image.
    2. Convert the image to grayscale.
    3. Create a binary image using a threshold value of 127.
    4. Create a 5 x 5 rectangular kernel.
    5. Apply dilation and erosion for comparison.
    6. Calculate the gradient with cv2.MORPH_GRADIENT.
    7. Verify that the result equals dilation minus erosion.
    8. Count the boundary pixels.
    9. Save and display the gradient result.

Input:
    images/sample.png

Output:
    outputs/06_Morphology/morphological_gradient_5x5.png
"""

import cv2
import numpy as np
from pathlib import Path


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
output_dir = ROOT / "outputs" / "06_Morphology"
output_path = output_dir / "morphological_gradient_5x5.png"

# Load the source image.
image = cv2.imread(str(image_path))

if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit

# Convert the color image to grayscale.
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

# Convert the grayscale image to a binary image.
# Pixels greater than 127 become white, and the others become black.
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)

# Create a 5 x 5 rectangular structuring element.
kernel = np.ones((5, 5), dtype=np.uint8)

# Calculate dilation and erosion separately for verification.
dilated = cv2.dilate(
    binary,
    kernel,
    iterations=1
)

eroded = cv2.erode(
    binary,
    kernel,
    iterations=1
)

# Calculate the morphological gradient.
gradient = cv2.morphologyEx(
    binary,
    cv2.MORPH_GRADIENT,
    kernel
)

# Verify the definition: gradient = dilation - erosion.
expected_gradient = cv2.subtract(dilated, eroded)
results_match = np.array_equal(gradient, expected_gradient)

# Create the output directory when it does not already exist.
output_dir.mkdir(parents=True, exist_ok=True)

# Save the gradient image.
if not cv2.imwrite(str(output_path), gradient):
    print(f"Error: Failed to save the result: {output_path}")
    raise SystemExit

# Measure the number of highlighted boundary pixels.
total_pixels = gradient.size
boundary_pixels = cv2.countNonZero(gradient)
boundary_percentage = boundary_pixels / total_pixels * 100

print(f"Kernel size: {kernel.shape[1]} x {kernel.shape[0]}")
print(f"Image size: {gradient.shape[1]} x {gradient.shape[0]}")
print(f"Total pixels: {total_pixels}")
print(f"Boundary pixels: {boundary_pixels}")
print(f"Boundary percentage: {boundary_percentage:.2f}%")
print(f"Gradient equals dilation minus erosion: {results_match}")
print(f"Saved result: {output_path}")

# Display the binary input and morphological gradient.
cv2.imshow("Binary Image", binary)
cv2.imshow("Morphological Gradient", gradient)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 8. How the Code Works

### Loading and validating the image

```python
image = cv2.imread(str(image_path))
```

The program loads the source image from `images/sample.png`.

```python
if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit
```

This validation prevents subsequent OpenCV functions from receiving an invalid image.

### Creating the binary image

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

The color image is converted into one grayscale channel.

```python
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)
```

Thresholding creates an image containing only black and white pixels.

### Calculating the gradient

```python
gradient = cv2.morphologyEx(
    binary,
    cv2.MORPH_GRADIENT,
    kernel
)
```

OpenCV expands and shrinks the foreground internally, then calculates their difference.

### Verifying the result

```python
expected_gradient = cv2.subtract(dilated, eroded)
results_match = np.array_equal(
    gradient,
    expected_gradient
)
```

This comparison verifies the mathematical definition at every pixel.

### Measuring boundary pixels

```python
total_pixels = gradient.size
boundary_pixels = cv2.countNonZero(gradient)
```

`gradient.size` returns the total number of pixels.

`cv2.countNonZero()` counts all white pixels highlighted in the gradient result.

```python
boundary_percentage = (
    boundary_pixels / total_pixels * 100
)
```

This calculation measures how much of the image is occupied by the highlighted boundary band.

---

## 9. Morphological Gradient Result

![Morphological gradient highlighting object boundaries with a 5 by 5 kernel]({{ '/assets/images/posts/post-13-morphological-gradient/morphological_gradient_5x5.png' | relative_url }})

The program produced the following results:

| Measurement | Value |
|---|---:|
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Kernel size | `5 × 5` |
| Boundary pixels | `83,720` |
| Boundary percentage | `17.44%` |
| Dilation minus erosion verification | `True` |

The boundary percentage was calculated as:

$$
\frac{83,720}{480,000} \times 100
\approx 17.44\%
$$

Approximately `17.44%` of all image pixels were highlighted by the morphological gradient.

These white pixels represent a boundary band rather than a mathematically thin, one-pixel contour. The `5 × 5` kernel causes the band to extend around both the inner and outer sides of the original foreground boundary.

The exact equality check returned `True`, confirming that `cv2.MORPH_GRADIENT` produced the same pixel values as the explicit dilation-minus-erosion calculation.

---

## 10. Effect of Kernel Size

The kernel controls the thickness and detail of the gradient.

### Using a `3 × 3` kernel

```python
small_kernel = np.ones((3, 3), dtype=np.uint8)
```

A smaller kernel generally produces:

- thinner boundary bands
- more detailed contours
- less highlighted area
- greater sensitivity to fine structures

### Using a larger kernel

```python
large_kernel = np.ones((9, 9), dtype=np.uint8)
```

A larger kernel generally produces:

- thicker boundary bands
- stronger emphasis on large structures
- more highlighted pixels
- reduced separation between nearby boundaries

If the kernel is too large, adjacent boundary bands may overlap and fine shape details may be lost.

---

## 11. Morphological Gradient vs. Canny Edge Detection

Both methods can highlight boundaries, but they work differently.

| Property | Morphological Gradient | Canny Edge Detection |
|---|---|---|
| Main principle | Dilation − Erosion | Intensity gradient with thresholding |
| Primary input | Often binary or grayscale | Usually grayscale |
| Main parameters | Kernel shape and size | Low and high thresholds |
| Typical output | Thick boundary band | Thin edge map |
| Sensitivity | Morphological structure | Intensity changes |
| Best use | Mask and shape boundaries | General image edges |

The morphological gradient is especially useful when a reliable segmentation mask already exists.

Canny edge detection is often better when edges must be detected directly from grayscale intensity changes.

The appropriate method depends on whether the task focuses on segmented object shapes or general image intensity transitions.

---

## 12. Practical Applications

The morphological gradient can be used for:

- extracting boundaries from binary masks
- emphasizing segmented object outlines
- preparing regions for contour detection
- inspecting component shapes
- detecting boundary defects
- comparing object silhouettes
- visualizing segmentation results
- measuring boundary regions
- generating masks around object borders

In machine vision, the gradient can highlight the outline of a manufactured component before dimensional or defect inspection.

In document processing, it can emphasize character boundaries.

In medical image processing, it can visualize the boundaries of segmented anatomical regions.

---

## 13. Important Considerations

The morphological gradient result depends on:

- foreground convention
- threshold quality
- kernel size
- kernel shape
- object thickness
- distance between objects

An excessively large kernel may:

- create overly thick boundaries
- merge nearby boundary bands
- hide small gaps
- remove fine contour details
- exaggerate the apparent boundary area

Thin foreground objects may be almost entirely represented as gradient pixels because erosion can remove much of their interior.

The gradient result should therefore be interpreted as a morphological boundary region, not automatically as a precise one-pixel edge.

---

## 14. Morphology Operations Summary

The five morphology operations studied so far serve different purposes:

| Operation | Definition | Main Effect |
|---|---|---|
| Erosion | Minimum over kernel neighborhood | Shrinks white regions |
| Dilation | Maximum over kernel neighborhood | Expands white regions |
| Opening | Erosion → Dilation | Removes small white noise |
| Closing | Dilation → Erosion | Fills small black gaps |
| Gradient | Dilation − Erosion | Highlights boundaries |

Erosion and dilation are the basic building blocks.

Opening, closing, and the morphological gradient combine these operations to solve more specialized image-processing problems.

---

## What I Learned

In this lesson, I learned that:

- the morphological gradient is dilation minus erosion
- dilation expands the foreground while erosion shrinks it
- their difference highlights object boundaries
- `cv2.MORPH_GRADIENT` applies the operation directly
- the explicit subtraction and OpenCV result matched exactly
- a `5 × 5` kernel produced `83,720` highlighted pixels
- the boundary band occupied approximately `17.44%` of the image
- kernel size controls the thickness of the boundary band
- the result is not necessarily a one-pixel edge
- morphological gradients are useful for segmented masks and shape analysis

The most important idea from this lesson is:

> **The morphological gradient highlights object boundaries by subtracting the eroded foreground from the dilated foreground.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `05_morphological_gradient.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/05_morphological_gradient.py)

[View the generated result on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/outputs/06_Morphology/morphological_gradient_5x5.png)

---

## Next Step

We have now studied erosion, dilation, opening, closing, and the morphological gradient.

In the next morphology lesson, we will explore the **top-hat transformation**, which extracts small bright regions from an image by subtracting the opening result from the original image.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
