---
layout: post
title: "Black-Hat Transformation in OpenCV: Extracting Small Dark Details"
date: 2026-09-03 22:10:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Black-Hat Transformation, Feature Extraction]
description: "Learn how the Black-Hat transformation extracts small dark details by subtracting the original image from morphological closing in OpenCV."
---

# Black-Hat Transformation in OpenCV: Extracting Small Dark Details

In the previous post, we studied the Top-Hat transformation, which extracts small bright details from an image.

The **Black-Hat transformation** performs the complementary operation. It extracts small dark details from a brighter surrounding background.

It is defined as:

```text
Black-Hat = Morphological Closing − Original Image
```

Morphological closing fills or suppresses small dark structures. Subtracting the original image from the closed image reveals the dark details that were affected by closing.

In this post, we will learn how to:

- understand the Black-Hat transformation
- apply Black-Hat to a grayscale image
- create an elliptical structuring element
- calculate morphological closing
- apply `cv2.MORPH_BLACKHAT`
- verify the mathematical definition
- measure the extracted dark pixels
- understand the effect of kernel size
- compare Black-Hat with Top-Hat
- explore practical applications and limitations

---

## 1. What Is the Black-Hat Transformation?

The Black-Hat transformation is the difference between the morphological closing of an image and the original image.

It can be expressed as:

$$
T_{\text{black}}(A) = (A \bullet B) - A
$$

Here:

- $A$ represents the original image
- $B$ represents the structuring element
- $\bullet$ represents morphological closing
- $T_{\text{black}}$ represents the Black-Hat result

Morphological closing consists of dilation followed by erosion:

$$
A \bullet B = (A \oplus B) \ominus B
$$

Therefore, the complete operation is:

$$
T_{\text{black}}(A)
=
\left((A \oplus B) \ominus B\right) - A
$$

The processing flow is:

```text
Original image
      │
      ├───────────────────────────────┐
      │                               │
      └→ Dilation → Erosion → Closing │
                                      │
Closing − Original image ─────────────┘
                  │
                  ▼
          Black-Hat result
```

Closing suppresses dark structures that cannot contain the structuring element. The subtraction step extracts the differences created by closing.

---

## 2. Why Does Black-Hat Extract Dark Details?

Consider a grayscale image containing:

- a relatively bright background
- small dark spots
- thin dark lines
- local shadows
- dark texture or surface defects

Morphological closing expands brighter regions into nearby dark regions during dilation. The following erosion restores the larger bright structures while many small dark details remain filled or reduced.

The closed image is therefore brighter than the original image at locations containing small dark structures.

Subtracting the original image produces:

```text
Closing − Original
```

For an unchanged region:

```text
Closing: 150
Original: 150
Result:     0
```

For a small dark feature:

```text
Closing: 180
Original:  60
Result:   120
```

The dark feature in the original image becomes a bright response in the Black-Hat result.

Therefore:

- black pixels indicate little or no change
- bright pixels indicate dark details affected by closing
- stronger result intensity indicates a larger local dark difference

---

## 3. Loading the Grayscale Image

The source image is loaded with:

```python
image = cv2.imread(str(image_path))
```

The program validates the result:

```python
if image is None:
    print(f"Error: Image file not found: {image_path}")
    raise SystemExit
```

The BGR image is then converted to grayscale:

```python
gray = cv2.cvtColor(
    image,
    cv2.COLOR_BGR2GRAY
)
```

Unlike a binary image, a grayscale image retains intensity values between `0` and `255`.

This allows Black-Hat to represent not only the position of each dark feature but also the strength of its difference from the local background.

---

## 4. Creating a `15 × 15` Elliptical Kernel

The example uses an elliptical structuring element:

```python
kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (15, 15)
)
```

The parameters are:

- `cv2.MORPH_ELLIPSE`: kernel shape
- `(15, 15)`: kernel width and height

An elliptical kernel works well with rounded or naturally shaped details and does not strongly favor only horizontal and vertical directions.

The three common OpenCV kernel shapes are:

| Kernel shape | OpenCV constant | Typical characteristic |
|---|---|---|
| Rectangle | `cv2.MORPH_RECT` | Uses the complete rectangular neighborhood |
| Ellipse | `cv2.MORPH_ELLIPSE` | Suitable for rounded and curved features |
| Cross | `cv2.MORPH_CROSS` | Emphasizes horizontal and vertical connectivity |

