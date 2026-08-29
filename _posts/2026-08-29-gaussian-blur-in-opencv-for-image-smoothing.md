---
layout: post
title: "Gaussian Blur in OpenCV: Weighted Image Smoothing"
date: 2026-08-29 18:40:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Filtering, Gaussian Blur]
---

# Gaussian Blur in OpenCV: Weighted Image Smoothing

Average blur smooths an image by giving every neighboring pixel the same weight.

Gaussian blur uses a different approach.

It gives greater importance to pixels near the center of the kernel and less importance to pixels farther away.

This weighted smoothing often produces a more natural result while preserving image structure better than a simple average filter.

In this post, we will learn how to:

- apply Gaussian blur with `cv2.GaussianBlur()`
- understand a `5 × 5` Gaussian kernel
- examine center and corner weights
- compare Gaussian blur with average blur
- measure changes using pixel statistics
- understand when Gaussian smoothing is useful

---

## 1. What Is Gaussian Blur?

Gaussian blur is a smoothing filter based on the Gaussian distribution.

Instead of treating every neighboring pixel equally, it assigns weights according to distance from the center.

Pixels near the center receive larger weights.

Pixels farther from the center receive smaller weights.

Conceptually:

```text
Low weight   Medium weight   Low weight
Medium       High            Medium
Low          Medium          Low
```

This makes nearby pixels more influential when calculating the output pixel.

---

## 2. Applying Gaussian Blur in OpenCV

OpenCV provides Gaussian smoothing through:

```python
gaussian = cv2.GaussianBlur(image, (5, 5), 0)
```

The function arguments are:

```text
image   → source image
(5, 5)  → Gaussian kernel size
0       → sigmaX calculated automatically by OpenCV
```

The result has the same dimensions and channels as the source image.

```text
Input shape:  (600, 800, 3)
Output shape: (600, 800, 3)
```

Gaussian blur changes pixel values but does not resize the image.

---

## 3. Understanding Sigma

The Gaussian distribution is controlled by a value called sigma.

Sigma determines how widely the weights are distributed.

A small sigma concentrates more weight near the center.

A large sigma spreads the weights over a wider area.

The example uses:

```python
cv2.GaussianBlur(image, (5, 5), 0)
```

When `sigmaX` is set to `0`, OpenCV calculates an appropriate value from the kernel size.

This is convenient for basic image-smoothing experiments.

For more control, sigma can be specified directly:

```python
gaussian = cv2.GaussianBlur(image, (5, 5), 1.0)
```

---

## 4. The Measured `5 × 5` Gaussian Kernel

The kernel used in this experiment was examined with:

```python
kernel_1d = cv2.getGaussianKernel(5, 0)
kernel_2d = kernel_1d @ kernel_1d.T
```

The resulting two-dimensional kernel was:

```text
[[0.0039 0.0156 0.0234 0.0156 0.0039]
 [0.0156 0.0625 0.0938 0.0625 0.0156]
 [0.0234 0.0938 0.1406 0.0938 0.0234]
 [0.0156 0.0625 0.0938 0.0625 0.0156]
 [0.0039 0.0156 0.0234 0.0156 0.0039]]
```

The largest weight is at the center:

```text
Center weight = 0.140625
```

The smallest weights are at the corners:

```text
Corner weight = 0.00390625
```

The center is therefore weighted 36 times more strongly than a corner:

```text
0.140625 / 0.00390625 = 36
```

---

## 5. Why Must the Kernel Sum Be 1?

The measured kernel sum was:

```text
Kernel sum: 1.0
```

A normalized smoothing kernel should sum to 1.

This helps preserve the average brightness of the image.

If the sum were greater than 1, the filtered image could become brighter.

If the sum were less than 1, it could become darker.

Gaussian weights differ by location, but their total remains 1.

---

## 6. Gaussian Blur vs. Average Blur

Average blur gives every pixel in a `5 × 5` neighborhood the same weight.

```text
Average weight = 1 / 25 = 0.04
```

Gaussian blur assigns different weights.

```text
Center:  0.140625
Corner:  0.00390625
```

The key difference is:

| Filter | Weight distribution |
| ------ | ------------------- |
| Average blur | Every neighbor has equal weight |
| Gaussian blur | Center pixels have greater weight |

Because of this, Gaussian blur is usually influenced more by nearby pixels and less by distant pixels within the kernel.

