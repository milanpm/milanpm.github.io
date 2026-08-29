---
layout: post
title: "Reducing Image Noise with Average Blur in OpenCV"
date: 2026-08-29 18:30:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Filtering, Average Blur]
---

# Reducing Image Noise with Average Blur in OpenCV

Digital images often contain noise, small intensity variations, and fine details that can interfere with later processing.

Image smoothing reduces these local variations by combining neighboring pixel values.

One of the simplest smoothing methods is average blur, also called box blur or mean filtering.

In this post, we will learn how to:

- apply average blur with `cv2.blur()`
- understand a `5 × 5` averaging kernel
- compare original and blurred images
- measure pixel differences
- understand the effect on image variation
- examine the advantages and limitations of average filtering

---

## 1. What Is Image Blurring?

Image blurring is a filtering operation that makes neighboring pixel values more similar.

A blurred image usually has:

- smoother intensity transitions
- less visible noise
- fewer small details
- softer edges
- reduced local variation

Blurring is commonly used before operations such as thresholding, edge detection, segmentation, and contour detection.

The goal is often to reduce unwanted detail so that larger and more meaningful structures are easier to analyze.

---

## 2. What Is Average Blur?

Average blur replaces each pixel with the average value of its neighboring pixels.

For example, using a `5 × 5` kernel means that each output pixel is calculated from 25 nearby pixels.

```text
Output pixel = Sum of 25 neighboring pixels / 25
```

OpenCV provides average blur through:

```python
average_blur = cv2.blur(image, (5, 5))
```

The first argument is the source image.

The second argument specifies the kernel width and height.

```text
Kernel width  = 5
Kernel height = 5
```

---

## 3. Understanding the `5 × 5` Kernel

A normalized `5 × 5` average kernel can be represented as:

```text
1/25 ×
[1 1 1 1 1
 1 1 1 1 1
 1 1 1 1 1
 1 1 1 1 1
 1 1 1 1 1]
```

All 25 positions have the same weight.

Each weight is:

```text
1 / 25 = 0.04
```

The sum of all weights is:

```text
25 × 0.04 = 1
```

Because the weights sum to 1, the filter generally preserves the overall brightness while smoothing local variations.

---

## 4. How the Kernel Moves Across the Image

The kernel is placed over each pixel and its neighborhood.

For every location:

1. Select the surrounding `5 × 5` pixels.
2. Add their values.
3. Divide the sum by 25.
4. Assign the average to the output pixel.
5. Move the kernel to the next location.

This sliding operation is applied across the entire image.

For a color image, OpenCV performs the calculation separately for the blue, green, and red channels.

---

## 5. Loading the Source Image

The example calculates the project root from the Python file location.

```python
ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
```

Unlike a simple relative path, this method does not depend on the directory from which the program is executed.

The image is loaded using:

```python
image = cv2.imread(str(image_path))
```

`Path` objects are converted to strings because OpenCV expects a filename string.

---

## 6. Checking for a Loading Error

If the image cannot be loaded, `cv2.imread()` returns `None`.

```python
if image is None:
    print("Error: Image file not found.")
    raise SystemExit
```

`raise SystemExit` terminates the program before OpenCV attempts to blur invalid image data.

This makes the example safer and produces a clear error message.

---

## 7. Applying Average Blur

The average filter is applied with:

```python
average_blur = cv2.blur(image, (5, 5))
```

The output keeps the same dimensions and channel count as the original.

```text
Original shape: (600, 800, 3)
Blurred shape:  (600, 800, 3)
```

The filter changes pixel values but does not resize the image or remove its color channels.

---

## 8. Displaying the Images

The example displays the original and filtered images in separate windows.

```python
cv2.imshow("Original", image)
cv2.imshow("Average Blur", average_blur)
```

The program waits for keyboard input:

```python
cv2.waitKey(0)
```

After a key is pressed, the windows are closed:

```python
cv2.destroyAllWindows()
```

Displaying both images makes it easier to compare fine details and edge softness.

---

## 9. Complete Example

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

average_blur = cv2.blur(image, (5, 5))

cv2.imshow("Original", image)
cv2.imshow("Average Blur", average_blur)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

Run the example with:

```bash
python examples/04_Filtering/src/01_blur.py
```

Two windows appear:

```text
Original
Average Blur
```

Press any key while an image window is active to finish the program.

---

## 10. Original Image

The following image is the unfiltered source image.

![Original image before average blur]({{ '/assets/images/posts/post-06/sample.png' | relative_url }})

Image information:

```text
Dimensions:         800 × 600
Shape:              (600, 800, 3)
Channels:           3
Standard deviation: 49.55
File size:          approximately 443 KB
```

The original preserves the full amount of fine texture and local pixel variation.

---

## 11. Average Blur Result

The following result uses a `5 × 5` average kernel.

![Image after 5 by 5 average blur]({{ '/assets/images/posts/post-06/average_blur_5x5.png' | relative_url }})

Result information:

```text
Dimensions:         800 × 600
Shape:              (600, 800, 3)
Kernel:             5 × 5
Standard deviation: 48.37
File size:          approximately 484 KB
```

