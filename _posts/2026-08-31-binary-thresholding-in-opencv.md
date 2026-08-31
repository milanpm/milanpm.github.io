---
layout: post
title: "Binary Thresholding in OpenCV: Converting Grayscale Images to Black and White"
date: 2026-08-31 22:55:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Thresholding, Binary Image]
description: "Learn how binary thresholding works in OpenCV and convert grayscale images into binary images using Python and cv2.threshold()."
---

# Binary Thresholding in OpenCV: Converting Grayscale Images to Black and White

Thresholding is one of the simplest and most useful image segmentation techniques. It separates pixels into two groups according to their intensity values, producing a binary image that contains only black and white pixels.

In this post, we will learn how to:

- convert a color image to grayscale
- apply a fixed threshold with `cv2.threshold()`
- separate pixels into foreground and background
- verify that the result contains only `0` and `255`
- measure the number of black and white pixels

---

## 1. What Is Binary Thresholding?

A grayscale image normally contains pixel values from `0` to `255`:

- `0` represents black.
- `255` represents white.
- Values between them represent different shades of gray.

Binary thresholding compares every pixel with a selected threshold value. Pixels above the threshold become white, while the remaining pixels become black.

The operation can be expressed as:

$$
dst(x,y)=
\begin{cases}
maxValue, & \text{if } src(x,y) > threshold \\
0, & \text{otherwise}
\end{cases}
$$

For example, if the threshold is `127` and the maximum value is `255`:

| Original pixel | Comparison | Output pixel |
|---:|:---:|---:|
| 40 | 40 ≤ 127 | 0 |
| 127 | 127 ≤ 127 | 0 |
| 180 | 180 > 127 | 255 |
| 240 | 240 > 127 | 255 |

This process reduces a grayscale image to two classes: foreground and background.

---

## 2. OpenCV `threshold()` Function

OpenCV provides the `cv2.threshold()` function:

```python
ret, dst = cv2.threshold(src, thresh, maxval, type)
```

Its parameters are:

- `src`: input grayscale image
- `thresh`: threshold value
- `maxval`: value assigned to pixels that satisfy the condition
- `type`: thresholding mode

The function returns:

- `ret`: the threshold value that was used
- `dst`: the resulting thresholded image

For standard binary thresholding, the mode is `cv2.THRESH_BINARY`.

---

## 3. Loading and Converting the Source Image

The example begins with the same `800 × 600` color source image used in earlier lessons.

![Original image before thresholding]({{ '/assets/images/posts/post-09/sample.png' | relative_url }})

OpenCV loads it as a BGR image and converts it to grayscale:

```python
image = cv2.imread(str(image_path))
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

![Grayscale image used for binary thresholding]({{ '/assets/images/posts/post-09/grayscale.png' | relative_url }})

The grayscale image has the following measured properties:

- shape: `600 × 800`
- minimum intensity: `20`
- maximum intensity: `248`
- mean intensity: `123.04`

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

threshold_value = 127

_, binary = cv2.threshold(
    gray,
    threshold_value,
    255,
    cv2.THRESH_BINARY
)

cv2.imshow("Original", image)
cv2.imshow("Grayscale", gray)
cv2.imshow("Binary Threshold", binary)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 5. How the Code Works

### 5.1 Load the source image

```python
image = cv2.imread(str(image_path))
```

The source image is loaded in OpenCV's default BGR color format.

### 5.2 Convert BGR to grayscale

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
```

Thresholding operates on intensity values, so the three-channel color image is converted to a single-channel grayscale image.

### 5.3 Select the threshold

```python
threshold_value = 127
```

The value `127` divides the grayscale range approximately in half. The best threshold, however, depends on the brightness, contrast, and contents of the image.

### 5.4 Apply binary thresholding

```python
_, binary = cv2.threshold(
    gray,
    threshold_value,
    255,
    cv2.THRESH_BINARY
)
```

Every pixel greater than `127` becomes `255`. Every other pixel becomes `0`.

### 5.5 Display the result

```python
cv2.imshow("Original", image)
cv2.imshow("Grayscale", gray)
cv2.imshow("Binary Threshold", binary)
```

The three windows make it easy to compare the source, grayscale, and binary images.

---

## 6. Understanding the Result

![Binary image produced with a threshold of 127]({{ '/assets/images/posts/post-09/binary_threshold_127.png' | relative_url }})

The output image contains only two intensity values:

```text
Binary values: [0, 255]
Black pixels: 236637
White pixels: 243363
```

The `800 × 600` image contains `480,000` pixels in total:

- black pixels: `236,637` (`49.30%`)
- white pixels: `243,363` (`50.70%`)

Bright regions in the original image usually appear white, while dark regions appear black. If the foreground becomes black instead, `cv2.THRESH_BINARY_INV` can reverse the output.

```python
_, binary_inverse = cv2.threshold(
    image,
    127,
    255,
    cv2.THRESH_BINARY_INV,
)
```

---

## 7. Why Threshold Selection Matters

A fixed threshold works well when foreground and background intensities are clearly separated. A poor value can cause important objects to disappear or background noise to become foreground.

- A threshold that is too low may turn too many pixels white.
- A threshold that is too high may remove useful foreground details.
- Uneven lighting may make one fixed threshold unsuitable for the entire image.

Testing several values helps reveal how the threshold affects segmentation:

```python
for value in (64, 127, 192):
    _, result = cv2.threshold(image, value, 255, cv2.THRESH_BINARY)
    cv2.imwrite(f"binary_threshold_{value}.png", result)
```

Later tutorials will address images with uneven illumination using adaptive thresholding and automatic threshold selection with Otsu's method.

---

## 8. Practical Applications

Binary thresholding is commonly used for:

- separating objects from a uniform background
- preparing images for contour detection
- document and text segmentation
- industrial inspection
- mask generation
- particle and cell analysis
- preprocessing before connected-component analysis

In a machine-vision pipeline, thresholding often converts the original image into a simple mask before measurements and defect detection are performed.

---

## What I Learned

- Binary thresholding converts grayscale pixels into two classes.
- The source image must first be converted from BGR to grayscale.
- `cv2.threshold()` applies the same threshold to every pixel.
- Pixels above `127` became `255`, while the remaining pixels became `0`.
- The output contained `236,637` black pixels and `243,363` white pixels.
- Fixed thresholding works best when illumination and contrast are consistent.

The most important idea from this lesson is:

> **Binary thresholding separates an image into foreground and background by comparing every grayscale pixel with one fixed value.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `01_binary_threshold.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/05_Threshold/src/01_binary_threshold.py)

---

## Next Step

In the next post, we will learn how **adaptive thresholding** calculates local thresholds for images with uneven illumination.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
