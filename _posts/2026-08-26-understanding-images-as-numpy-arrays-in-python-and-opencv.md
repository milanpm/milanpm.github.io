---
layout: post
title: "NumPy Arrays in OpenCV: How Digital Images Are Represented"
date: 2026-08-26
categories: [image-processing, opencv]
tags: [python, opencv, numpy, image-processing, computer-vision]
---

# Understanding Images as NumPy Arrays in Python and OpenCV

When we look at a digital image, we see objects, colors, shapes, and patterns.

A computer sees something very different.

It sees **numbers**.

Understanding how an image is represented as numerical data is one of the most important foundations of image processing and computer vision.

In this post, we will learn how Python, NumPy, and OpenCV represent digital images.

---

## 1. An Image Is an Array of Numbers

At its simplest level, a digital image can be represented as a matrix of pixel values.

For example:

```text
0    50   100
150  200  255
30   80   120
```

In Python, we can represent this image using a NumPy array.

```python
import numpy as np

image = np.array([
    [0, 50, 100],
    [150, 200, 255],
    [30, 80, 120]
], dtype=np.uint8)

print(image)
```

Output:

```text
[[  0  50 100]
 [150 200 255]
 [ 30  80 120]]
```

This small matrix can be considered a **3 × 3 grayscale image**.

Each number represents the intensity of one pixel.

This leads to one of the most important concepts in image processing:

> **An image is a NumPy array of pixel values.**

---

## 2. What Is a Pixel?

A pixel is the smallest individual element of a digital image.

In an 8-bit grayscale image, each pixel normally has a value between:

```text
0 ~ 255
```

The value represents brightness.

| Pixel Value | Meaning     |
| ----------: | ----------- |
|           0 | Black       |
|          64 | Dark gray   |
|         128 | Medium gray |
|         192 | Light gray  |
|         255 | White       |

A smaller value represents a darker pixel, while a larger value represents a brighter pixel.

Therefore, many image-processing algorithms are essentially mathematical operations performed on these pixel values.

---

## 3. Checking the Shape of an Image

NumPy arrays have a useful property called `shape`.

```python
print(image.shape)
```

Output:

```text
(3, 3)
```

For a grayscale image, the shape normally follows:

```text
(height, width)
```

Therefore:

```text
(3, 3)
```

means:

```text
Height = 3
Width  = 3
```

For example, an image with a resolution of 640 × 480 will normally have the following NumPy shape:

```text
(480, 640)
```

This can be confusing at first.

Image resolutions are usually described as:

```text
Width × Height
```

but NumPy represents image shape as:

```text
Height × Width
```

This distinction is important when working with OpenCV.

---

## 4. Image Data Type

We created our array using:

```python
dtype=np.uint8
```

We can check its data type:

```python
print(image.dtype)
```

Output:

```text
uint8
```

`uint8` means **unsigned 8-bit integer**.

It can represent values from:

```text
0 to 255
```

This makes it a common data type for standard 8-bit digital images.

Understanding image data types becomes increasingly important when performing operations such as filtering, normalization, arithmetic operations, and image conversion.

---

## 5. Accessing Individual Pixels

Because an image is a NumPy array, we can access pixels using array indexing.

For example:

```python
print(image[0, 0])
```

Output:

```text
0
```

Another example:

```python
print(image[1, 2])
```

Output:

```text
255
```

The indexing format is:

```text
image[row, column]
```

which can also be understood as:

```text
image[y, x]
```

This is another important point for beginners.

Pixel coordinates are often discussed as:

```text
(x, y)
```

but NumPy indexing uses:

```text
[y, x]
```

or:

```text
[row, column]
```

---

## 6. Getting Image Width and Height

For a grayscale image, we can obtain its dimensions using:

```python
height, width = image.shape

print("Width:", width)
print("Height:", height)
```

Output:

```text
Width: 3
Height: 3
```

This technique is frequently used in computer vision programs when calculating positions, regions of interest, image centers, or object coordinates.

---

## 7. How Are Color Images Represented?

A grayscale pixel contains one intensity value.

A color pixel contains multiple channel values.

OpenCV normally represents a color image using three channels:

```text
B = Blue
G = Green
R = Red
```

Therefore, OpenCV uses the channel order:

```text
BGR
```

rather than the commonly known RGB order.

For example:

```python
pixel = [255, 0, 0]
```

In OpenCV BGR format, this represents blue.

Similarly:

```text
[255,   0,   0] → Blue
[  0, 255,   0] → Green
[  0,   0, 255] → Red
```

Remembering the BGR channel order is important when working with OpenCV.

---

## 8. Shape of a Color Image

A color image normally has three dimensions:

```text
(height, width, channels)
```

For example:

```text
(480, 640, 3)
```

means:

```text
Height   = 480
Width    = 640
Channels = 3
```

The three channels correspond to Blue, Green, and Red when using OpenCV.

Therefore, a color image is essentially a three-dimensional NumPy array.

---

## 9. Complete Example

Here is the complete Python example used in this lesson.

```python
import numpy as np


def main():
    image = np.array([
        [0, 50, 100],
        [150, 200, 255],
        [30, 80, 120]
    ], dtype=np.uint8)

    print("Image:")
    print(image)

    print("\nShape:", image.shape)
    print("Data type:", image.dtype)

    print("\nPixel [0, 0]:", image[0, 0])
    print("Pixel [1, 2]:", image[1, 2])

    height, width = image.shape

    print("\nWidth:", width)
    print("Height:", height)


if __name__ == "__main__":
    main()
```

Expected output:

```text
Image:
[[  0  50 100]
 [150 200 255]
 [ 30  80 120]]

Shape: (3, 3)
Data type: uint8

Pixel [0, 0]: 0
Pixel [1, 2]: 255

Width: 3
Height: 3
```

---

## 10. Why Is This Important for Image Processing?

Understanding that an image is an array is fundamental because most image-processing techniques operate directly on pixel values.

For example:

- Thresholding compares pixel values.
- Brightness adjustment changes pixel values.
- Filtering calculates new values from neighboring pixels.
- Edge detection analyzes intensity differences.
- Histograms count pixel intensity distributions.
- Object detection analyzes patterns within image data.

Even advanced computer vision and AI systems ultimately receive numerical image data.

The algorithms become more sophisticated, but the fundamental representation remains numerical.

---

## What I Learned

In this lesson, I learned that:

- A digital image can be represented as a NumPy array.
- A pixel represents an individual element of an image.
- 8-bit grayscale pixel values normally range from 0 to 255.
- Grayscale images usually have the shape `(height, width)`.
- Color images usually have the shape `(height, width, channels)`.
- OpenCV normally uses BGR channel order.
- NumPy indexing uses `image[y, x]`.
- `uint8` is commonly used for standard 8-bit images.

The most important idea from this lesson is simple:

> **Images are data, and image processing is the process of analyzing and transforming that data.**

---

## Next Step

Now that we understand how images are represented as NumPy arrays, the next step is to load a real image with OpenCV and inspect its properties.

In the next lesson, we will explore:

- `cv2.imread()`
- image dimensions
- color channels
- individual pixels
- basic pixel manipulation

This will connect NumPy array concepts with real-world image processing using OpenCV.

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*