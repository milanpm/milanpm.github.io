---
layout: post
title: "Morphological Opening in OpenCV: Removing Small White Noise"
date: 2026-09-01 14:15:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Opening]
description: "Learn how morphological opening removes small white noise from a binary image by applying erosion followed by dilation in OpenCV."
---

# Morphological Opening in OpenCV: Removing Small White Noise

In the previous lessons, we studied the two fundamental morphological operations:

- erosion shrinks white foreground regions
- dilation expands white foreground regions

These operations can also be combined to solve more specific image-processing problems.

**Morphological opening** applies erosion first and dilation second. It is commonly used to remove small white noise while preserving larger foreground objects.

In this post, we will learn how to:

- prepare a binary image using thresholding
- create a `5 × 5` morphological kernel
- apply opening with `cv2.morphologyEx()`
- understand the erosion–dilation sequence
- measure how many foreground pixels are removed
- identify practical applications of morphological opening

---

## 1. What Is Morphological Opening?

Morphological opening is a compound operation consisting of two steps:

1. erosion
2. dilation

The operation can be expressed as:

$$
A \circ B = (A \ominus B) \oplus B
$$

Here:

- $A$ is the foreground set in the binary image
- $B$ is the structuring element
- $\ominus$ represents erosion
- $\oplus$ represents dilation
- $\circ$ represents opening

The erosion stage removes pixels from white foreground boundaries. Small white regions that cannot contain the kernel may disappear completely.

The dilation stage then expands the foreground that survived erosion. Larger objects are restored close to their original size, but small noise removed during erosion does not return.

The complete sequence is:

```text
Binary image → Erosion → Dilation → Opening result
```

---

## 2. Why Does Opening Remove White Noise?

In this example:

- white pixels (`255`) represent the foreground
- black pixels (`0`) represent the background

Suppose a binary image contains large white objects and several isolated white noise regions.

During erosion:

- large objects become temporarily smaller
- thin protrusions may disappear
- isolated white pixels and small white regions are removed

During the following dilation:

- surviving foreground objects expand again
- larger structures recover much of their original size
- removed noise cannot reappear because no foreground pixels remain there

This makes opening particularly effective for removing small bright features from a dark background.

Opening is different from applying erosion alone. Erosion removes noise, but it also leaves every foreground object smaller. Opening uses dilation to restore the surviving structures after noise removal.

---

## 3. Preparing the Binary Image

The source image is loaded with `cv2.imread()`:

```python
image = cv2.imread(str(image_path))
```

The image is then converted from BGR color to grayscale:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

Morphological operations can be applied to grayscale images, but a binary image makes the foreground changes easier to understand and measure.

The grayscale image is converted to binary using a threshold value of `127`:

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

Pixels brighter than `127` become white, while the remaining pixels become black.

---

## 4. Creating a `5 × 5` Kernel

The structuring element is created as a NumPy array:

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

This rectangular kernel examines a `5 × 5` neighborhood around each pixel.

The kernel size determines which foreground structures survive the erosion stage. Small white regions that cannot support the kernel are likely to disappear.

A larger kernel removes larger noise regions, but it can also remove thin details that belong to important objects.

---

## 5. Applying Morphological Opening

OpenCV provides `cv2.morphologyEx()` for compound morphological operations:

```python
opening = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel
)
```

The arguments are:

- `binary`: the input binary image
- `cv2.MORPH_OPEN`: the requested morphological operation
- `kernel`: the structuring element

This is equivalent to applying erosion and dilation separately:

```python
eroded = cv2.erode(binary, kernel, iterations=1)
opening = cv2.dilate(eroded, kernel, iterations=1)
```

Using `cv2.MORPH_OPEN` clearly communicates the intended operation and avoids managing the intermediate image manually.

---

## 6. Complete Python Example

