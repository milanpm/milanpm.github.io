---
layout: post
title: "Comparing Morphological Kernel Shapes in OpenCV"
date: 2026-09-04 20:30:00 +0900
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Morphology, Structuring Element, Kernel Shape]
description: "Compare Rectangle, Ellipse, and Cross structuring elements in OpenCV and learn how kernel shape affects morphological opening results."
---

# Comparing Morphological Kernel Shapes in OpenCV

In the previous morphology posts, we used structuring elements to perform erosion, dilation, opening, closing, morphological gradient, Top-Hat, and Black-Hat transformations.

However, a morphological kernel is defined by more than its size.

Its **shape** also determines:

- which neighboring pixels are examined
- which directions are emphasized
- how strongly thin structures are removed
- how corners and curved boundaries are preserved
- how horizontal, vertical, and diagonal connections are treated

OpenCV provides three common structuring-element shapes:

- Rectangle
- Ellipse
- Cross

In this post, we will apply all three kernel shapes to the same binary image using the same `5 × 5` size and the same morphological opening operation.

This controlled comparison allows us to observe how kernel shape alone changes the result.

---

## 1. What Is a Structuring Element?

A structuring element, commonly called a morphological kernel, is a small matrix used to examine the neighborhood around each image pixel.

For example, a `5 × 5` rectangular kernel contains 25 active positions:

```text
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
```

During a morphological operation, the kernel moves across the image.

The active positions marked with `1` determine which neighboring pixels participate in the calculation.

Therefore, two kernels with the same width and height can still produce different results when their active positions are arranged differently.

---

## 2. The Three OpenCV Kernel Shapes

OpenCV creates standard structuring elements with:

```python
cv2.getStructuringElement(
    shape,
    kernel_size
)
```

The three kernel-shape constants are:

| Kernel | OpenCV constant | Main characteristic |
|---|---|---|
| Rectangle | `cv2.MORPH_RECT` | Uses the entire rectangular neighborhood |
| Ellipse | `cv2.MORPH_ELLIPSE` | Provides a rounded neighborhood |
| Cross | `cv2.MORPH_CROSS` | Uses horizontal and vertical directions |

For this experiment, every kernel uses the same size:

```python
kernel_size = (5, 5)
```

Only the shape changes.

---

## 3. Creating the Kernels

The Rectangle kernel is created with:

```python
rectangle_kernel = cv2.getStructuringElement(
    cv2.MORPH_RECT,
    kernel_size
)
```

Its actual structure is:

```text
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
```

All 25 positions are active.

The Ellipse kernel is created with:

```python
ellipse_kernel = cv2.getStructuringElement(
    cv2.MORPH_ELLIPSE,
    kernel_size
)
```

Its structure is:

```text
0 0 1 0 0
1 1 1 1 1
1 1 1 1 1
1 1 1 1 1
0 0 1 0 0
```

It contains 17 active positions and excludes the outer corner areas.

The Cross kernel is created with:

```python
cross_kernel = cv2.getStructuringElement(
    cv2.MORPH_CROSS,
    kernel_size
)
```

Its structure is:

```text
0 0 1 0 0
0 0 1 0 0
1 1 1 1 1
0 0 1 0 0
0 0 1 0 0
```

It contains nine active positions arranged along the horizontal and vertical axes.

---

## 4. Preparing the Binary Image

The source image is loaded and converted to grayscale:

```python
image = cv2.imread(str(image_path))

gray = cv2.cvtColor(
    image,
    cv2.COLOR_BGR2GRAY
)
```

Binary thresholding is then applied:

```python
_, binary = cv2.threshold(
    gray,
    127,
    255,
    cv2.THRESH_BINARY
)
```

The resulting image contains only two pixel values:

- `0`: black
- `255`: white

The source image size is:

```text
800 × 600
```

The original binary image contains:

```text
243,363 white pixels
```

---

## 5. Why Use Morphological Opening?

Morphological opening is defined as erosion followed by dilation:

$$
A \circ B = (A \ominus B) \oplus B
$$

Here:

- $A$ is the input binary image
- $B$ is the structuring element
- $\ominus$ represents erosion
- $\oplus$ represents dilation
- $\circ$ represents opening

Opening is useful for:

- removing small white noise
- eliminating thin protrusions
- separating weakly connected objects
- smoothing object boundaries

Because erosion must first fit the structuring element inside a white region, the shape of the kernel directly affects which structures survive.

This makes opening a useful operation for comparing kernel shapes.

---

## 6. Applying the Same Operation

The Rectangle kernel is applied with:

```python
rectangle_result = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    rectangle_kernel
)
```

The Ellipse kernel is applied with:

```python
ellipse_result = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    ellipse_kernel
)
```

The Cross kernel is applied with:

```python
cross_result = cv2.morphologyEx(
    binary,
    cv2.MORPH_OPEN,
    cross_kernel
)
```

The input image, operation, kernel size, and iteration count are identical.

The only experimental variable is the kernel shape.

---

## 7. Visual Comparison

The four images are arranged in a `2 × 2` grid:

1. Original binary image
2. Rectangle result
3. Ellipse result
4. Cross result

![Comparison of Rectangle, Ellipse, and Cross morphological kernels]({{ '/assets/images/posts/post-16-kernel-shapes/kernel_shape_comparison_5x5.png' | relative_url }})

The Rectangle result removes the most small-scale white detail.

The Ellipse result produces a slightly softer result and preserves curved structures better.

The Cross result preserves the largest number of white pixels because it examines fewer neighboring positions and emphasizes horizontal and vertical connectivity.

---

## 8. Quantitative Results

The remaining white pixels are measured using:

```python
cv2.countNonZero(result)
```

The experiment produced the following results:

| Kernel | Active positions | Remaining white pixels | Removed white pixels | Removal rate |
|---|---:|---:|---:|---:|
| Original | — | 243,363 | — | — |
| Rectangle | 25 | 230,991 | 12,372 | 5.08% |
| Ellipse | 17 | 233,595 | 9,768 | 4.01% |
| Cross | 9 | 235,526 | 7,837 | 3.22% |

The removal rate is calculated as:

$$
\text{Removal Rate}
=
\frac{\text{Removed White Pixels}}
{\text{Original White Pixels}}
\times 100
$$

For the Rectangle kernel:

$$
\frac{12{,}372}{243{,}363}
\times 100
\approx 5.08\%
$$

For the Ellipse kernel:

$$
\frac{9{,}768}{243{,}363}
\times 100
\approx 4.01\%
$$

For the Cross kernel:

$$
\frac{7{,}837}{243{,}363}
\times 100
\approx 3.22\%
$$

The removal strength was therefore:

```text
Rectangle > Ellipse > Cross
```

---

## 9. Measuring Differences Between Results

The difference between two results is calculated with:

```python
cv2.countNonZero(
    cv2.absdiff(first_result, second_result)
)
```

The measured differences were:

| Comparison | Different pixels |
|---|---:|
| Rectangle vs Ellipse | 3,010 |
| Rectangle vs Cross | 5,147 |
| Ellipse vs Cross | 2,711 |

The largest difference occurred between Rectangle and Cross.

This is reasonable because they have the greatest difference in neighborhood structure:

- Rectangle activates all 25 positions
- Cross activates only nine positions

The two kernels therefore impose very different spatial requirements during erosion and opening.

---

## 10. Understanding the Rectangle Kernel

The Rectangle kernel examines the complete `5 × 5` neighborhood.

Its advantages include:

- strong removal of small white noise
- strong smoothing of irregular boundaries
- effective removal of narrow protrusions
- simple and predictable behavior

Its limitations include:

- greater loss of thin structures
- possible damage to rounded boundaries
- stronger removal of diagonal details
- tendency to produce more angular results

In this experiment, Rectangle removed:

```text
12,372 white pixels
```

This was the largest reduction among the three kernels.

Rectangle is useful when aggressive cleanup is more important than preserving fine shape details.

---

## 11. Understanding the Ellipse Kernel

The Ellipse kernel excludes the outer corners of the rectangular neighborhood.

Its advantages include:

