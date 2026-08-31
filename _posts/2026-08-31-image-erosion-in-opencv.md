---
layout: post
title: "Image Erosion in OpenCV: Shrinking White Regions with Morphology"
date: 2026-08-31 23:19:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Erosion]
description: "Learn how image erosion works in OpenCV using a 3 by 3 rectangular structuring element to shrink white foreground regions in a binary image."
---

# Image Erosion in OpenCV: Shrinking White Regions with Morphology

Binary thresholding separates an image into black and white regions.

Once we have a binary image, morphological operations can modify the shapes of those regions. One of the fundamental morphological operations is **erosion**.

Erosion removes pixels from the boundaries of white foreground objects. It can reduce small white noise, separate objects connected by thin bridges, and prepare shapes for later analysis.

In this post, we will learn how to:

- create a binary image with `cv2.threshold()`
- build a `3 × 3` rectangular structuring element
- apply erosion with `cv2.erode()`
- understand how erosion changes white foreground regions
- measure how many pixels are removed
- identify practical uses of erosion

---

## 1. What Is Morphological Erosion?

Erosion is a shape-processing operation normally applied to a binary image.

In this lesson:

- white pixels (`255`) represent the foreground
- black pixels (`0`) represent the background

For a white pixel to remain white after erosion, all pixels covered by the structuring element must satisfy the foreground condition. If part of the kernel reaches the background, the center pixel becomes black.

The result is that:

- white objects become smaller
- their boundaries move inward
- small white details may disappear
- narrow white connections may break
- black regions become larger

Erosion can be expressed conceptually as:

$$
A \ominus B = \{z \mid B_z \subseteq A\}
$$

Here, $A$ is the foreground set and $B$ is the structuring element.

---

## 2. The Structuring Element

A structuring element, often called a kernel, defines the neighborhood used by a morphological operation.

This example creates a `3 × 3` rectangular kernel:

```python
kernel = cv2.getStructuringElement(
    cv2.MORPH_RECT,
    (3, 3)
)
```

The generated kernel is:

```text
[[1 1 1]
 [1 1 1]
 [1 1 1]]
```

For each location, OpenCV examines the center pixel and its eight neighbors.

A larger kernel produces stronger erosion because it requires a larger neighborhood to remain entirely inside the white foreground. The kernel's size and shape should therefore be selected according to the objects in the image.

---

## 3. Preparing the Binary Image

The source image is the same `800 × 600` image used in the previous thresholding lesson.

![Original image before thresholding and erosion]({{ '/assets/images/posts/post-08-binary-thresholding/sample.png' | relative_url }})

The color image is first converted to grayscale:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

A fixed threshold of `127` then creates the binary input:

```python
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)
```

![Binary input before erosion]({{ '/assets/images/posts/post-08-binary-thresholding/binary_threshold_127.png' | relative_url }})

The binary image contains `243,363` white pixels before erosion.

---

## 4. Complete Python Example

```python
import cv2
from pathlib import Path

ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"

image = cv2.imread(str(image_path))

if image is None:
    print("Error: Image file not found.")
    raise SystemExit

gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)

kernel = cv2.getStructuringElement(
    cv2.MORPH_RECT,
    (3, 3)
)

eroded = cv2.erode(
    binary,
    kernel,
    iterations=1
)

cv2.imshow("Original", image)
cv2.imshow("Binary", binary)
cv2.imshow("Erosion", eroded)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 5. How the Code Works

### 5.1 Load the source image

```python
image = cv2.imread(str(image_path))
```

The source image is loaded in BGR color format. The error check prevents later operations from running when the image cannot be found.

### 5.2 Convert the image to binary

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)
```

Morphological erosion is applied to the binary result rather than directly to the color image.

### 5.3 Create the kernel

```python
kernel = cv2.getStructuringElement(
    cv2.MORPH_RECT,
    (3, 3)
)
```

`cv2.MORPH_RECT` creates a rectangular kernel containing nine active elements.

### 5.4 Apply erosion

```python
eroded = cv2.erode(
    binary,
    kernel,
    iterations=1
)
```

The parameters are:

- `binary`: input binary image
- `kernel`: structuring element
- `iterations=1`: apply erosion once

Increasing `iterations` repeats the operation and shrinks the white regions further.

---

## 6. Erosion Result

![Binary image after erosion with a 3 by 3 rectangular kernel]({{ '/assets/images/posts/post-09-erosion/erosion_3x3.png' | relative_url }})

The output remains a binary image containing only `0` and `255`.

Measured results:

| Measurement | Value |
|---|---:|
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Binary white pixels | `243,363` |
| Eroded white pixels | `219,927` |
| Removed white pixels | `23,436` |
| Changed pixels | `23,436` |

The number of white pixels decreased by approximately `9.63%` relative to the original white foreground:

$$
\frac{23,436}{243,363} \times 100 \approx 9.63\%
$$

All changed pixels went from white to black. This confirms the central behavior of erosion: it removes foreground pixels from object boundaries.

---

## 7. How Kernel Size and Iterations Affect Erosion

The strength of erosion depends mainly on the kernel and the number of iterations.

### Kernel size

- `3 × 3`: mild erosion suitable for small boundary changes
- `5 × 5`: stronger erosion that removes thicker boundary regions
- larger kernels: may remove small objects entirely

### Iterations

```python
eroded_twice = cv2.erode(binary, kernel, iterations=2)
```

Applying erosion twice repeats the shrinking process. This can be useful for removing larger noise, but excessive erosion can destroy meaningful foreground details.

---

## 8. Practical Applications

Erosion is commonly used for:

- removing isolated white noise
- separating objects connected by thin white lines
- reducing the thickness of foreground regions
- enlarging holes and black gaps inside objects
- preparing masks for contour detection
- refining segmented objects in machine vision
- preprocessing characters and documents

In industrial inspection, erosion can help separate touching components or remove small bright artifacts before objects are measured.

---

## 9. Erosion and Dilation

Erosion and dilation are complementary operations:

| Operation | White foreground | Black background |
|---|---|---|
| Erosion | Shrinks | Expands |
| Dilation | Expands | Shrinks |

Understanding both operations is essential because they form the basis of more advanced morphological techniques such as opening and closing.

---

## What I Learned

In this lesson, I learned that:

- erosion removes pixels from white object boundaries
- a structuring element defines the neighborhood used by erosion
- `cv2.MORPH_RECT` created a `3 × 3` kernel of ones
- `cv2.erode()` applied the operation once to the binary image
- the white foreground decreased from `243,363` to `219,927` pixels
- erosion removed `23,436` white pixels, or approximately `9.63%`
- kernel size and iterations control the strength of erosion

The most important idea from this lesson is:

> **Erosion shrinks white foreground regions by turning boundary pixels black according to the structuring element.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `01_erosion.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/01_erosion.py)

---

## Next Step

Erosion shrinks white regions. The natural next step is to study the opposite operation.

In the next post, we will explore:

- image dilation
- expanding white foreground regions
- using `cv2.dilate()`
- comparing dilation with erosion
- measuring how many pixels dilation adds

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
