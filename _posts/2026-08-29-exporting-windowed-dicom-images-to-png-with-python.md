---
layout: post
title: "Exporting Windowed DICOM Images to PNG with Python"
date: 2026-08-29 23:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, PNG Export, PyQt5, pydicom, Medical Imaging]
---

# Exporting Windowed DICOM Images to PNG with Python

DICOM images cannot always be used directly in ordinary documents, web pages, presentations, or image-processing tools.

A common image format such as PNG is more convenient for those purposes.

In this lesson, I added a PNG export feature to the Python DICOM viewer.

The application can now:

- load a DICOM image
- use the current Window Center and Window Width
- convert the selected intensity range to 8-bit grayscale
- create a PyQt5 `QImage`
- select an output path
- save the displayed result as a PNG file
- report whether the save operation succeeded
- keep generated PNG files out of the source repository

The important point is that this feature exports the current display rendering rather than the original medical pixel data.

---

## 1. DICOM and PNG Serve Different Purposes

DICOM is designed for medical imaging workflows.

It can contain:

- medical pixel data
- patient information
- study and series identifiers
- equipment information
- image geometry
- calibration data
- display parameters
- acquisition details

PNG is a general-purpose image format.

It normally stores:

- image pixels
- color or grayscale information
- optional general-purpose image metadata

A PNG export is therefore a convenient visual representation, not a replacement for the source DICOM object.

---

## 2. What This Feature Exports

The export feature saves:

```text
DICOM Pixel Data
        ↓
Current Window Center and Window Width
        ↓
Clipped intensity range
        ↓
Normalized 8-bit grayscale values
        ↓
PNG image
```

This means the saved PNG represents what the viewer is displaying with the current window settings.

Changing Window Center or Window Width before exporting can produce a different PNG from the same DICOM file.

---

## 3. What Is Not Preserved

The PNG does not preserve the complete DICOM dataset.

Information that is not carried into this basic export includes:

- Patient ID
- Patient Name
- Study Instance UID
- Series Instance UID
- SOP Instance UID
- Modality
- Study Date
- Pixel Spacing
- acquisition parameters
- original bit depth
- original stored pixel values
- DICOM private elements

The original `.dcm` file must be retained when medical context, traceability, geometry, or quantitative data is required.

---

## 4. Adding the Save PNG Button

The viewer creates a new button:

```python
self.save_png_button = QPushButton("Save PNG")
self.save_png_button.setEnabled(False)
self.save_png_button.clicked.connect(self.save_png)
```

The button is initially disabled.

This prevents the user from attempting to export before an image is available.

The button is added to the control panel:

```python
control_layout.addWidget(self.open_button)
control_layout.addWidget(self.save_png_button)
control_layout.addWidget(self.anonymize_button)
```

---

## 5. Enabling Export After Loading

When DICOM loading succeeds, the button becomes enabled.

```python
self.save_png_button.setEnabled(True)
```

The state transition is:

```text
Application starts
    ↓
Save PNG disabled
    ↓
DICOM file loads successfully
    ↓
Save PNG enabled
```

If file loading fails, the button remains unavailable.

This keeps the UI state consistent with the application data.

---

## 6. Validating the Viewer State

The save method checks both the pixel array and source path.

```python
if self.pixel_array is None or not self.current_file_path:
    QMessageBox.warning(
        self,
        "No DICOM File",
        "Please open a DICOM file first.",
    )
    return
```

Both values are required:

| State | Why it is needed |
| --- | --- |
| `pixel_array` | Provides the image values to export |
| `current_file_path` | Provides a default PNG filename and location |

The guard also protects the method if it is called programmatically while the button is disabled.

---

## 7. Creating the Default PNG Path

The source path is converted to a `Path` object.

```python
source_path = Path(self.current_file_path)
```

The extension is replaced with `.png`:

```python
default_output_path = source_path.with_suffix(".png")
```

Example:

```text
Input:  samples/test_image.dcm
Output: samples/test_image.png
```

`with_suffix()` changes the file extension without manually manipulating the filename string.

---

## 8. Selecting the Output Location

The application opens a save dialog:

```python
output_path, _ = QFileDialog.getSaveFileName(
    self,
    "Save PNG Image",
    str(default_output_path),
    "PNG Images (*.png);;All Files (*)",
)
```

The dialog provides:

- a suggested filename
- a PNG file filter
- the option to choose another directory
- a way to cancel the operation

If the user cancels, the method returns:

```python
if not output_path:
    return
```

No output file is created.

---

## 9. Adding a Missing Extension

If the selected filename has no suffix, `.png` is appended.

```python
if not Path(output_path).suffix:
    output_path += ".png"
```

Example:

```text
Selected name: dicom_export
Saved name:    dicom_export.png
```

