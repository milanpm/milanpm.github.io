---
layout: post
title: "Comparing Morphological Kernel Sizes in OpenCV"
date: 2026-09-07 14:40:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Structuring Element, Kernel Size]
description: "Compare 3×3, 5×5, and 9×9 elliptical kernels in OpenCV and learn how kernel size affects morphological opening, noise removal, and detail preservation."
---

# Comparing Morphological Kernel Sizes in OpenCV

In the previous post, we compared Rectangle, Ellipse, and Cross structuring elements using the same `5 × 5` kernel size.

That experiment demonstrated that kernel shape controls the geometry and directional behavior of a morphological operation.

However, kernel shape is only one part of the structuring element.

The **kernel size** determines the spatial scale over which the operation examines the image.

A small kernel tends to preserve fine details, while a large kernel removes larger features and produces stronger smoothing.

In this post, we will compare three Ellipse kernels:

```text
3 × 3
5 × 5
9 × 9
```

---

## 1. Why Kernel Size Matters

A morphological kernel defines the neighborhood examined around each pixel.

When the kernel becomes larger, the operation considers a wider area of the image.

For morphological opening, this produces two main effects:

1. erosion removes foreground regions that cannot contain the kernel
2. dilation restores the remaining regions using the same kernel

Opening is defined as:

```text
Opening(A, B) = Dilation(Erosion(A, B), B)
```

where:

- `A` is the binary image
- `B` is the structuring element

A larger structuring element requires a larger foreground neighborhood to survive the erosion stage.

Therefore, larger kernels generally remove more white pixels.

---

## 2. Controlled Comparison

To isolate the effect of kernel size, every other experimental condition must remain unchanged.

| Parameter | Fixed value |
|---|---|
| Source image | `images/sample.png` |
| Image size | `800 × 600` |
| Threshold | `127` |
| Maximum binary value | `255` |
| Operation | Morphological opening |
| Kernel shape | Ellipse |
| Iterations | 1 |
| Kernel sizes | `3 × 3`, `5 × 5`, `9 × 9` |

This is important because changing the shape and size simultaneously would make it difficult to determine which parameter caused the observed difference.

---

## 3. Creating Ellipse Kernels

OpenCV provides `cv2.getStructuringElement()` for generating standard morphological kernels.

```python
kernel_3x3 = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (3, 3)
)

kernel_5x5 = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (5, 5)
)

kernel_9x9 = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    (9, 9)
)
```

Only the kernel dimensions change.

The kernel shape remains `cv2.MORPH_ELLIPSE`.

---

## 4. Actual Kernel Structures

OpenCV generated the following `3 × 3` Ellipse kernel:

```text
0 1 0
1 1 1
0 1 0
```

The `5 × 5` Ellipse kernel was:

```text
0 0 1 0 0
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
0 0 1 0 0
```

The `9 × 9` Ellipse kernel was:

```text
0 0 0 0 1 0 0 0 0
0 1 1 1 1 1 1 1 0
0 1 1 1 1 1 1 1 0
1 1 1 1 1 1 1 1 1
1 1 1 1 1 1 1 1 1
1 1 1 1 1 1 1 1 1
0 1 1 1 1 1 1 1 0
0 1 1 1 1 1 1 1 0
0 0 0 0 1 0 0 0 0
```

As the kernel size increases, the number of active neighboring positions also increases.

This allows the opening operation to evaluate foreground structures over a wider spatial region.

---

## 5. Applying Morphological Opening

The same binary image is processed with each kernel:

```python
result_3x3 = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel_3x3
)

result_5x5 = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel_5x5
)

result_9x9 = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    kernel_9x9
)
```

Because the input image and operation are identical, differences between the output images are caused by kernel size.

---

## 6. Visual Comparison

The following image compares the original binary image with the three opening results:

![Morphological opening comparison using 3x3, 5x5, and 9x9 Ellipse kernels](/assets/images/posts/post-17-kernel-sizes/kernel_size_comparison_3x3_5x5_9x9.png)