---

## 7. Loading the Source Image

The example constructs an absolute image path from the Python file location.

```python
ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
```

The image is then loaded with:

```python
image = cv2.imread(str(image_path))
```

This path construction allows the program to locate the image even when the command is executed from another directory.

---

## 8. Handling an Image-Loading Error

If OpenCV cannot load the image, it returns `None`.

```python
if image is None:
    print("Error: Image file not found.")
    raise SystemExit
```

This stops the program before the filter is applied to invalid data.

Clear error handling is important because image paths and file formats are common sources of failure in computer-vision programs.

---

## 9. Displaying the Result

The source and Gaussian-filtered images are displayed separately.

```python
cv2.imshow("Original", image)
cv2.imshow("Gaussian Blur", gaussian)
```

The program waits until a key is pressed:

```python
cv2.waitKey(0)
```

It then closes all OpenCV windows:

```python
cv2.destroyAllWindows()
```

---

## 10. Complete Example

Here is the complete source code used in this lesson.

```python
import cv2
from pathlib import Path

ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"

image = cv2.imread(str(image_path))

if image is None:
    print("Error: Image file not found.")
    raise SystemExit

gaussian = cv2.GaussianBlur(image, (5, 5), 0)

cv2.imshow("Original", image)
cv2.imshow("Gaussian Blur", gaussian)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

Run the program with:

```bash
python examples/04_Filtering/src/02_gaussian_blur.py
```

Two windows appear:

```text
Original
Gaussian Blur
```

Press any key while an OpenCV window is active to finish the program.

---

## 11. Original Image

The following image is the original source.

![Original image before filtering]({{ '/assets/images/posts/post-06/sample.png' | relative_url }})

Image information:

```text
Dimensions:         800 × 600
Shape:              (600, 800, 3)
Standard deviation: 49.55
File size:          approximately 443 KB
```

---

## 12. Average Blur Result

The following result uses a `5 × 5` average kernel.

![Image filtered with average blur]({{ '/assets/images/posts/post-06/average_blur_5x5.png' | relative_url }})

Average blur measurements:

```text
Standard deviation:        48.37
Mean absolute difference:  3.45
Maximum difference:        67
Changed channel values:    approximately 84.8%
```

Every pixel in the `5 × 5` neighborhood contributes equally.

---

## 13. Gaussian Blur Result

The following result uses a `5 × 5` Gaussian kernel.

![Image filtered with Gaussian blur]({{ '/assets/images/posts/post-07/gaussian_blur_5x5.png' | relative_url }})

Gaussian blur measurements:

```text
Standard deviation:        48.84
Mean absolute difference:  2.16
Maximum difference:        41
Changed channel values:    1,121,811
Changed percentage:        approximately 77.9%
File size:                 approximately 543 KB
```

The image is smoothed, but the measured result remains closer to the original than the average-blur result.

---

## 14. Statistical Comparison

The three images produced the following measurements:

| Measurement | Original | Average Blur | Gaussian Blur |
| ----------- | -------: | -----------: | ------------: |
| Standard deviation | 49.55 | 48.37 | 48.84 |
| Mean absolute difference from original | 0 | 3.45 | 2.16 |
| Maximum difference from original | 0 | 67 | 41 |
| Changed channel values | 0 | 1,220,898 | 1,121,811 |
| Changed percentage | 0% | 84.8% | 77.9% |

Both filters reduced variation.

However, Gaussian blur produced:

- a smaller mean difference
- a smaller maximum difference
- fewer changed channel values
- a standard deviation closer to the original

For this image and kernel size, Gaussian blur preserved more of the original pixel structure.

---

## 15. Average Blur vs. Gaussian Blur Difference

The measured mean absolute difference between the two filtered images was:

```text
Average vs. Gaussian MAD: 1.37
```

The results are visually similar because both use a `5 × 5` neighborhood.

However, their pixel values are not identical.

The difference comes from their weighting strategies:

```text
Average blur  → equal weights
Gaussian blur → distance-based weights
```

Even a small change in the kernel weights affects the result across many pixels.

---

## 16. Why Does Gaussian Blur Preserve More Detail?

Average blur allows distant pixels in the kernel to influence the center as much as nearby pixels.

Gaussian blur reduces the influence of those distant pixels.

For the measured kernel:

```text
Center weight: 0.140625
Corner weight: 0.00390625
```

This means that the output pixel is influenced primarily by nearby information.

As a result, Gaussian smoothing can reduce small variations while avoiding some of the excessive softening produced by equal-weight averaging.

---

## 17. Effect of Kernel Size

Gaussian blur supports different odd-numbered kernel sizes.

```python
gaussian_3x3 = cv2.GaussianBlur(image, (3, 3), 0)
gaussian_5x5 = cv2.GaussianBlur(image, (5, 5), 0)
gaussian_9x9 = cv2.GaussianBlur(image, (9, 9), 0)
```

| Kernel | Typical effect |
| ------ | -------------- |
| `3 × 3` | Light smoothing |
| `5 × 5` | Moderate smoothing |
| `9 × 9` | Stronger smoothing |
| `15 × 15` | Heavy smoothing |

Kernel dimensions are normally positive odd numbers so that the filter has a clearly defined center pixel.

A larger kernel generally increases smoothing but may remove important details.

---

## 18. Gaussian Blur and Noise

Gaussian blur is especially useful for reducing high-frequency intensity variations.

These variations may come from:

- image-sensor noise
- compression artifacts
- fine textures
- small brightness fluctuations
- isolated pixel changes

It is often used before edge detection because noise can create unwanted edges.

A common sequence is:

```text
Input image
    ↓
