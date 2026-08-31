---
layout: post
title: "Reading DICOM Pixel Values and Hounsfield Units with Python"
date: 2026-08-31 23:30:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, PyQt5, pydicom, Pixel Value, Hounsfield Unit, Medical Imaging]
description: "Learn how to map PyQt5 mouse coordinates to DICOM image pixels and display stored pixel values and rescaled CT Hounsfield Units while preserving accuracy during zoom and pan."
---

An interactive DICOM viewer should do more than display an image. It should also help users inspect the data behind individual image locations.

When the mouse moves over a medical image, the viewer can report:

- the image coordinates
- the stored DICOM pixel value
- the rescaled modality value
- the Hounsfield Unit for applicable CT images

In the previous lesson, I implemented interactive Window Center and Window Width adjustment. In this lesson, I will add pixel inspection while preserving coordinate accuracy during zoom and pan.

---

## 1. Why Pixel Inspection Matters

Pixel inspection is useful when developing and testing:

- DICOM viewers
- CT image tools
- PACS workstations
- modality integrations
- image-processing algorithms
- segmentation tools
- measurement features
- Window/Level behavior
- dataset validation utilities

It helps answer questions such as:

```text
Which image pixel is under the mouse pointer?
What value is stored in the DICOM Pixel Data?
What value results after rescale conversion?
Does this CT pixel represent air, water, soft tissue, or bone?
Does coordinate mapping remain correct after zooming and panning?
```

The key difficulty is that mouse coordinates, scene coordinates, and image coordinates are not necessarily the same.

---

## 2. The Three Coordinate Systems

The viewer uses three related coordinate spaces:

| Coordinate space | Meaning |
|---|---|
| View coordinates | Mouse position inside the `QGraphicsView` widget |
| Scene coordinates | Position inside the `QGraphicsScene` |
| Image-item coordinates | Position relative to the displayed pixmap |

The conversion path is:

```text
Mouse event position
        ↓ mapToScene()
Scene position
        ↓ pixmap_item.mapFromScene()
Image position
        ↓ integer conversion
Pixel column x and row y
```

This transformation is essential because zoom and pan change the relationship between the viewport and the image.

Using `event.pos()` directly as a pixel location would produce incorrect results after the image is scaled or moved.

---

## 3. Adding a Pixel-Position Signal

`ImageView` declares a signal that carries the image column and row:

```python
pixel_position_changed = pyqtSignal(int, int)
```

The values are emitted as:

```text
x = image column
y = image row
```

This follows the visual coordinate convention, where `x` increases horizontally and `y` increases vertically.

NumPy arrays use the opposite indexing order when reading an element:

```python
pixel_array[y, x]
```

Remembering this difference prevents a common coordinate bug.

---

## 4. Enabling Mouse Tracking

Mouse tracking allows `mouseMoveEvent()` to run even when no mouse button is pressed:

```python
self.setMouseTracking(True)
```

Without mouse tracking, the widget would normally receive movement events only while a button is held. Pixel inspection should follow ordinary pointer movement, so tracking must be enabled.

---

## 5. Mapping the Mouse Position to the Image

The coordinate conversion is added to `mouseMoveEvent()`:

```python
if not self.pixmap_item.pixmap().isNull():
    scene_pos = self.mapToScene(event.pos())
    image_pos = self.pixmap_item.mapFromScene(scene_pos)

    x = int(image_pos.x())
    y = int(image_pos.y())
```

First, the viewport position is mapped to the scene:

```python
scene_pos = self.mapToScene(event.pos())
```

Then, the scene position is mapped into the pixmap item's local coordinate system:

```python
image_pos = self.pixmap_item.mapFromScene(scene_pos)
```

Finally, the floating-point position is converted into integer indices:

```python
x = int(image_pos.x())
y = int(image_pos.y())
```

Because Qt's view transformation is included in these mapping functions, the resulting location remains aligned with the displayed image during zoom and pan.

---

## 6. Checking Image Boundaries

The mouse can be positioned over the black background or outside the image. The coordinates must therefore be validated before a signal is emitted:

```python
pixmap = self.pixmap_item.pixmap()

if 0 <= x < pixmap.width() and 0 <= y < pixmap.height():
    self.pixel_position_changed.emit(x, y)
```

Valid coordinates satisfy:

```text
0 ≤ x < image width
0 ≤ y < image height
```

This prevents attempts to read outside the NumPy array.

The main window performs another boundary check before accessing pixel data. Although somewhat defensive, the second check protects the data layer if signal inputs or image dimensions change later.

---

## 7. Preserving Window/Level Dragging

The existing right-button Window/Level interaction still has priority:

```python
def mouseMoveEvent(self, event):
    """Handle mouse movement and adjust WW/WL during right-button dragging."""
    if self.adjusting_window and self.window_drag_start is not None:
        delta = event.pos() - self.window_drag_start
        scale = max(abs(self.drag_start_width) / 500.0, 1.0)

        center = self.drag_start_center - delta.y() * scale
        width = max(
            1.0,
            self.drag_start_width + delta.x() * scale,
        )

        self.set_window_values(center, width)
        self.window_changed.emit(center, width)
        event.accept()
        return
```

The early `return` means that a right-button drag adjusts the window instead of also emitting pixel-position updates.

When Window/Level adjustment is not active, the method performs pixel coordinate mapping and then passes the event to the base class:

```python
super().mouseMoveEvent(event)
```

This preserves the other behavior provided by `QGraphicsView`.

---

## 8. Complete Mouse-Movement Implementation

The complete Day 8 method is:

```python
def mouseMoveEvent(self, event):
    """Handle mouse movement and adjust WW/WL during right-button dragging."""
    if self.adjusting_window and self.window_drag_start is not None:
        delta = event.pos() - self.window_drag_start
        scale = max(abs(self.drag_start_width) / 500.0, 1.0)

        center = self.drag_start_center - delta.y() * scale
        width = max(
            1.0,
            self.drag_start_width + delta.x() * scale,
        )

        self.set_window_values(center, width)
        self.window_changed.emit(center, width)
        event.accept()
        return

    if not self.pixmap_item.pixmap().isNull():
        scene_pos = self.mapToScene(event.pos())
        image_pos = self.pixmap_item.mapFromScene(scene_pos)

        x = int(image_pos.x())
        y = int(image_pos.y())

        pixmap = self.pixmap_item.pixmap()

        if 0 <= x < pixmap.width() and 0 <= y < pixmap.height():
            self.pixel_position_changed.emit(x, y)

    super().mouseMoveEvent(event)
```

This one method supports two interaction modes:

```text
Normal mouse movement → Inspect pixel position
Right-button drag     → Adjust Window Center and Width
```

---

## 9. Creating the Pixel Information Label

The main window creates a label with placeholder values:

```python
self.pixel_info_label = QLabel(
    "X: -  Y: -  Pixel: -  HU: -"
)
```

It is added near the existing zoom and window labels:

```python
control_layout.addWidget(self.zoom_label)
control_layout.addWidget(self.window_label)
control_layout.addWidget(self.pixel_info_label)
```

This keeps the current navigation, display, and pixel information visible in one control area.

---

## 10. Connecting the Signal

The image view signal is connected to the main-window slot:

```python
self.image_view.pixel_position_changed.connect(
    self.update_pixel_info
)
```

The flow is:

```text
Mouse movement
    ↓
ImageView maps coordinates
    ↓
pixel_position_changed(x, y)
    ↓
DicomViewer reads the pixel arrays
    ↓
Pixel information label updates
```

Separating coordinate detection from pixel-data access keeps `ImageView` focused on interaction and the main window responsible for DICOM data.

---

## 11. Stored Pixel Values and Rescaled Values

The application keeps access to two different arrays.

### Stored pixel array

```python
raw_pixel_array = self.dataset.pixel_array
```

This array contains the decoded values stored in DICOM Pixel Data. In the user interface, the Day 8 implementation labels this value as `Pixel`.

### Rescaled pixel array

The loader created `self.pixel_array` using:

```python
stored_value * RescaleSlope + RescaleIntercept
```

For applicable CT images, this commonly produces Hounsfield Units.

The two values are read using the same row and column:

```python
raw_value = raw_pixel_array[y, x]
hu_value = self.pixel_array[y, x]
```

The array indexing order is `[y, x]`, not `[x, y]`.

---

## 12. Calculating Hounsfield Units

For a conventional rescaled CT image, the relationship is:

```text
HU = stored_pixel_value × RescaleSlope + RescaleIntercept
```

For example:

```text
Stored Pixel Value = 1024
Rescale Slope      = 1
Rescale Intercept  = -1024

HU = 1024 × 1 - 1024 = 0
```

A value near `0 HU` commonly represents water in CT.

Approximate reference values often discussed for CT include:

| Material or tissue | Approximate HU |
|---|---:|
| Air | -1000 |
| Lung | around -700 |
| Fat | around -100 |
| Water | 0 |
| Soft tissue | roughly 20 to 80 |
| Dense bone | several hundred or higher |

These are general reference ranges, not diagnostic thresholds. Actual values vary with acquisition, reconstruction, calibration, artifacts, and anatomy.

---

## 13. Updating the Pixel Information

The main window reads and displays both values:

```python
def update_pixel_info(self, x, y):
    """Display the stored pixel value and rescaled value at the mouse position."""
    if self.dataset is None or self.pixel_array is None:
        return

    raw_pixel_array = self.dataset.pixel_array

    if not (
        0 <= y < raw_pixel_array.shape[0]
        and 0 <= x < raw_pixel_array.shape[1]
    ):
        return

    raw_value = raw_pixel_array[y, x]
    hu_value = self.pixel_array[y, x]

    self.pixel_info_label.setText(
        f"X: {x}  Y: {y}  "
        f"Pixel: {raw_value}  "
        f"HU: {hu_value:.0f}"
    )
```

The Day 8 interface rounds the rescaled value to zero decimal places:

```python
f"HU: {hu_value:.0f}"
```

This provides a compact readout appropriate for typical integer-like CT HU values.

---

## 14. An Important Naming Limitation

The Day 8 code labels the rescaled value as `HU` for every loaded image:

```python
f"HU: {hu_value:.0f}"
```

This is appropriate only when the dataset and transformation actually produce Hounsfield Units, typically for suitable CT images.

For other modalities, the result should be described more generally as a **rescaled value** or **modality value**. A future improvement could inspect metadata such as Modality and Rescale Type before choosing the label:

```text
CT with HU semantics → HU
Other rescaled data  → Modality Value
Unrescaled data      → Pixel Value
```

This distinction prevents the interface from assigning CT-specific meaning to unrelated image data.

---

## 15. Why Windowing Does Not Change the Reported Value

The displayed pixmap contains an 8-bit windowed representation, but the pixel readout uses the original and rescaled arrays:

```python
raw_value = self.dataset.pixel_array[y, x]
hu_value = self.pixel_array[y, x]
```

It does not read the grayscale value from the displayed pixmap.

This is important because an 8-bit windowed value describes screen brightness, not the original medical-image measurement.

```text
Stored value → Rescale transformation → Modality value
                                      ↓
                              Window transformation
                                      ↓
                           8-bit display grayscale
```

Changing Window Center or Window Width changes the bottom branch only. It should not change the stored or rescaled value reported for the same image coordinate.

---

## 16. Accuracy During Zoom and Pan

Zoom and pan transform the view, not the image array. Coordinate mapping reverses those transformations before selecting the pixel:

```python
scene_pos = self.mapToScene(event.pos())
image_pos = self.pixmap_item.mapFromScene(scene_pos)
```

Therefore, the same anatomical or test-image location should report the same pixel indices and values regardless of the current zoom level or pan offset.

This principle is also necessary for later tools such as:

- distance measurement
- ROI statistics
- annotations
- segmentation painting
- pixel probes
- synchronized crosshairs

---

## 17. Testing Pixel Inspection

Run the viewer:

```bash
python src/main.py
```

Open a synthetic or anonymized DICOM file and test the following:

1. Move the mouse over the image without pressing a button.
2. Confirm that X, Y, Pixel, and HU values update.
3. Move the pointer onto the black background.
4. Confirm that no out-of-range access occurs.
5. Zoom in with the mouse wheel.
6. Inspect the same visible image feature again.
7. Pan with the left mouse button.
8. Confirm that coordinate mapping still follows the image.
9. Adjust Window/Level with the right mouse button.
10. Confirm that the image appearance changes while the underlying value at the same pixel remains unchanged.

Expected display format:

```text
X: 256  Y: 180  Pixel: 1024  HU: 0
```

The exact values depend on the DICOM file.

---

## 18. Common Problems

### Coordinates are wrong after zooming

Do not use viewport coordinates directly as array indices. Convert through the scene and pixmap item.

### X and Y appear reversed

Use visual coordinates as `(x, y)` but index NumPy arrays as `[y, x]`.

### Pixel inspection works only while dragging

Enable mouse tracking:

```python
self.setMouseTracking(True)
```

### An IndexError occurs near the image edge

Validate both pixmap bounds and array bounds before reading a value.

### The reported value changes with Window/Level

Read the stored and rescaled arrays, not the 8-bit windowed display buffer.

### Every modality is labeled HU

Use modality-aware labeling. HU is a CT-specific physical interpretation and should not be assumed for arbitrary DICOM images.

### Pixel inspection is slow

Avoid decoding `dataset.pixel_array` repeatedly for every mouse event in a production implementation. Cache the decoded stored array when the file is loaded.

---

## 19. Possible Improvements

The pixel probe can later support:

- modality-aware value labels
- `RescaleType` display
- floating-point precision options
- MONOCHROME1-aware presentation information
- multi-frame image indexing
- color pixel values
- SUV values for PET when properly derived
- Real World Value Mapping
- pixel padding detection
- crosshair overlays
- click-to-lock inspection
- copying values to the clipboard
- cached stored-pixel arrays
- throttling very frequent mouse events
- displaying physical patient coordinates

These additions would turn the basic readout into a more general medical-image inspection tool.

---

## 20. Medical-Imaging Considerations

A displayed value is meaningful only when its transformation path is understood.

Depending on the DICOM object, software may need to consider:

- signed versus unsigned stored values
- `BitsStored` and `HighBit`
- modality LUT processing
- rescale transformation
- pixel padding values
- VOI transformation
- presentation transformation
- Real World Value Mapping
- modality and SOP Class semantics
- multi-frame functional groups

The simplified Day 8 implementation is valuable for learning, but it is not a complete diagnostic pixel-value pipeline. This educational viewer is not a certified medical device.

---

## What I Learned

In this lesson, I learned that:

- mouse positions begin in viewport coordinates.
- `mapToScene()` converts a view position to a scene position.
- `mapFromScene()` converts a scene position to pixmap-item coordinates.
- correct coordinate mapping preserves accuracy during zoom and pan.
- `setMouseTracking(True)` enables inspection without pressing a button.
- image coordinates use `(x, y)` while NumPy indexing uses `[y, x]`.
- bounds must be checked before reading an array element.
- `dataset.pixel_array` contains decoded stored pixel values.
- the rescaled array contains modality values.
- suitable CT rescale values commonly represent Hounsfield Units.
- HU should not be assumed for every DICOM modality.
- Window/Level changes screen appearance rather than the underlying values.
- Qt signals keep image interaction separate from DICOM data access.
- pixel inspection provides the foundation for measurement and ROI tools.

The most important idea from this lesson is:

> **A reliable DICOM pixel probe must map transformed display coordinates back to the correct image pixel and clearly distinguish stored values from modality-specific rescaled values.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 8 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/a7a7296)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I implemented numeric and right-mouse controls for Window Center and Window Width:

[Interactive DICOM Window Level Adjustment with Python and PyQt5](/medical-imaging/dicom/2026/08/31/interactive-dicom-window-level-adjustment-with-python-and-pyqt5.html)

---

## Next Step

Now that the viewer can inspect individual pixels, the next step is to measure a distance between two image locations.

In the next lesson, we will explore:

- selecting two image points
- drawing a measurement overlay
- calculating pixel distance
- using DICOM Pixel Spacing
- converting pixel distance to millimeters
- preserving measurement accuracy during zoom and pan

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