The main structures remain visible, but fine details and sharp transitions become smoother.

---

## 12. Measuring the Difference

The original and blurred images were compared using:

```python
difference = cv2.absdiff(original, blurred)
```

The measured results were:

```text
Mean absolute difference: 3.45
Maximum difference:        67
Changed channel values:    1,220,898
```

The image contains:

```text
600 × 800 × 3 = 1,440,000 channel values
```

Therefore, approximately:

```text
1,220,898 / 1,440,000 ≈ 84.8%
```

of the individual BGR channel values changed.

Although many values changed, the average difference was only 3.45 on a scale from 0 to 255.

This means the filter made small adjustments across much of the image while preserving its overall appearance.

---

## 13. Understanding Standard Deviation

The measured standard deviations were:

```text
Original: 49.55
Blurred:  48.37
```

Standard deviation describes how widely pixel values vary.

A lower value after blurring indicates that the image became slightly more uniform.

```text
49.55 → 48.37
```

This is consistent with the purpose of smoothing:

- local variations decrease
- neighboring pixels become more similar
- noise and texture are softened

Standard deviation alone does not fully describe sharpness, but it provides useful evidence that pixel variation was reduced.

---

## 14. Why Did the PNG File Become Larger?

The observed file sizes were:

```text
Original image: 443 KB
Blurred image:  484 KB
```

It might seem reasonable to expect a smoother image to produce a smaller file, but this is not guaranteed.

PNG file size depends on:

- pixel patterns
- channel relationships
- filter selection
- compression behavior
- metadata
- encoder settings

Average blur changes many pixel values and may create patterns that the PNG encoder compresses differently.

Therefore:

> **Image smoothing does not guarantee a smaller file size.**

Image-processing quality should be evaluated using pixel data and visual results, not file size alone.

---

## 15. Effect of Kernel Size

The kernel size controls the strength of the blur.

Examples:

```python
blur_3x3 = cv2.blur(image, (3, 3))
blur_5x5 = cv2.blur(image, (5, 5))
blur_9x9 = cv2.blur(image, (9, 9))
```

| Kernel | Typical effect |
| ------ | -------------- |
| `3 × 3` | Light smoothing |
| `5 × 5` | Moderate smoothing |
| `9 × 9` | Stronger smoothing |
| `15 × 15` | Heavy loss of detail |

A larger kernel includes more neighboring pixels in the average.

This generally produces stronger smoothing, but it also removes more edge and texture information.

The kernel should be selected according to the noise level and the details that must be preserved.

---

## 16. Advantages of Average Blur

Average blur has several advantages:

- simple to understand
- easy to implement
- fast to calculate
- useful for basic smoothing
- effective for reducing small local variations
- available directly through `cv2.blur()`

It is a useful introduction to spatial filtering and convolution kernels.

---

## 17. Limitations of Average Blur

Average blur gives every neighboring pixel the same importance.

This can cause several limitations:

- edges become soft
- fine details disappear
- object boundaries become less precise
- strong noise may not be removed effectively
- all nearby pixels influence the result equally

For many applications, Gaussian blur produces a more natural result because pixels near the center receive greater weight.

---

## 18. Where Is Average Blur Used?

Average filtering can be useful in:

- basic noise reduction
- preprocessing before thresholding
- reducing texture before segmentation
- creating simple background estimates
- demonstrating convolution
- smoothing sensor data represented as images
- reducing small variations in industrial images

However, tasks requiring accurate edge locations should use the smallest effective kernel or consider an edge-preserving filter.

---

## 19. Blur and Edge Detection

Edge detectors respond to rapid intensity changes.

Noise can also create rapid changes and produce false edges.

A common processing sequence is:

```text
Input image
    ↓
Grayscale conversion
    ↓
Noise reduction
    ↓
Edge detection
```

Applying a moderate blur before edge detection can reduce small unwanted responses.

Too much blurring, however, can weaken real edges.

The goal is to reduce noise without destroying important object boundaries.

---

## What I Learned

In this lesson, I learned that:

- Average blur replaces a pixel with the mean of its neighbors.
- `cv2.blur()` applies an average filter.
- A `5 × 5` kernel uses 25 neighboring pixels.
- All average-kernel positions have equal weight.
- Blurring preserves image dimensions and channels.
- Average blur reduces local pixel variation.
- The standard deviation decreased from 49.55 to 48.37.
- Approximately 84.8% of channel values changed.
- The mean absolute difference was only 3.45.
- Larger kernels produce stronger smoothing.
- Smoothing can reduce noise but also soften edges.
- A blurred PNG is not guaranteed to have a smaller file size.

The most important idea from this lesson is:

> **Average blur reduces local variations by replacing each pixel with the mean of its neighborhood.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `01_blur.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/04_Filtering/src/01_blur.py)

---

## Next Step

Average blur gives every neighboring pixel the same weight.

The next step is to use a weighted filter that gives greater importance to pixels near the center.

In the next lesson, we will explore:

- Gaussian blur
- Gaussian-weighted kernels
- `cv2.GaussianBlur()`
- noise reduction
- average blur and Gaussian blur differences

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