- smoother treatment of curved boundaries
- better preservation of rounded objects
- less aggressive removal than Rectangle
- balanced behavior across multiple directions

Its limitations include:

- weaker cleanup than a full Rectangle kernel
- possible loss of very thin structures
- behavior still depends strongly on kernel size

In this experiment, Ellipse removed:

```text
9,768 white pixels
```

Its result was between Rectangle and Cross.

Ellipse is often a practical default when objects contain natural curves, circles, cells, particles, or anatomical structures.

---

## 12. Understanding the Cross Kernel

The Cross kernel uses only the center row and center column.

Its advantages include:

- preservation of horizontal and vertical connections
- less aggressive removal of image content
- useful directional behavior
- retention of more thin structures

Its limitations include:

- reduced sensitivity to diagonal neighborhoods
- directional bias
- weaker removal of some noise patterns
- possible preservation of unwanted axis-aligned details

In this experiment, Cross removed:

```text
7,837 white pixels
```

This was the smallest reduction.

Cross is useful when horizontal and vertical connectivity is important or when the operation should preserve more of the original structure.

---

## 13. Practical Kernel Selection

There is no single kernel shape that is best for every image.

A suitable kernel should reflect the geometry of the target feature.

| Image or task | Suggested kernel |
|---|---|
| Rectangular industrial objects | Rectangle |
| Circular particles or cells | Ellipse |
| Curved anatomical structures | Ellipse |
| Horizontal and vertical line networks | Cross |
| Strong general-purpose noise removal | Rectangle |
| Shape-preserving cleanup | Ellipse |
| Directional connectivity analysis | Cross |

For example:

- a Rectangle kernel may be suitable for inspecting manufactured parts with straight edges
- an Ellipse kernel may be more appropriate for biological cells or rounded defects
- a Cross kernel may help analyze grid-like or axis-aligned structures

Kernel selection should be verified using both visual inspection and quantitative measurements.

---

## 14. Kernel Shape and Kernel Size Are Different Parameters

Kernel shape and kernel size influence the result in different ways.

Kernel size determines the spatial scale of the operation:

```text
3 × 3 < 5 × 5 < 9 × 9
```

A larger kernel generally affects larger structures.

Kernel shape determines the geometry and directional behavior of the neighborhood:

```text
Rectangle ≠ Ellipse ≠ Cross
```

Therefore, using the same `5 × 5` size does not make the kernels equivalent.

A complete morphology experiment should consider:

- operation type
- kernel size
- kernel shape
- number of iterations
- image resolution
- size and orientation of the target feature

---

## 15. Complete Processing Flow

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
Rectangle         Ellipse         Cross
opening           opening         opening
     │              │              │
     └──────────────┴──────────────┘
                    │
                    ▼
      Visual and numeric comparison
```

Keeping every other parameter constant makes it possible to attribute the differences specifically to kernel shape.

---

## 16. What I Learned

Through this example, I learned that:

- morphological kernels have both size and shape
- OpenCV provides Rectangle, Ellipse, and Cross structuring elements
- kernels with the same size can contain different active neighborhoods
- Rectangle produced the strongest removal in this experiment
- Ellipse balanced cleanup and curved-shape preservation
- Cross preserved the greatest number of white pixels
- kernel shape introduces directional and geometric behavior
- visual comparison alone may hide small differences
- white-pixel counts provide a simple quantitative measurement
- the kernel should be selected according to the target object and application

The main lesson is:

> Kernel size controls the scale of a morphological operation, while kernel shape controls its geometry and directional behavior.

---

## Source Code

The complete Python example is available on GitHub:

[View `08_compare_kernel_shapes.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/06_Morphology/src/08_compare_kernel_shapes.py)

The example includes:

- binary image preparation
- three `5 × 5` structuring elements
- morphological opening
- white-pixel measurement
- pairwise difference measurement
- labeled comparison-image generation
- result saving and display

---

## Next Step

In the next post, we will compare different morphological kernel sizes such as:

```text
3 × 3
5 × 5
9 × 9
```

We will keep the kernel shape and morphological operation fixed so that we can examine how kernel size changes noise removal, detail preservation, and boundary deformation.
