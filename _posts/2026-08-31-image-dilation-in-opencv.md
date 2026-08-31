---
layout: post
title: "Image Dilation in OpenCV: Expanding White Regions with Morphology"
date: 2026-08-31 23:26:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Dilation]
description: "Learn how image dilation works in OpenCV using a 5 by 5 kernel to expand white foreground regions in a binary image."
---

# Image Dilation in OpenCV: Expanding White Regions with Morphology

In the previous lesson, erosion removed pixels from white object boundaries and made foreground regions smaller.

**Dilation** performs the complementary operation. It adds pixels to white foreground boundaries, making objects thicker and larger.

Dilation is useful for filling small gaps, connecting nearby regions, strengthening thin structures, and restoring foreground areas that became too small during segmentation.

In this post, we will learn how to:

- prepare a binary image for morphology
- create a `5 × 5` NumPy kernel
- apply dilation with `cv2.dilate()`
- understand why white regions expand
- measure the number of pixels added by dilation
- compare dilation with erosion

---

## 1. What Is Morphological Dilation?

Dilation is a morphological operation commonly applied to binary images.

In this lesson:

- white pixels (`255`) represent the foreground
- black pixels (`0`) represent the background

During dilation, a black pixel can become white when the structuring element overlaps a white foreground pixel. As the kernel moves across the image, white regions spread into their surrounding black background.

The result is that:

- white objects become larger
- their boundaries move outward
- small black gaps may close
- nearby white objects may connect
- black holes inside white objects become smaller

Dilation can be expressed conceptually as:

$$
A \oplus B = \{a+b \mid a \in A,\ b \in B\}
$$

Here, $A$ represents the foreground set and $B$ represents the structuring element.

---

## 2. Creating a `5 × 5` Kernel

The example creates a kernel using NumPy:

```python
kernel = np.ones((5, 5), dtype=np.uint8)
```

The resulting kernel contains 25 active elements:

```text
[[1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]
 [1 1 1 1 1]]
```

This kernel examines a wider neighborhood than the `3 × 3` kernel used in the erosion lesson. Because the operation reaches two pixels away from the center in every direction, it produces a clearly visible expansion of white regions.

The kernel uses `np.uint8`, which is the standard unsigned 8-bit data type expected for this binary morphological operation.

---

## 3. Preparing the Binary Image

The same `800 × 600` source image from the preceding lessons is used.

![Original image before thresholding and dilation]({{ '/assets/images/posts/post-08-binary-thresholding/sample.png' | relative_url }})

The image is converted from BGR to grayscale:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

A fixed threshold of `127` creates the binary input:

```python
_, binary = cv2.threshold(
    gray, 127, 255, cv2.THRESH_BINARY
)
```

![Binary input before dilation]({{ '/assets/images/posts/post-08-binary-thresholding/binary_threshold_127.png' | relative_url }})

Before dilation, the binary image contains `243,363` white pixels.

---

## 4. Complete Python Example

```python
import cv2
import numpy as np
from pathlib import Path

ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"

image = cv2.imread(str(image_path))

if image is None:
    print("Error: Image file not found.")
    raise SystemExit

gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

_, binary = cv2.threshold(
    gray, 127, 255, cv2.THRESH_BINARY
)

kernel = np.ones((5, 5), dtype=np.uint8)

dilated = cv2.dilate(
    binary,
    kernel,
    iterations=1
)

cv2.imshow("Binary Image", binary)
cv2.imshow("Dilated Image", dilated)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 5. How the Code Works

### 5.1 Load and validate the source image

```python
image = cv2.imread(str(image_path))

if image is None:
    print("Error: Image file not found.")
    raise SystemExit
```

The program stops safely if OpenCV cannot load the source image.

### 5.2 Create the binary input

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
_, binary = cv2.threshold(gray, 127, 255, cv2.THRESH_BINARY)
```

The color image becomes a single-channel binary image containing only black and white pixels.

### 5.3 Create the kernel

```python
kernel = np.ones((5, 5), dtype=np.uint8)
```

The `5 × 5` array defines the neighborhood used to expand the white foreground.

### 5.4 Apply dilation

```python
dilated = cv2.dilate(
    binary,
    kernel,
    iterations=1
)
```

The parameters are:

- `binary`: input binary image
- `kernel`: `5 × 5` structuring element
- `iterations=1`: perform dilation once

Additional iterations would expand the white regions further.

---

## 6. Dilation Result

![Binary image after dilation with a 5 by 5 kernel]({{ '/assets/images/posts/post-10-dilation/dilation_5x5.png' | relative_url }})

The dilated output remains binary and contains only `0` and `255`.

Measured results:

| Measurement | Value |
|---|---:|
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Binary white pixels | `243,363` |
| Dilated white pixels | `287,676` |
| Added white pixels | `44,313` |
| Changed pixels | `44,313` |

The white foreground increased by approximately `18.21%` relative to its original size:

$$
\frac{44,313}{243,363} \times 100 \approx 18.21\%
$$

All `44,313` changed pixels went from black to white. This demonstrates the main effect of dilation: expanding foreground boundaries into the background.

The increase is stronger than the reduction measured in the preceding erosion lesson because this example uses a larger `5 × 5` kernel instead of a `3 × 3` kernel.

---

## 7. How Kernel Size and Iterations Affect Dilation

The strength of dilation depends on the kernel size, kernel shape, and number of iterations.

### Kernel size

- `3 × 3`: produces a relatively small expansion
- `5 × 5`: produces a wider foreground boundary
- larger kernels: may connect objects that should remain separate

### Iterations

```python
dilated_twice = cv2.dilate(binary, kernel, iterations=2)
```

Repeating dilation grows the foreground again. This may help close larger gaps, but too much dilation can merge separate objects and remove important shape information.

---

## 8. Practical Applications

Dilation is commonly used for:

- filling small black gaps in white objects
- connecting broken foreground segments
- strengthening thin lines and strokes
- reducing small black holes
- improving masks after thresholding
- joining nearby regions before contour detection
- restoring foreground lost during erosion

In document processing, dilation can connect broken character strokes. In machine vision, it can strengthen segmented components before measurement or inspection.

---

## 9. Comparing Erosion and Dilation

Erosion and dilation affect white foreground regions in opposite ways:

| Property | Erosion | Dilation |
|---|---|---|
| Foreground effect | Shrinks | Expands |
| Background effect | Expands | Shrinks |
| Small white noise | Can remove | Can enlarge |
| Small black gaps | Can enlarge | Can fill |
| Thin white connections | Can break | Can connect |

The two operations are building blocks for **opening** and **closing**, which combine erosion and dilation in different orders.

---

## What I Learned

In this lesson, I learned that:

- dilation adds pixels to white foreground boundaries
- a `5 × 5` NumPy array can be used as the kernel
- `cv2.dilate()` expanded the binary foreground once
- the white pixel count increased from `243,363` to `287,676`
- dilation added `44,313` white pixels
- the foreground increased by approximately `18.21%`
- larger kernels and more iterations produce stronger dilation
- erosion and dilation have complementary effects

The most important idea from this lesson is:

> **Dilation expands white foreground regions by turning nearby background pixels white according to the kernel.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `02_dilation.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/02_dilation.py)

---

## Next Step

We now understand the two fundamental morphological operations: erosion and dilation.

In the next post, we will combine them to explore **morphological opening**, which is useful for removing small white noise while preserving larger foreground structures.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
