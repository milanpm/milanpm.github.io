---
layout: post
title: "Top-Hat Transformation in OpenCV: Extracting Small Bright Details"
date: 2026-09-03 21:45:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Top-Hat Transformation, Feature Extraction]
description: "Learn how the Top-Hat transformation extracts small bright details by subtracting morphological opening from the original image in OpenCV."
---

# Top-Hat Transformation in OpenCV: Extracting Small Bright Details

In the previous lessons, we studied erosion, dilation, opening, closing, and the morphological gradient.

The **Top-Hat transformation** is another morphological operation that extracts bright image details that are smaller than a selected structuring element.

It is defined as:

```text
Top-Hat = Original Image − Morphological Opening
```

Morphological opening removes small bright structures from an image. Subtracting the opened image from the original image reveals the bright details that were removed.

In this post, we will learn how to:

- understand the Top-Hat transformation
- apply Top-Hat to a grayscale image
- create an elliptical structuring element
- calculate morphological opening
- apply `cv2.MORPH_TOPHAT`
- verify the mathematical definition
- measure the extracted bright pixels
- understand the effect of kernel size and shape
- explore practical applications and limitations

---

## 1. What Is the Top-Hat Transformation?

The Top-Hat transformation is the difference between an original image and its morphological opening.

It can be expressed as:

$$
T_{\text{white}}(A) = A - (A \circ B)
$$

Here:

- $A$ represents the original image
- $B$ represents the structuring element
- $\circ$ represents morphological opening
- $T_{\text{white}}$ represents the white Top-Hat result

Morphological opening consists of erosion followed by dilation:

$$
A \circ B = (A \ominus B) \oplus B
$$

Therefore, the complete operation can be written as:

$$
T_{\text{white}}(A)
=
A - \left((A \ominus B) \oplus B\right)
$$

The conceptual processing flow is:

```text
Original image
      │
      ├──────────────────────────────┐
      │                              │
      └→ Erosion → Dilation → Opening│
                                     │
Original image − Opening ────────────┘
                  │
                  ▼
          Top-Hat result
```

The opening operation suppresses bright structures that cannot contain the structuring element. The subtraction step restores only the details removed by opening.

---

## 2. Why Does Top-Hat Extract Bright Details?

Consider a grayscale image containing:

- a slowly changing background
- large bright regions
- small bright features
- local intensity variations

Morphological opening smooths bright regions and removes bright details smaller than the kernel.

Large structures that can contain the kernel mostly remain in the opened image. Small bright details are reduced or removed.

The subtraction then produces:

```text
Original − Opening
```

Pixels that changed very little have values close to zero. Bright structures removed by opening produce larger positive values.

Therefore:

- black or dark pixels indicate little difference
- brighter pixels indicate details removed by opening
- stronger Top-Hat intensity indicates a greater local brightness difference

Unlike the earlier binary morphology examples, Top-Hat is especially useful with grayscale images because it preserves intensity differences.

---

## 3. Loading the Source Image

The source image is loaded with `cv2.imread()`:

```python
image = cv2.imread(str(image_path))
```

The program validates the result:

```python
if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit
```

This prevents later OpenCV functions from receiving an invalid image.

The source image is then converted from BGR color to grayscale:

```python
gray = cv2.cvtColor(
    image,
    cv2.COLOR_BGR2GRAY
)
```

A grayscale image contains one intensity value per pixel, normally ranging from `0` to `255`.

In this example, thresholding is not applied. Retaining the grayscale intensities allows the Top-Hat result to represent the strength of each local brightness difference.

---

## 4. Creating an Elliptical Kernel

The example uses a `15 × 15` elliptical structuring element:

```python
kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (15, 15)
)
```

The arguments are:

- `cv2.MORPH_ELLIPSE`: creates an approximately elliptical kernel
- `(15, 15)`: specifies the kernel width and height

An elliptical kernel is useful when the bright details of interest are rounded, curved, or do not align strongly with horizontal and vertical directions.

Kernel shape affects how the image is analyzed:

| Kernel shape | Typical characteristics |
|---|---|
| Rectangle | Treats every position in the rectangular neighborhood equally |
| Ellipse | Works well with rounded and naturally shaped structures |
| Cross | Emphasizes horizontal and vertical connectivity |

Kernel size is also important. Features smaller than the kernel are more likely to be removed by opening and then revealed by Top-Hat.

---

## 5. Calculating Morphological Opening

Morphological opening is calculated separately so that we can verify the Top-Hat definition:

```python
opening = cv2.morphologyEx(
    gray,
    cv2.MORPH_OPEN,
    kernel
)
```

Opening performs two operations internally:

```text
Opening = Erosion → Dilation
```

For a grayscale image:

- erosion replaces each pixel with a local minimum
- dilation then expands the remaining bright regions using local maxima

This process suppresses small bright details while retaining larger background structures.

The opened image can therefore be treated as an estimate of the slowly varying background when the kernel is appropriately selected.

---

## 6. Applying the Top-Hat Transformation

OpenCV provides the Top-Hat transformation through `cv2.morphologyEx()`:

```python
top_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_TOPHAT,
    kernel
)
```

The arguments are:

- `gray`: input grayscale image
- `cv2.MORPH_TOPHAT`: requested morphological operation
- `kernel`: structuring element

Internally, OpenCV calculates:

```text
Top-Hat = Grayscale Image − Opening
```

The same result is calculated manually:

```python
expected_top_hat = cv2.subtract(
    gray,
    opening
)
```

The two results are compared with:

```python
results_match = np.array_equal(
    top_hat,
    expected_top_hat
)
```

The program returned:

```text
Top-Hat equals original minus opening: True
```

This confirms that the OpenCV Top-Hat result exactly matches the mathematical definition.

---

## 7. Complete Python Example

