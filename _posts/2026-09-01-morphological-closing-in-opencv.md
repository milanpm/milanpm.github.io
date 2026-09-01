---
layout: post
title: "Morphological Closing in OpenCV: Filling Small Black Gaps"
date: 2026-09-01 14:25:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Closing]
description: "Learn how morphological closing fills small black gaps and holes in a binary image by applying dilation followed by erosion in OpenCV."
---

# Morphological Closing in OpenCV: Filling Small Black Gaps

In the previous lesson, we learned that morphological opening applies erosion followed by dilation to remove small white foreground noise.

**Morphological closing** reverses this sequence. It applies dilation first and erosion second.

Closing is commonly used to fill small black holes, close narrow gaps, connect nearby white regions, and repair small breaks inside foreground objects.

In this post, we will learn how to:

- prepare a binary image using thresholding
- create a `5 × 5` morphological kernel
- apply closing with `cv2.morphologyEx()`
- understand the dilation–erosion sequence
- measure the number of white pixels added
- compare morphological opening and closing
- identify practical applications of closing

---

## 1. What Is Morphological Closing?

Morphological closing is a compound operation consisting of two steps:

1. dilation
2. erosion

It can be expressed as:

$$
A \bullet B = (A \oplus B) \ominus B
$$

Here:

- $A$ represents the foreground set
- $B$ represents the structuring element
- $\oplus$ represents dilation
- $\ominus$ represents erosion
- $\bullet$ represents closing

Dilation first expands white foreground regions. This expansion can fill small black holes and narrow gaps.

Erosion then shrinks the expanded regions. Larger foreground structures return close to their original size, but sufficiently small gaps filled during dilation remain closed.

The complete sequence is:

```text
Binary image → Dilation → Erosion → Closing result
```

---

## 2. Why Does Closing Fill Black Gaps?

In this example:

- white pixels (`255`) represent the foreground
- black pixels (`0`) represent the background

Suppose a white object contains a small black hole or a narrow break.

During dilation:

- the white foreground expands
- small black holes become smaller or disappear
- narrow breaks may become connected
- nearby white regions may merge

During erosion:

- the expanded outer boundaries contract
- the main objects return close to their original size
- small gaps eliminated during dilation do not necessarily reappear

Closing therefore repairs small background defects without permanently applying the full expansion caused by dilation alone.

---

## 3. Preparing the Binary Image

The source image is loaded with `cv2.imread()`:

```python
image = cv2.imread(str(image_path))
```

The BGR image is converted to grayscale:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

A threshold value of `127` converts the grayscale image into a binary image:

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

Pixels greater than `127` become white, while all other pixels become black.

A binary image makes it easier to observe and measure how closing changes foreground and background regions.

---

## 4. Creating a `5 × 5` Kernel

The rectangular structuring element is created with NumPy:

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

The `5 × 5` kernel examines a neighborhood extending two pixels from its center in every direction.

Its size determines which holes and gaps can be closed:

- small kernels fill small defects
- larger kernels can fill wider gaps
- excessively large kernels may merge separate objects

The kernel must therefore be chosen according to the size of the defects and the foreground structures that should remain separate.

---

## 5. Applying Morphological Closing

OpenCV provides `cv2.morphologyEx()` for compound morphological operations:

```python
closing = cv2.morphologyEx(
    binary,
    cv2.MORPH_CLOSE,
    kernel
)
```

The arguments are:

- `binary`: input binary image
- `cv2.MORPH_CLOSE`: closing operation
- `kernel`: structuring element

This operation is equivalent to applying dilation and erosion separately:

```python
dilated = cv2.dilate(binary, kernel, iterations=1)
closing = cv2.erode(dilated, kernel, iterations=1)
```

Using `cv2.MORPH_CLOSE` communicates the intended operation clearly and avoids managing an intermediate image manually.

---

## 6. Complete Python Example