The kernel should generally be larger than the dark features that need to be extracted.

---

## 5. Calculating Morphological Closing

Morphological closing is calculated separately:

```python
closing = cv2.morphologyEx(
    gray,
    cv2.MORPH_CLOSE,
    kernel
)
```

Closing performs:

```text
Closing = Dilation → Erosion
```

For a grayscale image:

- dilation selects local maximum values and expands bright regions
- erosion then contracts the expanded regions
- small dark gaps and details may remain filled

The result can serve as a local estimate of the brighter surrounding background.

---

## 6. Applying the Black-Hat Transformation

OpenCV provides Black-Hat through `cv2.morphologyEx()`:

```python
black_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_BLACKHAT,
    kernel
)
```

The arguments are:

- `gray`: input grayscale image
- `cv2.MORPH_BLACKHAT`: requested operation
- `kernel`: structuring element

OpenCV internally calculates:

```text
Black-Hat = Closing − Original
```

The expected result is also calculated manually:

```python
expected_black_hat = cv2.subtract(
    closing,
    gray
)
```

The results are compared with:

```python
results_match = np.array_equal(
    black_hat,
    expected_black_hat
)
```

The program returned:

```text
Black-Hat equals closing minus original: True
```

This confirms that the OpenCV result exactly matches the mathematical definition.

---

## 7. Why Use `cv2.subtract()`?

The images use the `uint8` data type, whose valid range is `0` to `255`.

OpenCV subtraction applies saturated arithmetic:

```python
result = cv2.subtract(image_a, image_b)
```

If the mathematical result is negative, OpenCV limits it to zero.

For example:

```text
50 − 100 = 0
```

Using NumPy subtraction directly with `uint8` data may cause values to wrap around:

```python
result = image_a - image_b
```

A negative value can unexpectedly become a large positive value. Using `cv2.subtract()` prevents this behavior.

For Black-Hat, the closed image should be greater than or equal to the original image under the intended grayscale morphology relationship:

```text
Closing ≥ Original
```

Therefore, the subtraction produces the brightness difference created when small dark details are filled.

---

## 8. Complete Python Example

```python
"""
File: 07_black_hat.py
Author: Alex
Created: 2026-09-03
Last Updated: 2026-09-03

Description:
    Demonstrates the Black-Hat transformation using OpenCV.

    The Black-Hat transformation extracts small dark regions from an
    image by subtracting the original grayscale image from its
    morphological closing:

        Black-Hat = Closing - Original

    Morphological closing fills or suppresses dark structures that are
    smaller than the structuring element. Subtracting the original
    image from the closing reveals the affected dark details.

Processing Steps:
    1. Load the source image.
    2. Convert the image to grayscale.
    3. Create a 15 x 15 elliptical structuring element.
    4. Apply morphological closing.
    5. Calculate the Black-Hat transformation.
    6. Verify that Black-Hat equals closing minus the original.
    7. Measure the extracted dark pixels.
    8. Save and display the result.

Input:
    images/sample.png

Output:
    outputs/06_Morphology/black_hat_15x15.png
"""

import cv2
import numpy as np
from pathlib import Path


ROOT = Path(__file__).resolve().parents[3]
image_path = ROOT / "images" / "sample.png"
output_dir = ROOT / "outputs" / "06_Morphology"
output_path = output_dir / "black_hat_15x15.png"

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

# Apply morphological closing.
closing = cv2.morphologyEx(
    gray,
    cv2.MORPH_CLOSE,
    kernel
)

# Apply the Black-Hat transformation.
black_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_BLACKHAT,
    kernel
)

# Verify the definition: Black-Hat = closing - original.
expected_black_hat = cv2.subtract(closing, gray)
results_match = np.array_equal(black_hat, expected_black_hat)

# Create the output directory when it does not already exist.
output_dir.mkdir(parents=True, exist_ok=True)

# Save the Black-Hat result.
if not cv2.imwrite(str(output_path), black_hat):
    print(f"Error: Failed to save the result: {output_path}")
    raise SystemExit

# Measure the extracted dark pixels.
total_pixels = black_hat.size
extracted_pixels = cv2.countNonZero(black_hat)
extracted_percentage = extracted_pixels / total_pixels * 100
maximum_intensity = int(black_hat.max())
mean_intensity = float(black_hat.mean())

print("Kernel shape: Ellipse")
print(f"Kernel size: {kernel.shape[1]} x {kernel.shape[0]}")
print(f"Image size: {black_hat.shape[1]} x {black_hat.shape[0]}")
print(f"Total pixels: {total_pixels}")
print(f"Extracted dark pixels: {extracted_pixels}")
print(f"Extracted percentage: {extracted_percentage:.2f}%")
print(f"Maximum Black-Hat intensity: {maximum_intensity}")
print(f"Mean Black-Hat intensity: {mean_intensity:.2f}")
print(f"Black-Hat equals closing minus original: {results_match}")
print(f"Saved result: {output_path}")

# Display the original grayscale image, closing, and Black-Hat result.
cv2.imshow("Original Grayscale Image", gray)
cv2.imshow("Morphological Closing", closing)
cv2.imshow("Black-Hat Transformation", black_hat)

cv2.waitKey(0)
cv2.destroyAllWindows()
```

