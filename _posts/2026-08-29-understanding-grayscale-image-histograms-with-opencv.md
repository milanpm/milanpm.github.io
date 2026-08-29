---
layout: post
title: "Understanding Grayscale Image Histograms with OpenCV"
date: 2026-08-29 18:20:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Matplotlib, Image Processing, Histogram]
---

# Understanding Grayscale Image Histograms with OpenCV

A grayscale image contains intensity values ranging from dark to bright.

Looking at the image shows us its visual appearance, but it does not directly tell us how those intensity values are distributed.

An image histogram provides a numerical summary of that distribution.

In this post, we will learn how to:

- load a grayscale image with OpenCV
- flatten a two-dimensional image array
- create a 256-bin histogram with Matplotlib
- understand histogram axes and peaks
- calculate basic intensity statistics
- verify that every pixel is included
- interpret brightness and contrast from a histogram

---

## 1. What Is an Image Histogram?

An image histogram counts how many pixels have each intensity value.

For an 8-bit grayscale image, possible pixel values range from:

```text
0 to 255
```

Each value represents a brightness level:

```text
0   = Black
255 = White
```

A histogram displays this information using two axes.

| Axis | Meaning |
| ---- | ------- |
| X-axis | Pixel intensity from 0 to 255 |
| Y-axis | Number of pixels with that intensity |

If an image contains many dark pixels, the histogram has more values on the left.

If it contains many bright pixels, the histogram has more values on the right.

---

## 2. Loading the Image as Grayscale

The example loads the source image directly in grayscale mode.

```python
img = cv2.imread(image_path, cv2.IMREAD_GRAYSCALE)
```

The flag:

```python
cv2.IMREAD_GRAYSCALE
```

tells OpenCV to return a single-channel image.

The resulting shape is:

```text
(600, 800)
```

This means:

```text
Height = 600 pixels
Width  = 800 pixels
```

The total number of pixels is:

```text
600 × 800 = 480,000 pixels
```

---

## 3. Checking Whether the Image Was Loaded

If OpenCV cannot find or decode the image, `cv2.imread()` returns `None`.

```python
if img is None:
    print("Error: image file not found.")
    exit()
```

This check prevents the histogram code from running with invalid image data.

Common causes of failure include an incorrect path, a missing file, or running the script from the wrong directory.

---

## 4. Flattening the Image with `ravel()`

A grayscale image is a two-dimensional NumPy array.

```text
(600, 800)
```

Matplotlib needs the individual pixel values as a single sequence. The example uses:

```python
img.ravel()
```

`img.ravel()` returns the pixel values as a flattened one-dimensional array. It returns a view when possible and creates a copy only when necessary.

```text
(600, 800)
```

to:

```text
(480000,)
```

Conceptually, the pixel rows are placed one after another:

```text
Row 1 → Row 2 → Row 3 → ... → Row 600
```

The pixel values and their order remain unchanged; only their array representation becomes one-dimensional.

---

## 5. Creating the Histogram

The histogram is created with:

```python
plt.hist(img.ravel(), bins=256, range=[0, 256])
```

The arguments mean:

| Argument | Meaning |
| -------- | ------- |
| `img.ravel()` | All grayscale pixel values |
| `bins=256` | One bin for each 8-bit intensity level |
| `range=[0, 256]` | Analyze values from 0 through 255 |

Although the highest valid `uint8` value is 255, the upper histogram boundary is written as 256 so that intensity 255 is included in the final bin.

---

## 6. Adding the Title and Axis Labels

The example adds information to make the graph easier to understand.

```python
plt.title("Grayscale Histogram")
plt.xlabel("Pixel Value")
plt.ylabel("Frequency")
```

The X-axis represents pixel intensity:

```text
Dark ← 0 ... 255 → Bright
```

The Y-axis represents frequency, which is the number of pixels assigned to each intensity bin.

Finally, the graph is displayed with:

```python
plt.show()
```

The program waits while the graph window is open and finishes after the window is closed.

---

## 7. Complete Example

Here is the complete source code used in this lesson.

```python
from pathlib import Path

import cv2
import matplotlib.pyplot as plt
import numpy as np


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"

img = cv2.imread(str(image_path), cv2.IMREAD_GRAYSCALE)

if img is None:
    raise FileNotFoundError(f"Could not load image: {image_path}")

frequencies, _, _ = plt.hist(
    img.ravel(),
    bins=256,
    range=[0, 256],
)

peak_intensity = int(np.argmax(frequencies))
peak_frequency = int(frequencies[peak_intensity])

print(f"Shape:           {img.shape}")
print(f"Total pixels:    {img.size:,}")
print(f"Minimum:         {int(img.min())}")
print(f"Maximum:         {int(img.max())}")
print(f"Mean:            {img.mean():.2f}")
print(f"Median:          {np.median(img):.1f}")
print(f"Peak intensity:  {peak_intensity}")
print(f"Peak frequency:  {peak_frequency}")
print(f"Histogram total: {int(frequencies.sum()):,}")

plt.title("Grayscale Histogram")
plt.xlabel("Pixel Value")
plt.ylabel("Frequency")
plt.xlim([0, 256])
plt.show()
```

Run the example from the project root:

```bash
python examples/03_Histogram/src/01_histogram.py
```

The script resolves the image path from the project root, prints the measured image statistics, and displays the grayscale histogram in a Matplotlib window.

---

## 8. Grayscale Source Image

The following grayscale image was analyzed.

![Grayscale source image]({{ '/assets/images/posts/post-05/sample_gray.png' | relative_url }})

Image information:

```text
Dimensions: 800 × 600
Shape:      (600, 800)
Data type:  uint8
Pixels:     480,000
```

Every pixel contributes once to the histogram.

---

## 9. Histogram Result