Grayscale conversion
    ↓
Gaussian blur
    ↓
Edge detection
```

---

## 19. Gaussian Blur Before Canny Edge Detection

Canny edge detection commonly uses Gaussian smoothing internally or as an explicit preprocessing step.

Example:

```python
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)
blurred = cv2.GaussianBlur(gray, (5, 5), 0)
edges = cv2.Canny(blurred, 50, 150)
```

The blur reduces small noise responses before gradient-based edge detection.

However, an excessively large kernel can weaken real edges.

The best kernel depends on image resolution, noise level, and the size of important features.

---

## 20. Where Is Gaussian Blur Used?

Gaussian smoothing is widely used in:

- noise reduction
- edge-detection preprocessing
- contour detection
- object segmentation
- image pyramids
- feature detection
- industrial machine vision
- medical image preprocessing
- background estimation
- computer-vision pipelines

It is one of the most common preprocessing operations in classical computer vision.

---

## 21. Limitations of Gaussian Blur

Gaussian blur still smooths across object boundaries.

Its limitations include:

- edge softening
- loss of fine texture
- reduced visibility of small objects
- loss of precise boundary positions
- unsuitable results when strong edge preservation is required

For edge-preserving smoothing, alternatives include bilateral filtering and guided filtering.

Each filtering method has different trade-offs between noise removal, detail preservation, and computational cost.

---

## 22. Choosing Between Average and Gaussian Blur

Average blur is useful when:

- simplicity is important
- equal neighborhood averaging is desired
- basic smoothing is sufficient
- learning convolution fundamentals

Gaussian blur is useful when:

- more natural smoothing is desired
- nearby pixels should matter more
- noise reduction is needed before edge detection
- better detail preservation is helpful

For many general computer-vision preprocessing tasks, Gaussian blur is preferred over average blur.

---

## What I Learned

In this lesson, I learned that:

- Gaussian blur uses distance-based weights.
- Center pixels receive greater weight than distant pixels.
- `cv2.GaussianBlur()` applies Gaussian smoothing.
- A sigma value of 0 lets OpenCV calculate sigma automatically.
- The measured `5 × 5` kernel sums to 1.
- Its center weight is 0.140625.
- Its corner weight is 0.00390625.
- The center is weighted 36 times more strongly than a corner.
- Gaussian blur remained closer to the original than average blur.
- Its mean absolute difference was 2.16.
- Its maximum pixel-channel difference was 41.
- Approximately 77.9% of channel values changed.
- Gaussian blur is useful before edge detection and segmentation.

The most important idea from this lesson is:

> **Gaussian blur smooths an image while giving nearby pixels more influence than distant pixels.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `02_gaussian_blur.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/04_Filtering/src/02_gaussian_blur.py)

---

## Next Step

Now that we can reduce small intensity variations with Gaussian blur, the next step is to separate pixels according to brightness.

In the next lesson, we will explore:

- binary thresholding
- threshold values
- `cv2.threshold()`
- foreground and background separation
- grayscale-to-binary conversion

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