---

## 9. Understanding the Measurements

The program produced:

```text
Kernel shape: Ellipse
Kernel size: 15 x 15
Image size: 800 x 600
Total pixels: 480000
Extracted dark pixels: 398728
Extracted percentage: 83.07%
Maximum Black-Hat intensity: 170
Mean Black-Hat intensity: 9.94
Black-Hat equals closing minus original: True
```

The results can be summarized as follows:

| Measurement | Result |
|---|---:|
| Kernel shape | Ellipse |
| Kernel size | `15 × 15` |
| Image size | `800 × 600` |
| Total pixels | `480,000` |
| Nonzero Black-Hat pixels | `398,728` |
| Nonzero percentage | `83.07%` |
| Maximum Black-Hat intensity | `170` |
| Mean Black-Hat intensity | `9.94` |
| Definition verified | `True` |

The nonzero percentage is:

$$
\frac{398728}{480000} \times 100
\approx 83.07\%
$$

Every Black-Hat pixel greater than zero is counted. Even a difference of one intensity level is included.

Therefore, the high nonzero percentage does not mean that `83.07%` of the image contains strong dark objects.

The mean intensity is only `9.94`, indicating that:

- many pixels have small local intensity differences
- most Black-Hat responses are weak
- some dark details produce much stronger responses
- thresholding may be required to isolate meaningful features

---

## 10. Black-Hat Result

The following image is the result of applying the `15 × 15` elliptical Black-Hat transformation:

![Black-Hat transformation result](/assets/images/posts/post-15-black-hat-transformation/black_hat_15x15.png)

Dark areas indicate locations where closing produced little or no change.

Bright areas indicate dark details that were filled or reduced by morphological closing.

A larger Black-Hat value represents a stronger difference between the closed image and the original image.

---

## 11. Extracting Strong Dark Features

The raw result can contain many weak nonzero values. Thresholding can retain only stronger Black-Hat responses:

```python
_, strong_dark_features = cv2.threshold(
    black_hat,
    30,
    255,
    cv2.THRESH_BINARY
)
```

The thresholding rule is:

$$
dst(x,y) =
\begin{cases}
255, & \text{if } B(x,y) > 30 \\
0, & \text{otherwise}
\end{cases}
$$

The threshold should be selected according to:

- target contrast
- lighting conditions
- image noise
- acceptable false positives
- required detection sensitivity

After thresholding, connected components or contours can be used to count and measure candidate dark objects.

---

## 12. Effect of Kernel Size

Kernel size determines the scale of dark details extracted by Black-Hat.

### Small kernel

```python
small_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (5, 5)
)
```

Possible effects:

- extracts very small dark details
- preserves larger dark structures
- may respond strongly to noise

### Medium kernel

```python
medium_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (15, 15)
)
```

Possible effects:

- extracts small and medium dark details
- provides a useful balance for this example
- produces a local background estimate

### Large kernel

```python
large_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (31, 31)
)
```

Possible effects:

- extracts larger dark structures
- increases the area affected by closing
- may include unwanted background variation
- requires more processing time

The correct kernel is determined by the expected size and shape of the target features.