The following graph shows the grayscale intensity distribution.

![Grayscale image histogram]({{ '/assets/images/posts/post-05/grayscale_histogram.png' | relative_url }})

The graph contains several visible peaks around intensity regions such as:

```text
50, 100, 150, and 200
```

These peaks suggest that the image contains several groups of pixels with different brightness characteristics.

For example, different peaks may correspond to dark backgrounds, midtone objects, illuminated surfaces, or highlights.

A histogram reports intensity frequencies, but it does not show where those pixels are located in the image.

---

## 10. Measured Image Statistics

The following values were measured from the sample image:

```text
Shape:           (600, 800)
Total pixels:    480000
Minimum:         20
Maximum:         248
Mean:            123.04
Median:          129.0
Peak intensity:  153
Peak frequency:  5000
Histogram total: 480000
```

These values provide a numerical summary of the image.

| Measurement | Result | Interpretation |
| ----------- | -----: | -------------- |
| Minimum | 20 | Darkest pixel |
| Maximum | 248 | Brightest pixel |
| Mean | 123.04 | Average brightness |
| Median | 129 | Middle intensity |
| Peak intensity | 153 | Most frequent brightness |
| Peak frequency | 5,000 | Pixels with intensity 153 |

The image uses a wide intensity range from 20 to 248, indicating that it contains both relatively dark and relatively bright regions.

---

## 11. Verifying the Histogram

A useful validation is to compare the histogram total with the number of pixels.

```text
Histogram total = 480,000
Image pixels    = 600 × 800
                = 480,000
```

The values are equal.

This confirms that every pixel was assigned to exactly one histogram bin.

In general:

```text
Sum of all histogram frequencies = Total number of image pixels
```

This is a simple but valuable way to verify histogram calculations.

---

## 12. Understanding the Mean and Median

The mean intensity is the average of all pixel values.

```text
Mean = 123.04
```

The median is the middle value after all pixel intensities are sorted.

```text
Median = 129
```

Because the mean and median are relatively close, the center of the intensity distribution appears reasonably balanced. However, the complete histogram is needed to assess the distribution's shape and skewness.

The multiple histogram peaks also show that the distribution is not a single smooth group.

The image contains several distinct intensity populations.

---

## 13. Understanding the Peak Intensity

The most frequent pixel intensity is:

```text
153
```

Its frequency is:

```text
5,000 pixels
```

This means that more pixels have an intensity of 153 than any other individual intensity value.

The tallest histogram bar therefore appears at intensity 153.

A peak does not necessarily represent one object. Different regions can have the same intensity and contribute to the same histogram bin.

---

## 14. Brightness and Contrast in a Histogram

A histogram can help us make general observations about brightness and contrast.

### Dark image

Most values are concentrated on the left:

```text
0 ←██████────────→ 255
```

### Bright image

Most values are concentrated on the right:

```text
0 ←────────██████→ 255
```

### Low-contrast image

Values occupy a narrow range:

```text
0 ←─────████─────→ 255
```

### High-contrast image

Values are spread across a wide range:

```text
0 ←██──██──██──██→ 255
```

The sample image spans from 20 to 248, so it uses a broad portion of the available 8-bit intensity range.

---

## 15. What a Histogram Cannot Show

A histogram summarizes how many pixels have each intensity, but it removes spatial information.

It cannot tell us:

- where a bright pixel is located
- which object produced a histogram peak
- whether pixels form an edge or shape
- whether two regions are connected
- what the image subject is

Two different images can have the same histogram if they contain the same number of pixels at each intensity.

Therefore, a histogram is a useful statistical description, but it is not a complete representation of image structure.

---

## 16. Matplotlib Histogram vs. OpenCV Histogram

This example uses Matplotlib:

```python
plt.hist(img.ravel(), bins=256, range=[0, 256])
```

OpenCV also provides:

```python
hist = cv2.calcHist(
    [img],
    [0],
    None,
    [256],
    [0, 256]
)
```

Both approaches can calculate a grayscale histogram.

| Method | Main advantage |
| ------ | -------------- |
| `plt.hist()` | Simple calculation and visualization |
| `cv2.calcHist()` | Efficient histogram calculation in OpenCV workflows |

The current lesson uses `plt.hist()` because it provides a direct and beginner-friendly way to visualize pixel intensity frequencies.

---

## 17. Where Are Histograms Used?

Image histograms are useful in many areas of image processing and computer vision.

Examples include:

- brightness analysis
- contrast measurement
- exposure evaluation
- histogram equalization
- threshold selection
- image comparison
- medical image windowing
- industrial inspection
- preprocessing for segmentation
- detecting changes in illumination

In medical and industrial imaging, histograms can help reveal whether useful pixel information is concentrated in a narrow intensity range.

---

## What I Learned

In this lesson, I learned that:

- A grayscale histogram counts pixels at each intensity.
- An 8-bit image has 256 possible intensity values.
- `img.ravel()` flattens a 2D image into a 1D sequence.
- `plt.hist()` can calculate and display an image histogram.
- The X-axis represents intensity.
- The Y-axis represents pixel frequency.
- Histogram frequencies should sum to the total number of pixels.
- The sample image contains 480,000 pixels.
- Its mean intensity is 123.04.
- Its median intensity is 129.
- Its most frequent intensity is 153.
- A histogram describes intensity distribution but not pixel locations.

The most important idea from this lesson is:

> **A histogram shows how image brightness values are distributed across all pixels.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `01_histogram.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/03_Histogram/src/01_histogram.py)

---

## Next Step

Now that we can analyze an image's intensity distribution, the next step is to reduce noise and small intensity variations.

In the next lesson, we will explore:

- image smoothing
- average blur
- convolution kernels
- `cv2.blur()`
- the effect of kernel size

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