```python
"""
Morphological Opening Example

This example demonstrates how to apply morphological opening to a binary
image using OpenCV.

Morphological opening performs two operations in sequence:

    1. Erosion
    2. Dilation

Erosion first removes small white regions and shrinks foreground boundaries.
Dilation then restores the main foreground objects close to their original
size. As a result, opening is useful for removing small white noise while
preserving larger objects.

Processing steps:
    1. Load the source image.
    2. Convert the image to grayscale.
    3. Create a binary image using a threshold value of 127.
    4. Create a 5 x 5 rectangular kernel.
    5. Apply morphological opening with cv2.morphologyEx().
    6. Count the foreground pixels removed by the operation.
    7. Save and display the opening result.

Output:
    outputs/06_Morphology/opening_5x5.png
"""

import cv2
import numpy as np
from pathlib import Path


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
output_dir = ROOT / "outputs" / "06_Morphology"
output_path = output_dir / "opening_5x5.png"

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

# Apply erosion followed by dilation.
opening = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel
)

# Create the output directory when it does not already exist.
output_dir.mkdir(parents=True, exist_ok=True)

# Save the processed image.
if not cv2.imwrite(str(output_path), opening):
    print(f"Error: Failed to save the result: {output_path}")
    raise SystemExit

# Count foreground pixels before and after opening.
original_white_pixels = cv2.countNonZero(binary)
opening_white_pixels = cv2.countNonZero(opening)
removed_white_pixels = original_white_pixels - opening_white_pixels

# Count every pixel whose value changed.
difference = cv2.absdiff(binary, opening)
changed_pixels = cv2.countNonZero(difference)

print(f"Kernel size: {kernel.shape[1]} x {kernel.shape[0]}")
print(f"Original white pixels: {original_white_pixels}")
print(f"Opening white pixels: {opening_white_pixels}")
print(f"Removed white pixels: {removed_white_pixels}")
print(f"Changed pixels: {changed_pixels}")
print(f"Saved result: {output_path}")

# Display the binary input and opening result.
cv2.imshow("Binary Image", binary)
cv2.imshow("Morphological Opening", opening)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 7. How the Code Works

### Loading the image

```python
image = cv2.imread(str(image_path))
```

The source image is loaded as a BGR color image.

The program verifies that loading succeeded:

```python
if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit
```

This prevents later OpenCV functions from receiving an invalid image.

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

### Applying opening

```python
opening = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel
)
```

OpenCV performs erosion followed by dilation using the same `5 × 5` kernel.

### Saving the result

```python
output_dir.mkdir(parents=True, exist_ok=True)
```

The output directory is created automatically if it does not exist.

```python
cv2.imwrite(str(output_path), opening)
```

The processed binary image is saved as `opening_5x5.png`.

### Measuring the foreground

```python
original_white_pixels = cv2.countNonZero(binary)
opening_white_pixels = cv2.countNonZero(opening)
```

Because foreground pixels have a value of `255`, `cv2.countNonZero()` measures the number of white pixels.

```python
removed_white_pixels = (
    original_white_pixels - opening_white_pixels
)
```

The difference indicates how much foreground was removed.

### Counting changed pixels

```python
difference = cv2.absdiff(binary, opening)
changed_pixels = cv2.countNonZero(difference)
```

`cv2.absdiff()` compares the input and output pixel by pixel. Every nonzero position in the difference image represents a pixel changed by opening.

---

## 8. Morphological Opening Result

![Binary image after morphological opening with a 5 by 5 kernel]({{ '/assets/images/posts/post-11-opening/opening_5x5.png' | relative_url }})

The output remains a binary image containing only `0` and `255`.

The program produced the following measurements:

| Measurement | Value |
|---|---:|
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Kernel size | `5 × 5` |
| Original white pixels | `243,363` |
| Opening white pixels | `230,991` |
| Removed white pixels | `12,372` |
| Changed pixels | `12,372` |

The percentage of foreground removed was approximately:

$$
\frac{12,372}{243,363} \times 100
\approx 5.08\%
$$

Opening removed approximately `5.08%` of the original white foreground.

The number of removed white pixels and changed pixels is the same:

```text
Removed white pixels: 12,372
Changed pixels:       12,372
```

This means that all measured changes were white-to-black transitions. The small foreground regions removed during erosion were not restored by the following dilation.

Larger foreground structures survived the erosion stage and were expanded again during dilation.

---

## 9. Opening Compared with Erosion

Erosion and opening can both remove small white noise, but they do not produce the same result.

| Property | Erosion | Opening |
|---|---|---|
| Operation sequence | Erosion only | Erosion → Dilation |
| Removes small white noise | Yes | Yes |
| Shrinks large objects | Yes | Temporarily |
| Restores surviving objects | No | Yes |
| Preserves overall object size | Less effectively | More effectively |
| Smooths small protrusions | Yes | Yes |

Erosion is useful when shrinking the foreground is itself the goal.

Opening is more appropriate when the goal is to remove small foreground noise while retaining the main objects as much as possible.

---

## 10. Effect of Kernel Size

The behavior of opening depends strongly on the kernel.

### Smaller kernel

```python
small_kernel = np.ones((3, 3), dtype=np.uint8)
```

A smaller kernel:

- removes very small noise
- preserves more thin foreground details
- produces a less aggressive result

### Larger kernel

```python
large_kernel = np.ones((9, 9), dtype=np.uint8)
```

A larger kernel:

- removes larger white regions
- smooths boundaries more strongly
- may eliminate narrow object parts
- may separate objects connected by thin bridges

The kernel should therefore be chosen according to the expected noise size and the smallest meaningful foreground structure.

---

## 11. Practical Applications

Morphological opening is commonly used for:

- removing isolated white pixels
- cleaning binary segmentation masks
- eliminating small bright particles
- separating objects connected by thin white bridges
- smoothing small outward boundary irregularities
- preparing images for contour detection
- improving connected-component analysis
- removing small artifacts before measurement

In machine vision, opening can remove small reflections or segmentation artifacts before inspecting a component.

In document processing, it can eliminate isolated white marks from binary masks.

In medical image processing, opening may help clean segmented regions, although the kernel size must be chosen carefully to avoid removing small anatomical structures.

---

## 12. Important Considerations

Opening is effective only when the noise is smaller than the meaningful foreground objects.

An excessively large kernel may:

- remove thin structures
- break narrow connections
- distort small objects
- eliminate important details
- reduce measurement accuracy

The foreground convention is also important. In this example, white represents the foreground.

If the objects are black and the background is white, the apparent effect will be reversed. The image may need to be inverted before applying the operation.

Opening should therefore be configured according to:

- foreground color
- noise size
- object size
- kernel shape
- kernel size
- required level of detail preservation

---

## What I Learned

In this lesson, I learned that:

- morphological opening applies erosion followed by dilation
- opening is represented as $(A \ominus B) \oplus B$
- thresholding creates a measurable binary input
- a `5 × 5` NumPy array can be used as the kernel
- `cv2.MORPH_OPEN` selects opening in `cv2.morphologyEx()`
- the white pixel count decreased from `243,363` to `230,991`
- opening removed `12,372` white pixels
- approximately `5.08%` of the original foreground was removed
- small white noise removed during erosion does not return during dilation
- kernel size determines which foreground structures survive

The most important idea from this lesson is:

> **Morphological opening removes small white foreground regions by applying erosion first and then restoring the surviving structures with dilation.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `03_opening.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/03_opening.py)

[View the generated result on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/outputs/06_Morphology/opening_5x5.png)

---

## Next Step

Opening removes small white foreground noise by applying erosion followed by dilation.

In the next post, we will reverse the operation order and explore **morphological closing**, which applies dilation followed by erosion to fill small black gaps and holes.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