A future improvement should also handle a conflicting suffix such as `.jpg`, because the current method explicitly writes PNG data even if another extension is entered.

---

## 10. Applying the Current Window Settings

The export uses the current values from the spin boxes.

```python
windowed = apply_window(
    self.pixel_array,
    self.window_center_spin.value(),
    self.window_width_spin.value(),
)
```

The same `apply_window()` function is used for display and export.

This keeps the saved image consistent with the visible rendering.

The window calculation is:

```text
window_min = center - width / 2
window_max = center + width / 2
```

Pixel values are then clipped, normalized, and converted to `uint8`.

---

## 11. Windowing Function

The windowing function returns an 8-bit array.

```python
def apply_window(
    pixel_array: np.ndarray,
    window_center: float,
    window_width: float,
) -> np.ndarray:
    if window_width <= 0:
        raise ValueError(
            "Window width must be greater than 0."
        )

    window_min = window_center - window_width / 2
    window_max = window_center + window_width / 2

    windowed = np.clip(
        pixel_array,
        window_min,
        window_max,
    )

    windowed = (
        windowed - window_min
    ) / (
        window_max - window_min
    )

    windowed = (
        windowed * 255
    ).astype(np.uint8)

    return windowed
```

The result contains grayscale values between 0 and 255.

---

## 12. Making the Array Contiguous

Before constructing the `QImage`, the array is made contiguous.

```python
windowed = np.ascontiguousarray(windowed)
```

A NumPy view may use a non-contiguous memory layout.

Qt expects predictable row-by-row memory access.

`np.ascontiguousarray()` ensures that the exported data is arranged appropriately.

---

## 13. Reading Image Dimensions

The array shape is unpacked as:

```python
height, width = windowed.shape
```

For the generated test image:

```text
Height: 512
Width:  512
```

The viewer handles a two-dimensional grayscale array.

Color DICOM images would require additional handling for samples, channels, planar configuration, and photometric interpretation.

---

## 14. Creating a Grayscale QImage

The windowed NumPy array is converted to a `QImage`.

```python
image = QImage(
    windowed.data,
    width,
    height,
    windowed.strides[0],
    QImage.Format_Grayscale8,
).copy()
```

The arguments describe:

| Argument | Meaning |
| --- | --- |
| `windowed.data` | Address of the NumPy pixel data |
| `width` | Number of columns |
| `height` | Number of rows |
| `windowed.strides[0]` | Bytes used by one array row |
| `Format_Grayscale8` | One 8-bit grayscale value per pixel |

The `.copy()` call gives the `QImage` ownership of its own pixel buffer.

This avoids depending on the lifetime of the NumPy array.

---

## 15. Saving the QImage as PNG

The image is saved with:

```python
image.save(output_path, "PNG")
```

The second argument explicitly selects the PNG format.

`QImage.save()` returns a Boolean result.

```text
True  → save succeeded
False → save failed
```

This return value allows the viewer to provide accurate feedback.

---

## 16. Reporting a Successful Save

When the operation succeeds, the viewer displays:

```python
QMessageBox.information(
    self,
    "PNG Save Complete",
    f"PNG image saved:\n{output_path}",
)
```

The message includes the full output path so the user can locate the new file.

---

## 17. Reporting a Save Failure

When `QImage.save()` returns `False`, the viewer displays:

```python
QMessageBox.critical(
    self,
    "PNG Save Error",
    "Failed to save the PNG image.",
)
```

Possible causes include:

- invalid output path
- missing directory permission
- storage failure
- unsupported destination
- unavailable image writer
- conflicting filename or format

A clear failure message is better than silently assuming the file was saved.

---

## 18. Complete Save Method

The complete method is:

```python
def save_png(self):
    """현재 Window 설정이 적용된 영상을 PNG로 저장합니다."""
    if (
        self.pixel_array is None
        or not self.current_file_path
    ):
        QMessageBox.warning(
            self,
            "No DICOM File",
            "Please open a DICOM file first.",
        )
        return

    source_path = Path(self.current_file_path)
    default_output_path = source_path.with_suffix(".png")

    output_path, _ = QFileDialog.getSaveFileName(
        self,
        "Save PNG Image",
        str(default_output_path),
        "PNG Images (*.png);;All Files (*)",
    )

    if not output_path:
        return

    if not Path(output_path).suffix:
        output_path += ".png"

    windowed = apply_window(
        self.pixel_array,
        self.window_center_spin.value(),
        self.window_width_spin.value(),
    )

    windowed = np.ascontiguousarray(windowed)
    height, width = windowed.shape

    image = QImage(
        windowed.data,
        width,
        height,
        windowed.strides[0],
        QImage.Format_Grayscale8,
    ).copy()

    if image.save(output_path, "PNG"):
        QMessageBox.information(
            self,
            "PNG Save Complete",
            f"PNG image saved:\n{output_path}",
        )
    else:
        QMessageBox.critical(
            self,
            "PNG Save Error",
            "Failed to save the PNG image.",
        )
```