The four panels show:

- top left: original binary image
- top right: Ellipse `3 × 3`
- bottom left: Ellipse `5 × 5`
- bottom right: Ellipse `9 × 9`

The `3 × 3` result remains visually close to the original image.

The `5 × 5` kernel removes more small regions and weak connections.

The `9 × 9` kernel produces the strongest cleanup, but it also removes more fine foreground details.

---

## 7. Measuring White Pixels

Visual inspection is useful, but small differences can be difficult to evaluate accurately.

The number of white pixels can be measured with:

```python
original_white_pixels = cv2.countNonZero(binary)

white_pixels_3x3 = cv2.countNonZero(result_3x3)
white_pixels_5x5 = cv2.countNonZero(result_5x5)
white_pixels_9x9 = cv2.countNonZero(result_9x9)
```

The number of removed pixels is calculated by subtracting the remaining white pixels from the original count:

```python
removed_3x3 = original_white_pixels - white_pixels_3x3
removed_5x5 = original_white_pixels - white_pixels_5x5
removed_9x9 = original_white_pixels - white_pixels_9x9
```

---

## 8. Quantitative Results

The original binary image contained:

```text
243,363 white pixels
```

The measured results were:

| Ellipse kernel | Remaining white pixels | Removed white pixels | Removal rate |
|---|---:|---:|---:|
| `3 × 3` | 240,517 | 2,846 | 1.17% |
| `5 × 5` | 233,595 | 9,768 | 4.01% |
| `9 × 9` | 224,874 | 18,489 | 7.60% |

The removal strength was:

```text
9 × 9 (7.60%) > 5 × 5 (4.01%) > 3 × 3 (1.17%)
```

The result supports the expected relationship: increasing the kernel size strengthens the morphological opening effect.

---

## 9. Interpreting the Results

### Ellipse 3 × 3

The `3 × 3` kernel removed only 2,846 white pixels.

Its removal rate was `1.17%`.

This kernel provides gentle cleanup and preserves most fine foreground details. It is suitable when the noise is very small and maintaining object structure is important.

### Ellipse 5 × 5

The `5 × 5` kernel removed 9,768 white pixels.

Its removal rate increased to `4.01%`.

This kernel provides a stronger balance between noise removal and detail preservation. It removes more isolated pixels and narrow structures without changing the main subject as aggressively as the `9 × 9` kernel.

### Ellipse 9 × 9

The `9 × 9` kernel removed 18,489 white pixels.

Its removal rate reached `7.60%`.

This kernel removes larger foreground details, thin connections, and small bright regions. It produces stronger cleanup, but important fine structures may also disappear.

---

## 10. Measuring Differences Between Results

The processed images can be compared using `cv2.absdiff()`:

```python
difference_3x3_5x5 = cv2.countNonZero(
    cv2.absdiff(result_3x3, result_5x5)
)

difference_5x5_9x9 = cv2.countNonZero(
    cv2.absdiff(result_5x5, result_9x9)
)

difference_3x3_9x9 = cv2.countNonZero(
    cv2.absdiff(result_3x3, result_9x9)
)
```

The measurements were:

| Comparison | Different pixels |
|---|---:|
| `3 × 3` vs `5 × 5` | 7,186 |
| `5 × 5` vs `9 × 9` | 8,807 |
| `3 × 3` vs `9 × 9` | 15,723 |

The largest difference occurred between the smallest and largest kernels.

This confirms that the accumulated visual change becomes greater as the difference in kernel scale increases.

---

## 11. Why Difference Counts Are Not Simple Subtractions

The difference between the remaining white-pixel counts for `3 × 3` and `5 × 5` is:

```text
240,517 - 233,595 = 6,922
```

However, `cv2.absdiff()` detected:

```text
7,186 different pixels
```

These values are not required to be identical.

The white-pixel count measures only the total amount of foreground in each image.