```python
"""
File: 04_closing.py
Author: Alex
Created: 2026-09-01
Last Updated: 2026-09-01

Description:
    Demonstrates how to apply morphological closing to a binary image
    using OpenCV.

    Morphological closing performs two operations in sequence:

        1. Dilation
        2. Erosion

    Dilation first expands white foreground regions, filling small black
    holes and gaps. Erosion then restores the main foreground objects close
    to their original size.

Processing Steps:
    1. Load the source image.
    2. Convert the image to grayscale.
    3. Create a binary image using a threshold value of 127.
    4. Create a 5 x 5 rectangular kernel.
    5. Apply morphological closing with cv2.morphologyEx().
    6. Count the foreground pixels added by closing.
    7. Count all changed pixels.
    8. Save and display the closing result.

Input:
    images/sample.png

Output:
    outputs/06_Morphology/closing_5x5.png
"""

import cv2
import numpy as np
from pathlib import Path


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
output_dir = ROOT / "outputs" / "06_Morphology"
output_path = output_dir / "closing_5x5.png"

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

# Apply dilation followed by erosion.
closing = cv2.morphologyEx(
    binary,
    cv2.MORPH_CLOSE,
    kernel
)

# Create the output directory when it does not already exist.
output_dir.mkdir(parents=True, exist_ok=True)

# Save the processed image.
if not cv2.imwrite(str(output_path), closing):
    print(f"Error: Failed to save the result: {output_path}")
    raise SystemExit

# Count foreground pixels before and after closing.
original_white_pixels = cv2.countNonZero(binary)
closing_white_pixels = cv2.countNonZero(closing)
added_white_pixels = closing_white_pixels - original_white_pixels

# Count every pixel whose value changed.
difference = cv2.absdiff(binary, closing)
changed_pixels = cv2.countNonZero(difference)

print(f"Kernel size: {kernel.shape[1]} x {kernel.shape[0]}")
print(f"Original white pixels: {original_white_pixels}")
print(f"Closing white pixels: {closing_white_pixels}")
print(f"Added white pixels: {added_white_pixels}")
print(f"Changed pixels: {changed_pixels}")
print(f"Saved result: {output_path}")

# Display the binary input and closing result.
cv2.imshow("Binary Image", binary)
cv2.imshow("Morphological Closing", closing)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 7. How the Code Works

### Loading and validating the image

```python
image = cv2.imread(str(image_path))
```

The source image is loaded from the repository's `images` directory.

```python
if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit
```

The validation prevents later OpenCV operations from receiving an invalid image.

### Creating the binary input

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

The color image is converted to a single grayscale channel.

```python
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)
```

Thresholding produces an image containing only black and white pixels.

### Applying closing

```python
closing = cv2.morphologyEx(
    binary,
    cv2.MORPH_CLOSE,
    kernel
)
```

OpenCV applies dilation followed by erosion using the same `5 × 5` kernel.

### Saving the result

```python
output_dir.mkdir(parents=True, exist_ok=True)
```

The output directory is created automatically when necessary.

```python
cv2.imwrite(str(output_path), closing)
```

The result is saved as `closing_5x5.png`.

### Measuring foreground changes

```python
original_white_pixels = cv2.countNonZero(binary)
closing_white_pixels = cv2.countNonZero(closing)
```

`cv2.countNonZero()` counts the white foreground pixels in the binary images.

```python
added_white_pixels = (
    closing_white_pixels - original_white_pixels
)
```

The difference measures the net foreground added by closing.

### Counting changed pixels

```python
difference = cv2.absdiff(binary, closing)
changed_pixels = cv2.countNonZero(difference)
```

`cv2.absdiff()` compares the input and output at every pixel. Each nonzero position represents a changed pixel.

---

## 8. Morphological Closing Result

![Binary image after morphological closing with a 5 by 5 kernel]({{ '/assets/images/posts/post-12-closing/closing_5x5.png' | relative_url }})

The output remains binary and contains only `0` and `255`.

The program produced the following measurements:

| Measurement | Value |
|---|---:|
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Kernel size | `5 × 5` |
| Original white pixels | `243,363` |
| Closing white pixels | `253,375` |
| Added white pixels | `10,012` |
| Changed pixels | `10,012` |

The white foreground increased by approximately:

$$
\frac{10,012}{243,363} \times 100
\approx 4.11\%
$$

Closing increased the white foreground by approximately `4.11%`.

The added foreground count and changed pixel count are identical:

```text
Added white pixels: 10,012
Changed pixels:     10,012
```

This means all measured changes were black-to-white transitions. Closing filled black regions without producing a net loss of white pixels in this example.

The dilation stage filled small holes and gaps, while the following erosion restored the larger object boundaries.

---

## 9. Closing Compared with Dilation

Dilation and closing can both fill black gaps, but they produce different final object sizes.

| Property | Dilation | Closing |
|---|---|---|
| Operation sequence | Dilation only | Dilation → Erosion |
| Fills small black gaps | Yes | Yes |
| Expands outer boundaries | Yes | Temporarily |
| Restores object size | No | Yes |
| Connects nearby regions | Yes | Yes, depending on gap size |
| Preserves overall object size | Less effectively | More effectively |

Dilation is appropriate when expanding the foreground is the goal.

Closing is more suitable when filling small background defects while preserving the overall size of foreground objects.

---

## 10. Opening Compared with Closing

Opening and closing combine the same two basic operations in opposite orders.

| Property | Opening | Closing |
|---|---|---|
| Operation order | Erosion → Dilation | Dilation → Erosion |
| Primary target | Small white noise | Small black gaps |
| Foreground tendency | Removes small regions | Fills small holes |
| Boundary effect | Removes small protrusions | Fills small indentations |
| Typical pixel change | White → Black | Black → White |
| Main purpose | Clean foreground noise | Repair foreground gaps |

Opening is useful when unwanted features are white.

Closing is useful when unwanted defects are black.

The correct operation depends on the foreground convention and the type of noise present in the image.

---

## 11. Effect of Kernel Size

A smaller kernel produces a less aggressive closing result:

```python
small_kernel = np.ones((3, 3), dtype=np.uint8)
```

It may:

- fill very small holes
- connect narrow one-pixel breaks
- preserve more boundary detail
- keep nearby objects separate

A larger kernel produces a stronger result:

```python
large_kernel = np.ones((9, 9), dtype=np.uint8)
```

It may:

- fill larger holes
- connect wider breaks
- merge nearby foreground regions
- smooth larger inward boundary defects

An excessively large kernel can incorrectly join objects that should remain separate.

---

## 12. Practical Applications

Morphological closing is commonly used for:

- filling small black holes inside white objects
- repairing broken foreground masks
- connecting narrow gaps in lines
- joining nearby foreground regions
- smoothing small inward boundary irregularities
- improving segmentation masks
- preparing images for contour detection
- restoring broken text strokes
- cleaning masks before connected-component analysis

In document processing, closing can reconnect broken character strokes.

In machine vision, it can repair incomplete component masks before measurement or inspection.

In medical imaging, it may fill small gaps in segmented regions, although the kernel must be chosen carefully to avoid merging separate structures.

---

## 13. Important Considerations

Closing is effective when the unwanted black gaps are smaller than the important spaces that should remain open.

An excessively large kernel may:

- merge separate objects
- close meaningful holes
- connect unrelated regions
- distort object topology
- reduce measurement accuracy

The foreground convention must also be considered. This example treats white as foreground and black as background.

If the objects are black on a white background, the image may need to be inverted, or opening may produce the desired apparent effect instead.

The appropriate closing configuration depends on:

- gap size
- object spacing
- kernel shape
- kernel size
- foreground color
- required detail preservation

---

## What I Learned

In this lesson, I learned that:

- morphological closing applies dilation followed by erosion
- closing is represented as $(A \oplus B) \ominus B$
- dilation first fills small gaps by expanding the foreground
- erosion then restores the surviving object boundaries
- `cv2.MORPH_CLOSE` selects closing in `cv2.morphologyEx()`
- a `5 × 5` NumPy array can serve as the kernel
- the white pixel count increased from `243,363` to `253,375`
- closing added `10,012` white pixels
- the foreground increased by approximately `4.11%`
- opening removes small white noise, while closing fills small black gaps
- kernel size determines which gaps are filled

The most important idea from this lesson is:

> **Morphological closing fills small black gaps by expanding the white foreground first and then restoring the larger structures with erosion.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `04_closing.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/04_closing.py)

[View the generated result on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/outputs/06_Morphology/closing_5x5.png)

---

## Next Step

We now understand the two compound morphological operations:

- opening removes small white foreground noise
- closing fills small black background gaps

In the next post, we will explore the **morphological gradient**, which calculates the difference between dilation and erosion to highlight object boundaries.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