This method reuses the existing display pipeline rather than introducing a second image-processing implementation.

---

## 19. Export Result

The generated DICOM contains a horizontal 16-bit intensity gradient.

After windowing, the exported PNG is a 512×512 8-bit grayscale image.

![PNG image exported from the windowed DICOM pixel array](/assets/images/posts/post-08/dicom-test-image.png)

*The exported PNG represents the currently selected DICOM intensity window as an 8-bit grayscale image.*

The exported image is convenient for:

- blog posts
- documentation
- presentations
- visual inspection
- thumbnails
- non-medical image tools
- demonstration data

It should not be treated as the original diagnostic dataset.

---

## 20. Original Values vs. Exported Values

The source test DICOM uses 16-bit unsigned pixel storage.

Conceptually:

```text
Original DICOM range: 0–65535
Exported PNG range:   0–255
```

The conversion is not lossless with respect to the original pixel values.

Many original intensities map to the same 8-bit output value.

The PNG is designed for viewing rather than recovering the original measurement data.

---

## 21. Effect of Window Settings

Suppose two PNG files are exported from the same DICOM image.

```text
Export A:
Window Center = 32768
Window Width  = 65536

Export B:
Window Center = 20000
Window Width  = 10000
```

The files can look very different because they emphasize different intensity ranges.

This is expected.

Windowing controls presentation rather than changing the original DICOM Pixel Data.

---

## 22. PNG Export and Patient Privacy

PNG removes most DICOM metadata in this basic workflow, but that does not automatically guarantee anonymity.

Identifying information may be:

- burned directly into image pixels
- visible in annotations
- included in screen captures
- added through other PNG metadata
- encoded in the output filename

Before sharing an exported image, the pixels and filename should be inspected.

Metadata removal alone cannot erase visible text embedded in the image.

---

## 23. Ignoring Generated PNG Files

The project adds this rule to `.gitignore`:

```gitignore
samples/*.png
```

Generated PNG images are runtime outputs rather than source code.

Ignoring them helps prevent accidental commits after testing the export feature.

The source DICOM generator and application code remain version-controlled.

Specific documentation images can still be deliberately copied into a blog asset directory when needed.

---

## 24. Why Reuse the Display Pipeline?

The viewer uses `apply_window()` for both display and export.

This provides:

- consistent brightness and contrast
- less duplicated code
- easier maintenance
- one place to validate Window Width
- predictable output

If display and export used different conversion logic, the saved PNG might not match what the user saw.

Shared processing reduces that risk.

---

## 25. Possible Improvements

The PNG export feature can later support:

- forcing the `.png` extension
- validating writable output directories
- preserving selected non-sensitive metadata separately
- exporting multiple window presets
- exporting original-resolution images
- handling MONOCHROME1 inversion
- applying Rescale Slope and Rescale Intercept
- handling color DICOM images
- handling multi-frame datasets
- adding annotations optionally
- removing burned-in identifiers
- adding an export preview
- batch conversion
- adding unit tests for saved pixel values

Medical image conversion requires careful handling beyond a basic grayscale example.

---

## What I Learned

In this lesson, I learned that:

- DICOM and PNG serve different purposes.
- PNG export does not preserve the complete DICOM dataset.
- the current Window Center and Width determine the exported appearance.
- `Path.with_suffix()` creates a convenient default output name.
- a save button should remain disabled until an image is loaded.
- `np.ascontiguousarray()` provides a predictable memory layout.
- `QImage.Format_Grayscale8` represents an 8-bit grayscale image.
- `QImage.copy()` separates the image buffer from the NumPy array.
- `QImage.save()` reports success as a Boolean value.
- a windowed PNG is not equivalent to original DICOM pixel data.
- converting 16-bit values to 8-bit loses numerical precision.
- exported images must still be checked for burned-in patient information.
- generated files should normally be excluded from source control.

The most important idea from this lesson is:

> **A PNG export is a windowed visual representation of a DICOM image, not a replacement for the original medical dataset.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 5 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/f07a9b9)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I added tag-name and keyword search:

[Searching DICOM Metadata with Python and PyQt5](/medical-imaging/dicom/2026/08/29/searching-dicom-metadata-with-python-and-pyqt5.html)

---

## Next Step

Now that the viewer can export the displayed image, the next step is to navigate large medical images more effectively.

In the next lesson, we will explore:

- zooming in and out
- panning the displayed image
- mouse-wheel interaction
- coordinate transformation
- preserving image quality during navigation

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