---

## 13. Top-Hat vs. Black-Hat

Top-Hat and Black-Hat are complementary morphological transformations.

| Transformation | Formula | Extracted feature |
|---|---|---|
| Top-Hat | Original − Opening | Small bright details |
| Black-Hat | Closing − Original | Small dark details |

### Top-Hat example

```python
top_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_TOPHAT,
    kernel
)
```

Top-Hat asks:

```text
Which bright details were removed by opening?
```

### Black-Hat example

```python
black_hat = cv2.morphologyEx(
    gray,
    cv2.MORPH_BLACKHAT,
    kernel
)
```

Black-Hat asks:

```text
Which dark details were filled by closing?
```

The appropriate operation depends on whether the target objects are brighter or darker than their surrounding background.

---

## 14. Comparison with Other Morphological Operations

| Operation | Definition | Primary purpose |
|---|---|---|
| Erosion | Local minimum | Shrink bright foreground |
| Dilation | Local maximum | Expand bright foreground |
| Opening | Erosion followed by dilation | Remove small bright structures |
| Closing | Dilation followed by erosion | Fill small dark structures |
| Morphological gradient | Dilation minus erosion | Highlight object boundaries |
| Top-Hat | Original minus opening | Extract small bright details |
| Black-Hat | Closing minus original | Extract small dark details |

The morphological gradient focuses on boundaries. Top-Hat and Black-Hat focus on local contrast relative to structures selected by the kernel.

---

## 15. Practical Applications

### Uneven illumination correction

Black-Hat can enhance dark objects under a slowly varying bright background.

It is useful when global thresholding fails because background brightness is not uniform.

### Document and text processing

Dark text on a light background can be enhanced before thresholding, OCR, or character segmentation.

### Industrial defect inspection

Black-Hat can emphasize:

- dark scratches
- pits
- cracks
- contamination
- missing reflective material
- dark surface defects

### Medical image processing

Depending on the imaging modality and image polarity, Black-Hat may enhance locally dark anatomical or pathological structures.

The result is a preprocessing feature and must not be treated as a diagnosis without appropriate clinical validation.

### Road and line detection

Dark lines on a bright surface may be enhanced when their width is smaller than the selected structuring element.

### Feature extraction

Black-Hat can be used before:

- binary thresholding
- contour detection
- connected-component analysis
- line detection
- defect measurement
- machine-learning feature extraction

---

## 16. Limitations

### Kernel size is application-dependent

A kernel that is too small may fail to fill the target dark structures.

A kernel that is too large may extract unrelated background changes.

### Noise can be enhanced

Small dark noise may have the same scale as the target features and produce false responses.

Filtering before Black-Hat may be necessary.

### Nonzero pixels do not equal detected objects

A nonzero value only means that closing changed the pixel. It does not prove that the pixel belongs to a meaningful defect or object.

Thresholding and object-level analysis are required for detection.

### Image polarity matters

Black-Hat is appropriate for dark features on brighter surroundings. If the target is bright on a darker background, Top-Hat is usually more suitable.

### Border effects can occur

Morphological operations require a neighborhood around every pixel. Results near image boundaries can depend on border handling.

---

## 17. Key Takeaways

- Black-Hat extracts small dark details from a brighter surrounding background.
- Its definition is `Closing − Original`.
- Closing consists of dilation followed by erosion.
- This example used a `15 × 15` elliptical kernel.
- OpenCV provides Black-Hat through `cv2.MORPH_BLACKHAT`.
- The OpenCV result exactly matched the manually calculated subtraction.
- The image contained `398,728` nonzero Black-Hat pixels, or `83.07%`.
- The maximum Black-Hat intensity was `170`.
- The mean intensity was `9.94`, indicating that many differences were weak.
- Kernel size and shape determine the scale and geometry of extracted features.
- Thresholding can isolate stronger dark features for later analysis.
- Common applications include document processing, defect inspection, illumination correction, and dark feature enhancement.

---

## Source Code

The complete Python example is available in the GitHub repository:

[View the Black-Hat Transformation Source Code on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/07_black_hat.py)

---

## Next Step

In the next post, we will compare morphological kernel shapes and sizes.

We will examine how rectangular, elliptical, and cross-shaped structuring elements affect the results of morphological image processing.
