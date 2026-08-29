---
layout: post
title: "Saving Images in Python with OpenCV: Using cv2.imwrite()"
date: 2026-08-28
categories: [image-processing, opencv]
tags: [Python, OpenCV, Image Processing, Computer Vision, imwrite]
---

# Saving Images in Python with OpenCV

Loading an image is only the beginning of an image-processing workflow.

After processing or modifying an image, we often need to save the result as a new file.

OpenCV provides the `cv2.imwrite()` function for this purpose.

In this post, we will learn how to:

- load an image with `cv2.imread()`
- verify that the image was loaded successfully
- save an image with `cv2.imwrite()`
- understand how the output format is selected
- compare the original and saved images
- verify that their pixel data is identical

---

## 1. Loading the Original Image

Before saving an image, we first need to load it.

```python
import cv2

image_path = "images/sample.png"
img = cv2.imread(image_path)
```

The `cv2.imread()` function reads the image file and converts it into a NumPy array.

For a standard color image, the array normally has the following shape:

```text
(height, width, channels)
```

The sample image used in this example has the following shape:

```text
(600, 800, 3)
```

This means:

```text
Height   = 600 pixels
Width    = 800 pixels
Channels = 3
```

OpenCV uses the BGR channel order for color images.

---

## 2. Checking Whether the Image Was Loaded

If OpenCV cannot find or decode the image, `cv2.imread()` returns `None`.

Therefore, it is important to check the result before continuing.

```python
if img is None:
    print("Error: image file not found.")
    exit()
```

Without this check, later operations may fail because there is no valid image data to process.

Common causes of loading failure include:

- an incorrect file path
- a missing image file
- an unsupported file format
- insufficient file permissions
- running the script from an unexpected directory

Checking for `None` makes the program safer and provides a clearer error message.

---

## 3. Saving an Image with `cv2.imwrite()`

OpenCV saves an image using the `cv2.imwrite()` function.

```python
cv2.imwrite("images/output.png", img)
```

Its basic syntax is:

```python
cv2.imwrite(filename, image)
```

The first argument is the output file path.

The second argument is the NumPy array containing the image data.

In this example:

```text
Input  : images/sample.png
Output : images/output.png
```

The original image is loaded into memory and then written to a new PNG file.

---

## 4. How OpenCV Selects the File Format

OpenCV determines the output format from the file extension.

For example:

```python
cv2.imwrite("images/output.png", img)
cv2.imwrite("images/output.jpg", img)
cv2.imwrite("images/output.bmp", img)
```

These extensions produce different image formats.

| Extension         | Format | Compression          |
| ----------------- | ------ | -------------------- |
| `.png`            | PNG    | Lossless             |
| `.jpg` or `.jpeg` | JPEG   | Lossy                |
| `.bmp`            | Bitmap | Usually uncompressed |
| `.tif` or `.tiff` | TIFF   | Depends on settings  |

PNG is useful when preserving exact pixel values is important.

JPEG generally produces smaller files, but compression can change some pixel values.

For image-processing experiments, PNG is often a good choice because it uses lossless compression.

---

## 5. Complete Example

The following code loads an image and saves it as a new PNG file.

```python
from pathlib import Path
import cv2

ROOT = Path(__file__).resolve().parents[3]
image_path = "images/sample.png"

img = cv2.imread(image_path)

if img is None:
    print("Error: image file not found.")
    exit()

cv2.imwrite("images/output.png", img)
print("Saved: images/output.png")
```

The example can be executed from the project root directory:

```bash
python examples/01_Image_Basics/src/03_image_save.py
```

Output:

```text
Saved: images/output.png
```

The saved file is created at:

```text
images/output.png
```

---

## 6. Original Image

The following image is the original input file.

![Original sample image]({{ '/assets/images/posts/post-03/sample.png' | relative_url }})

File information:

```text
File: images/sample.png
Size: approximately 443 KB
```

---

## 7. Saved Image