By contrast, `cv2.absdiff()` measures every pixel position whose value differs between the two results.

Therefore, two images can have similar total foreground areas while placing some foreground pixels at different positions along boundaries and small structures.

This demonstrates why both area measurements and positional difference measurements are useful.

---

## 12. Kernel Size Selection

There is no single kernel size that is best for every image.

A suitable kernel should be selected according to the size of the noise and the structures that must be preserved.

| Goal | Suggested starting point |
|---|---|
| Remove tiny isolated noise | Small kernel such as `3 × 3` |
| Balance cleanup and detail preservation | Medium kernel such as `5 × 5` |
| Remove larger noise or thin structures | Larger kernel such as `9 × 9` |
| Preserve fine medical or industrial features | Begin with the smallest effective kernel |
| Separate objects connected by narrow bridges | Test progressively larger kernels |

The kernel size should be related to the image resolution and the physical size of the target feature.

A `9 × 9` kernel may be large for a low-resolution image but relatively small for a high-resolution image.

---

## 13. Practical Applications

Kernel-size selection is important in many computer-vision tasks.

### Document processing

Opening can remove small printing noise while preserving characters. An excessively large kernel may damage thin character strokes.

### Medical imaging

Small bright artifacts can be reduced before segmentation, but important anatomical structures must not be removed.

### Industrial inspection

Opening can remove reflective specks, dust, or small defects. Kernel size can also be used to distinguish defects according to their approximate scale.

### Object segmentation

Small foreground regions can be eliminated after thresholding. Larger kernels can remove narrow bridges between objects.

### Machine vision

Kernel size can be selected according to the expected dimensions of scratches, particles, holes, or connected components.

In all these cases, the kernel should be chosen using measurable target dimensions rather than arbitrary values.

---

## 14. Complete Processing Flow

The experiment follows this sequence:

```text
Source image
     │
     ▼
Grayscale conversion
     │
     ▼
Binary thresholding
     │
     ├──────────────┬──────────────┐
     ▼              ▼              ▼
Ellipse 3×3      Ellipse 5×5     Ellipse 9×9
opening          opening         opening
     │              │              │
     └──────────────┴──────────────┘
                    │
                    ▼
      Visual and numeric comparison
```

The controlled experiment changes only one parameter: kernel size.

---

## 15. What I Learned

Through this example, I learned that:

- kernel size controls the spatial scale of a morphological operation
- larger kernels generally produce stronger opening effects
- a `3 × 3` kernel preserves more fine details
- a `5 × 5` kernel provides moderate cleanup
- a `9 × 9` kernel removes larger foreground structures
- white-pixel counts quantify the overall removal strength
- `cv2.absdiff()` measures positional differences between results
- difference-pixel counts are not always equal to differences in total white-pixel counts
- kernel size must be selected according to image resolution and target-feature size
- stronger noise removal can also remove meaningful image details

The main lesson is:

> A larger morphological kernel removes structures at a larger scale, but stronger cleanup always increases the risk of losing useful detail.

---

## Source Code

The complete Python example is available on GitHub:

[View `09_compare_kernel_sizes.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/09_compare_kernel_sizes.py)

The example includes:

- binary image preparation
- three Ellipse structuring elements
- morphological opening
- white-pixel measurement
- removal-rate calculation
- pairwise difference measurement
- labeled comparison-image generation
- result saving and display

---

## Previous Post

In the previous post, we compared Rectangle, Ellipse, and Cross kernels while keeping the kernel size fixed:

[Comparing Morphological Kernel Shapes in OpenCV](/image-processing/opencv/2026/09/04/comparing-morphological-kernel-shapes-in-opencv.html)

---

## Next Step

In the next morphology experiment, we will examine how the number of iterations changes the strength of a morphological operation.

We will keep the source image, operation, kernel shape, and kernel size fixed, then compare:

```text
1 iteration
2 iterations
3 iterations
```

This will help distinguish the effect of increasing kernel size from the effect of repeatedly applying the same kernel.