```python
"""
File: 06_top_hat.py
Author: Alex
Created: 2026-09-03
Last Updated: 2026-09-03

Description:
    Demonstrates the Top-Hat transformation using OpenCV.

    The Top-Hat transformation extracts small bright regions from an
    image by subtracting the morphological opening from the original
    grayscale image:

        Top-Hat = Original - Opening

    Morphological opening removes bright structures that are smaller
    than the structuring element. Subtracting the opened image from the
    original image reveals the removed bright details.

Processing Steps:
    1. Load the source image.
    2. Convert the image to grayscale.
    3. Create a 15 x 15 elliptical structuring element.
    4. Apply morphological opening.
    5. Calculate the Top-Hat transformation.
    6. Verify that Top-Hat equals the original minus the opening.
    7. Measure the extracted bright pixels.
    8. Save and display the result.

Input:
    images/sample.png

Output:
    outputs/06_Morphology/top_hat_15x15.png
"""

import cv2
import numpy as np
from pathlib import Path


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
output_dir = ROOT / "outputs" / "06_Morphology"
output_path = output_dir / "top_hat_15x15.png"

# Load the source image.
image = cv2.imread(str(image_path))

if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit

# Convert the color image to grayscale.
gray = cv2.cvtColor(image, cv2.COLOR_BGR2GRAY)

# Create a 15 x 15 elliptical structuring element.
kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (15, 15)
)

# Apply morphological opening.
opening = cv2.morphologyEx(
    gray,
    cv2.MORPH_OPEN,
    kernel
)

# Apply the Top-Hat transformation.
top_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_TOPHAT,
    kernel
)

# Verify the definition: Top-Hat = original - opening.
expected_top_hat = cv2.subtract(gray, opening)
results_match = np.array_equal(top_hat, expected_top_hat)

# Create the output directory when it does not already exist.
output_dir.mkdir(parents=True, exist_ok=True)

# Save the Top-Hat result.
if not cv2.imwrite(str(output_path), top_hat):
    print(f"Error: Failed to save the result: {output_path}")
    raise SystemExit

# Measure the extracted bright pixels.
total_pixels = top_hat.size
extracted_pixels = cv2.countNonZero(top_hat)
extracted_percentage = extracted_pixels / total_pixels * 100
maximum_intensity = int(top_hat.max())
mean_intensity = float(top_hat.mean())

print(f"Kernel shape: Ellipse")
print(f"Kernel size: {kernel.shape[1]} x {kernel.shape[0]}")
print(f"Image size: {top_hat.shape[1]} x {top_hat.shape[0]}")
print(f"Total pixels: {total_pixels}")
print(f"Extracted bright pixels: {extracted_pixels}")
print(f"Extracted percentage: {extracted_percentage:.2f}%")
print(f"Maximum Top-Hat intensity: {maximum_intensity}")
print(f"Mean Top-Hat intensity: {mean_intensity:.2f}")
print(f"Top-Hat equals original minus opening: {results_match}")
print(f"Saved result: {output_path}")

# Display the original grayscale image, opening, and Top-Hat result.
cv2.imshow("Original Grayscale Image", gray)
cv2.imshow("Morphological Opening", opening)
cv2.imshow("Top-Hat Transformation", top_hat)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 8. Understanding the Measurements

The program produced the following output:

```text
Kernel shape: Ellipse
Kernel size: 15 x 15
Image size: 800 x 600
Total pixels: 480000
Extracted bright pixels: 394348
Extracted percentage: 82.16%
Maximum Top-Hat intensity: 186
Mean Top-Hat intensity: 9.26
Top-Hat equals original minus opening: True
```

The measurements can be summarized as follows:

| Measurement | Result |
|---|---:|
| Kernel shape | Ellipse |
| Kernel size | `15 × 15` |
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Nonzero Top-Hat pixels | `394,348` |
| Nonzero percentage | `82.16%` |
| Maximum Top-Hat intensity | `186` |
| Mean Top-Hat intensity | `9.26` |
| Definition verified | `True` |

The nonzero percentage is calculated as:

$$
\frac{394348}{480000} \times 100
\approx 82.16\%
$$

A nonzero Top-Hat pixel is any pixel for which the original grayscale value is greater than the corresponding opened value.

However, a nonzero value does not necessarily represent a strong feature. Even a difference of one intensity level is counted.

For this reason, the nonzero percentage is relatively high, while the mean Top-Hat intensity remains only `9.26`.

This indicates that:

- many pixels contain small local brightness differences
- most extracted differences are weak
- a smaller number of pixels contain stronger bright details
- additional thresholding may be useful when only prominent features are required

---

## 9. Top-Hat Result

The following image is the result of applying the `15 × 15` elliptical Top-Hat transformation:

![Top-Hat transformation result](/assets/images/posts/post-14-top-hat-transformation/top_hat_15x15.png)

Dark regions indicate pixels that were similar to the morphological opening.

Bright regions indicate image details that were reduced or removed during opening.

A stronger result intensity means that the original pixel was considerably brighter than the locally opened background.

---

## 10. Extracting Only Strong Features

The raw Top-Hat result can contain many weak nonzero values.

If only strong bright features are required, thresholding can be applied after Top-Hat:

```python
_, strong_features = cv2.threshold(
    top_hat,
    30,
    255,
    cv2.THRESH_BINARY
)
```

The rule is:

$$
dst(x,y) =
\begin{cases}
255, & \text{if } T(x,y) > 30 \\
0, & \text{otherwise}
\end{cases}
$$

This converts stronger Top-Hat responses into white pixels and suppresses weaker intensity differences.

The threshold value should be selected according to:

- image contrast
- lighting conditions
- sensor noise
- required feature sensitivity
- false-positive tolerance

For varying images, adaptive or statistically derived thresholds may be more reliable than one fixed value.

---

## 11. Effect of Kernel Size

The kernel determines the scale of the bright details extracted by Top-Hat.

### Small kernel

A small kernel removes only very small bright structures.

```python
small_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (5, 5)
)
```

Possible effects:

- fine details are extracted
- larger structures remain in the opening
- sensitivity to noise may increase

### Medium kernel

```python
medium_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (15, 15)
)
```

Possible effects:

- small and medium local details are extracted
- useful balance between detail and background estimation
- suitable for the current demonstration

### Large kernel

```python
large_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (31, 31)
)
```

Possible effects:

- larger bright structures may be extracted
- more of the image may differ from the opening
- processing time increases
- the estimated background becomes smoother

The kernel should generally be larger than the bright objects that need to be extracted.

---

## 12. Top-Hat Compared with Other Morphological Operations

| Operation | Definition | Primary purpose |
|---|---|---|
| Erosion | Local minimum | Shrink bright foreground regions |
| Dilation | Local maximum | Expand bright foreground regions |
| Opening | Erosion followed by dilation | Remove small bright structures |
| Closing | Dilation followed by erosion | Fill small dark gaps and holes |
| Morphological gradient | Dilation minus erosion | Highlight object boundaries |
| Top-Hat | Original minus opening | Extract small bright details |
| Black-Hat | Closing minus original | Extract small dark details |

Top-Hat and opening are directly related but serve different purposes.

Opening returns the image after small bright details have been suppressed:

```text
Opening = Background and larger structures
```

Top-Hat returns the details removed by opening:

```text
Top-Hat = Removed bright details
```

The morphological gradient responds primarily to boundaries, while Top-Hat responds to local bright structures relative to their surroundings.

---

## 13. Practical Applications

### Uneven illumination correction

A Top-Hat result can emphasize bright objects under a slowly changing background.

This is useful when lighting is brighter in one area and darker in another.

### Industrial defect inspection

Small bright scratches, particles, reflections, or surface defects can be enhanced before thresholding and contour analysis.

### Medical image processing

Top-Hat may help enhance locally bright anatomical or pathological structures, depending on the imaging modality and scale.

It must not be interpreted as a diagnosis by itself. Kernel scale, image calibration, acquisition conditions, and clinical validation remain essential.

### Document processing

Bright or dark background variations can interfere with character segmentation. Morphological background estimation can improve preprocessing before binarization.

### Particle and spot detection

Small bright particles, cells, stars, or fluorescent spots can be enhanced when their sizes are smaller than the selected kernel.

### Feature preprocessing

Top-Hat can be used before:

- thresholding
- connected-component analysis
- contour detection
- blob detection
- object measurement
- machine-learning feature extraction

---

## 14. Limitations

### Kernel selection is scale-dependent

A kernel that is too small may fail to remove the target details during opening.

A kernel that is too large may extract unwanted background variations or larger structures.

### Noise may also be enhanced

Small bright noise can produce strong Top-Hat responses because it has the same scale characteristics as small bright target objects.

Denoising may be required before applying Top-Hat.

### Nonzero pixels are not always meaningful objects

The current measurement counts every pixel greater than zero. Minor interpolation, texture, compression, or illumination differences can therefore increase the count.

Thresholding or connected-component filtering can provide a more meaningful object-level measurement.

### Shape matters

A circular kernel may respond differently from a rectangular or cross-shaped kernel. The kernel shape should reflect the expected geometry of the target features.

### Border effects may occur

Morphological operations require neighborhoods around each pixel. Image borders depend on OpenCV's border-handling behavior and may produce responses that require careful interpretation.

---

## 15. Key Takeaways

- The Top-Hat transformation extracts bright details removed by morphological opening.
- Its mathematical definition is `Original − Opening`.
- This example used a `15 × 15` elliptical kernel.
- OpenCV provides the operation through `cv2.MORPH_TOPHAT`.
- The implementation exactly matched the manually calculated result.
- The example contained `394,348` nonzero Top-Hat pixels, or `82.16%`.
- The mean Top-Hat intensity was only `9.26`, showing that many nonzero differences were weak.
- Kernel size determines the scale of the extracted features.
- Thresholding can isolate stronger Top-Hat responses.
- Common applications include lighting correction, defect inspection, medical image enhancement, and bright spot detection.

---

## Source Code

The complete Python example is available in the GitHub repository:

[View the Top-Hat Transformation Source Code on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/06_top_hat.py)

---

## Next Step

In the next post, we will study the **Black-Hat transformation**.

Black-Hat performs the complementary operation:

```text
Black-Hat = Closing − Original Image
```

While Top-Hat extracts small bright details, Black-Hat extracts small dark details from a brighter surrounding background.