The following image was saved using `cv2.imwrite()`.

![Image saved by OpenCV]({{ '/assets/images/posts/post-03/output.png' | relative_url }})

File information:

```text
File: images/output.png
Size: approximately 720 KB
```

The two images look identical, but their file sizes are different.

Why did this happen?

---

## 8. Why Are the File Sizes Different?

`cv2.imwrite()` does not copy the original file byte for byte.

The process is closer to the following:

1. `cv2.imread()` decodes the original PNG.
2. OpenCV stores the decoded pixels in a NumPy array.
3. `cv2.imwrite()` encodes those pixels into a new PNG file.

The newly encoded file may use different:

- compression settings
- metadata
- PNG filters
- internal file structure

As a result, two PNG files can contain identical pixel data while having different file sizes.

This is an important distinction:

> **File equality and pixel equality are not the same thing.**

Two image files can have different binary data and file sizes but still represent exactly the same pixels.

---

## 9. Verifying the Pixel Data

We can compare the original and saved images using NumPy and OpenCV.

```python
import cv2

original = cv2.imread("images/sample.png")
saved = cv2.imread("images/output.png")

print("Shape:", original.shape, saved.shape)
print("Same pixels:", (original == saved).all())
print("Maximum difference:", cv2.absdiff(original, saved).max())
```

Output:

```text
Shape: (600, 800, 3) (600, 800, 3)
Same pixels: True
Maximum difference: 0
```

The results show that:

- both images have the same dimensions
- every corresponding pixel value is equal
- the maximum pixel difference is zero

Therefore, the images contain identical pixel data even though their file sizes differ.

This is possible because PNG uses lossless compression.

---

## 10. Checking the Return Value

The `cv2.imwrite()` function returns a Boolean value.

```python
success = cv2.imwrite("images/output.png", img)
```

The result is:

```text
True  → the image was saved successfully
False → the image could not be saved
```

A safer version of the code can check this value:

```python
success = cv2.imwrite("images/output.png", img)

if success:
    print("Saved: images/output.png")
else:
    print("Error: failed to save the image.")
```

This is better than always printing a success message because it verifies whether the write operation actually succeeded.

---

## 11. A Note About Relative Paths

This example uses relative paths:

```python
image_path = "images/sample.png"
cv2.imwrite("images/output.png", img)
```

Relative paths are interpreted from the directory where the command is executed.

Therefore, this script should be executed from the project root:

```bash
python examples/01_Image_Basics/src/03_image_save.py
```

Running it from another directory may cause OpenCV to fail to find `images/sample.png`.

The code calculates a project root path:

```python
ROOT = Path(__file__).resolve().parents[3]
```

However, the current example does not yet use `ROOT` when creating its input and output paths.

A future improvement could construct paths from `ROOT`, making the script independent of the current working directory.

---

## What I Learned

In this lesson, I learned that:

- `cv2.imread()` loads an image as a NumPy array.
- A failed image load returns `None`.
- `cv2.imwrite()` saves image data to a file.
- The output file extension determines the image format.
- PNG uses lossless compression.
- OpenCV re-encodes an image instead of copying the original file directly.
- Files with different sizes can contain identical pixel data.
- `cv2.imwrite()` returns `True` or `False`.
- Relative paths depend on the directory where the program is executed.

The most important idea from this lesson is:

> **Saving an image means encoding its pixel data into an image file format.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View `03_image_save.py` on GitHub](https://github.com/milanpm/01_ImageProcessing/blob/main/examples/01_Image_Basics/src/03_image_save.py)

---

## Next Step

Now that we can load, inspect, and save an image, the next step is to convert a color image into grayscale.

In the next lesson, we will explore:

- grayscale images
- `cv2.cvtColor()`
- BGR-to-grayscale conversion
- grayscale image dimensions
- why grayscale is useful in image processing

---

*This post is part of my journey from beginner-level image processing toward computer vision and AI vision.*
