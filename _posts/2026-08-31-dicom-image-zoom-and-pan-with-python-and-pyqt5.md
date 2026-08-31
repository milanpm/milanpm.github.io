---
layout: post
title: "DICOM Image Zoom and Pan with Python and PyQt5"
date: 2026-08-31 23:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, PyQt5, Zoom, Pan, Medical Imaging]
description: "Learn how to implement smooth DICOM image zoom and pan controls in a Python and PyQt5 viewer using QGraphicsView, mouse-wheel events, and transformation anchors."
---

A DICOM viewer must allow users to inspect medical images beyond their initial display size.

Large X-ray, CT, and other diagnostic images often contain details that are difficult to examine when the entire image is fitted inside a small application window. Users need to zoom in, zoom out, and move around an enlarged image without changing the original pixel data.

In the previous lesson, I exported the currently displayed DICOM image as a PNG file. In this lesson, I will add interactive **zoom and pan** controls to the PyQt5 viewer.

The completed viewer will support:

- zooming with the mouse wheel
- panning with the left mouse button
- keeping the point under the mouse cursor stable while zooming
- fitting an image inside the available view
- resetting zoom and pan state
- displaying the current zoom percentage
- limiting the minimum and maximum zoom levels

---

## 1. Why Zoom and Pan Matter in a DICOM Viewer

Medical images can be much larger than the visible area of a desktop application. If an image is always reduced to fit the window, small structures may become difficult to examine.

Zoom and pan solve two related navigation problems:

- **Zoom** changes the visual scale of the image.
- **Pan** moves the enlarged image within the viewport.

These operations should affect only the view transformation. They should not resize, resample, or overwrite the original DICOM pixel array.

This distinction is important:

```text
Original DICOM pixels
        ↓
Windowed display image
        ↓
QPixmap in the graphics scene
        ↓
View transformation for zoom and pan
```

The pixel data remains unchanged while `QGraphicsView` controls how the image is displayed.

---

## 2. Why Use QGraphicsView?

PyQt5 provides several ways to display an image. A `QLabel` is convenient for a simple preview, but `QGraphicsView` is better suited to interactive navigation.

This implementation uses three graphics components:

| Component | Responsibility |
|---|---|
| `QGraphicsScene` | Stores and manages graphics items |
| `QGraphicsPixmapItem` | Holds the image displayed in the scene |
| `QGraphicsView` | Displays the scene and applies transformations |

The scene stores the pixmap item, while the view handles scaling, scrolling, alignment, and mouse interaction.

---

## 3. Creating a Reusable ImageView Class

The zoom and pan behavior is separated into a custom `ImageView` class.

```python
from PyQt5.QtCore import Qt, pyqtSignal
from PyQt5.QtGui import QPainter
from PyQt5.QtWidgets import QGraphicsPixmapItem, QGraphicsScene, QGraphicsView


class ImageView(QGraphicsView):
    """A graphics view responsible for zooming and panning a DICOM image."""

    zoom_changed = pyqtSignal(int)

    def __init__(self, parent=None):
        super().__init__(parent)

        self.scene = QGraphicsScene(self)
        self.setScene(self.scene)

        self.pixmap_item = QGraphicsPixmapItem()
        self.scene.addItem(self.pixmap_item)
```

The `QGraphicsScene` belongs to the view. A single `QGraphicsPixmapItem` is added to it and reused whenever a new DICOM image is loaded.

The custom signal sends the current zoom percentage back to the main window:

```python
zoom_changed = pyqtSignal(int)
```

This keeps the navigation logic inside `ImageView` while allowing the main interface to update its label.

---

## 4. Defining the Zoom Settings

The class stores four values that control zoom behavior:

```python
self.zoom_factor = 1.0
self.zoom_step = 1.25
self.min_zoom = 0.1
self.max_zoom = 10.0
```

Their meanings are:

| Variable | Value | Purpose |
|---|---:|---|
| `zoom_factor` | `1.0` | Tracks the current view scale |
| `zoom_step` | `1.25` | Changes the scale by 25% per wheel step |
| `min_zoom` | `0.1` | Prevents zooming below 10% |
| `max_zoom` | `10.0` | Prevents zooming above 1000% |

When the user zooms in, the current scale is multiplied by `1.25`. When the user zooms out, it is divided by `1.25`.

For example:

```text
Zoom in:  1.00 × 1.25 = 1.25
Zoom out: 1.00 ÷ 1.25 = 0.80
```

The minimum and maximum limits prevent the view from becoming impractically small or excessively large.

---

## 5. Configuring Pan and Transformation Behavior

The following settings enable interactive navigation:

```python
self.setDragMode(QGraphicsView.ScrollHandDrag)
self.setTransformationAnchor(QGraphicsView.AnchorUnderMouse)
self.setResizeAnchor(QGraphicsView.AnchorViewCenter)
self.setRenderHint(QPainter.SmoothPixmapTransform)

self.setAlignment(Qt.AlignCenter)
self.setBackgroundBrush(Qt.black)
```

### ScrollHandDrag

```python
self.setDragMode(QGraphicsView.ScrollHandDrag)
```

`ScrollHandDrag` provides panning without requiring custom mouse press, move, and release handlers. When the image is larger than the viewport, the user can drag it with the left mouse button.

### AnchorUnderMouse

```python
self.setTransformationAnchor(QGraphicsView.AnchorUnderMouse)
```

This setting makes zooming feel natural. The position below the mouse pointer remains the visual focus while the view is scaled.

Without it, the image may zoom around the view center instead of the area the user wants to inspect.

### AnchorViewCenter

```python
self.setResizeAnchor(QGraphicsView.AnchorViewCenter)
```

When the widget is resized, the center of the view remains the anchor point.

### SmoothPixmapTransform

```python
self.setRenderHint(QPainter.SmoothPixmapTransform)
```

This render hint improves the visual appearance of a scaled pixmap. It affects only presentation and does not modify the original DICOM pixel array.

The black background is also appropriate for many medical-image viewing environments because it reduces visual distraction around a grayscale image.

---

## 6. Loading a Pixmap into the Scene

The `set_pixmap()` method updates the existing graphics item:

```python
def set_pixmap(self, pixmap):
    """Set the image to display and fit it within the current view."""
    self.pixmap_item.setPixmap(pixmap)
    self.scene.setSceneRect(self.pixmap_item.boundingRect())
    self.fit_to_view()
```

This method performs three operations:

1. Assigns the new pixmap to the graphics item.
2. Updates the scene rectangle to match the image bounds.
3. Fits the image inside the current viewport.

Updating the scene rectangle is important because it defines the navigable area used by the graphics view and its scroll bars.

---

## 7. Implementing Mouse-Wheel Zoom

The mouse-wheel behavior is implemented by overriding `wheelEvent()`:

```python
def wheelEvent(self, event):
    """Zoom the image in or out using the mouse wheel."""
    if self.pixmap_item.pixmap().isNull():
        return

    if event.angleDelta().y() > 0:
        new_zoom = self.zoom_factor * self.zoom_step
        scale_factor = self.zoom_step
    else:
        new_zoom = self.zoom_factor / self.zoom_step
        scale_factor = 1 / self.zoom_step

    if self.min_zoom <= new_zoom <= self.max_zoom:
        self.scale(scale_factor, scale_factor)
        self.zoom_factor = new_zoom
        self.zoom_changed.emit(self.get_zoom_percentage())
```

### Checking Whether an Image Exists

```python
if self.pixmap_item.pixmap().isNull():
    return
```

There is nothing to scale before an image is loaded, so the method returns immediately.

### Detecting the Wheel Direction

```python
event.angleDelta().y()
```

A positive vertical delta means the wheel moved upward, and a negative value means it moved downward.

```python
if event.angleDelta().y() > 0:
    # Zoom in
else:
    # Zoom out
```

### Calculating the Next Zoom Level

The code calculates `new_zoom` before changing the view. This makes it possible to validate the requested scale against the allowed range.

```python
if self.min_zoom <= new_zoom <= self.max_zoom:
```

Only a valid zoom operation is applied:

```python
self.scale(scale_factor, scale_factor)
```

The same factor is used for the horizontal and vertical axes, preserving the image aspect ratio.

After scaling, the class updates its stored state and emits the new percentage:

```python
self.zoom_factor = new_zoom
self.zoom_changed.emit(self.get_zoom_percentage())
```

---

## 8. Fitting the Image to the View

When a new image is loaded or the user resets the view, the image should fit inside the available display area.

```python
def fit_to_view(self):
    """Fit the image within the current view while preserving its aspect ratio."""
    if self.pixmap_item.pixmap().isNull():
        return

    self.resetTransform()
    self.fitInView(self.pixmap_item, Qt.KeepAspectRatio)
    self.zoom_factor = self.transform().m11()
    self.zoom_changed.emit(self.get_zoom_percentage())
```

First, `resetTransform()` removes any previous scaling transformation:

```python
self.resetTransform()
```

Then, `fitInView()` scales the item to fit while preserving its width-to-height ratio:

```python
self.fitInView(self.pixmap_item, Qt.KeepAspectRatio)
```

The resulting scale is read from the transformation matrix:

```python
self.zoom_factor = self.transform().m11()
```

The `m11()` value represents the horizontal scale component. Because the image is scaled uniformly, it also represents the overall zoom level.

This means the fitted image is not always exactly 100%. Its displayed percentage depends on the image dimensions and viewport size.

---

## 9. Resetting Zoom and Pan

The reset operation delegates to `fit_to_view()`:

```python
def reset_view(self):
    """Reset the zoom and pan state."""
    self.fit_to_view()
```

This restores a predictable state by:

- clearing the previous transformation
- fitting the complete image in the viewport
- returning the image to the centered position
- recalculating the displayed zoom percentage

Resetting to fit-to-view is generally more useful than forcing the scale to 100%, especially when the image is larger than the application window.

---

## 10. Converting the Scale to a Percentage

The zoom label needs an integer percentage rather than a floating-point scale:

```python
def get_zoom_percentage(self):
    """Return the current zoom level as a percentage."""
    return round(self.zoom_factor * 100)
```

Examples include:

```text
0.50 → 50%
1.00 → 100%
1.25 → 125%
2.00 → 200%
```

Using `round()` keeps the label simple and readable.

---

## 11. Connecting ImageView to the Main Window

The main window creates the custom view and gives it a minimum display size:

```python
# Image display area
self.image_view = ImageView()
self.image_view.setMinimumSize(512, 512)
```

A button resets the view:

```python
# Reset View button
self.reset_view_button = QPushButton("Reset View")
self.reset_view_button.clicked.connect(
    self.image_view.reset_view
)
```

The interface also creates a zoom label and connects the custom signal:

```python
# Zoom percentage display
self.zoom_label = QLabel("Zoom: 100%")

self.image_view.zoom_changed.connect(
    self.update_zoom_label
)
```

The label is added to the control layout:

```python
control_layout.addWidget(self.reset_view_button)
control_layout.addWidget(self.zoom_label)
```

Finally, the slot updates the displayed text:

```python
def update_zoom_label(self, percentage):
    """Display the current zoom percentage."""
    self.zoom_label.setText(f"Zoom: {percentage}%")
```

This is a clean use of Qt's signal-and-slot mechanism:

```text
Mouse wheel
    ↓
ImageView changes its transformation
    ↓
zoom_changed emits an integer
    ↓
DicomViewer updates the label
```

---

## 12. Complete image_view.py

The complete Day 6 implementation is:

```python
from PyQt5.QtCore import Qt, pyqtSignal
from PyQt5.QtGui import QPainter
from PyQt5.QtWidgets import QGraphicsPixmapItem, QGraphicsScene, QGraphicsView


class ImageView(QGraphicsView):
    """A graphics view responsible for zooming and panning a DICOM image."""

    zoom_changed = pyqtSignal(int)

    def __init__(self, parent=None):
        super().__init__(parent)

        self.scene = QGraphicsScene(self)
        self.setScene(self.scene)

        self.pixmap_item = QGraphicsPixmapItem()
        self.scene.addItem(self.pixmap_item)

        self.zoom_factor = 1.0
        self.zoom_step = 1.25
        self.min_zoom = 0.1
        self.max_zoom = 10.0

        self.setDragMode(QGraphicsView.ScrollHandDrag)
        self.setTransformationAnchor(QGraphicsView.AnchorUnderMouse)
        self.setResizeAnchor(QGraphicsView.AnchorViewCenter)
        self.setRenderHint(QPainter.SmoothPixmapTransform)

        self.setAlignment(Qt.AlignCenter)
        self.setBackgroundBrush(Qt.black)

    def set_pixmap(self, pixmap):
        """Set the image to display and fit it within the current view."""
        self.pixmap_item.setPixmap(pixmap)
        self.scene.setSceneRect(self.pixmap_item.boundingRect())
        self.fit_to_view()

    def wheelEvent(self, event):
        """Zoom the image in or out using the mouse wheel."""
        if self.pixmap_item.pixmap().isNull():
            return

        if event.angleDelta().y() > 0:
            new_zoom = self.zoom_factor * self.zoom_step
            scale_factor = self.zoom_step
        else:
            new_zoom = self.zoom_factor / self.zoom_step
            scale_factor = 1 / self.zoom_step

        if self.min_zoom <= new_zoom <= self.max_zoom:
            self.scale(scale_factor, scale_factor)
            self.zoom_factor = new_zoom
            self.zoom_changed.emit(self.get_zoom_percentage())

    def fit_to_view(self):
        """Fit the image within the current view while preserving its aspect ratio."""
        if self.pixmap_item.pixmap().isNull():
            return

        self.resetTransform()
        self.fitInView(self.pixmap_item, Qt.KeepAspectRatio)
        self.zoom_factor = self.transform().m11()
        self.zoom_changed.emit(self.get_zoom_percentage())

    def reset_view(self):
        """Reset the zoom and pan state."""
        self.fit_to_view()

    def get_zoom_percentage(self):
        """Return the current zoom level as a percentage."""
        return round(self.zoom_factor * 100)
```

---

## 13. Testing Zoom and Pan

Run the application and open a DICOM file.

```bash
python src/main.py
```

Test the following operations:

1. Move the mouse pointer over a specific part of the image.
2. Scroll upward to zoom in.
3. Confirm that the area under the pointer remains the focus.
4. Scroll downward to zoom out.
5. Zoom in until the image becomes larger than the viewport.
6. Drag the image with the left mouse button.
7. Confirm that the zoom label changes.
8. Click **Reset View**.
9. Confirm that the complete image fits inside the view again.

Expected behavior:

```text
Mouse wheel up     → Zoom in
Mouse wheel down   → Zoom out
Left mouse drag    → Pan the enlarged image
Reset View         → Fit and center the complete image
Zoom label         → Display the current scale percentage
```

The zoom level should remain between 10% and 1000%.

---

## 14. Common Problems

### The image zooms before a file is loaded

Check for a null pixmap at the start of `wheelEvent()`.

### The image becomes distorted

Use the same scale factor for both axes:

```python
self.scale(scale_factor, scale_factor)
```

### Zooming focuses on the wrong area

Use:

```python
self.setTransformationAnchor(QGraphicsView.AnchorUnderMouse)
```

### The image cannot be panned

Panning becomes useful when the scaled image is larger than the viewport. Confirm that `ScrollHandDrag` is enabled and zoom in before testing.

### Reset does not restore a clean state

Call `resetTransform()` before `fitInView()`.

### The zoom label does not update

Confirm that `zoom_changed` is emitted after both manual scaling and fit-to-view, and that the signal is connected to `update_zoom_label()`.

---

## 15. Possible Improvements

The navigation system can later be expanded with:

- keyboard shortcuts for zooming
- double-click reset
- a dedicated 100% zoom button
- zoom presets such as 50%, 100%, and 200%
- touchpad and pinch-gesture support
- a miniature overview navigator
- synchronized navigation across multiple images
- preserving zoom state while changing Window/Level
- mapping mouse coordinates back to DICOM pixel coordinates
- interpolation options for diagnostic and non-diagnostic viewing

Future measurement tools must also convert view coordinates into scene and image coordinates correctly. Otherwise, zooming and panning can produce incorrect pixel selection or distance measurements.

---

## 16. Medical-Imaging Considerations

Zooming a displayed image does not create additional diagnostic information. It enlarges existing pixels but cannot recover details that were not present in the original acquisition.

There is also an important difference between display scaling and pixel resampling:

- View scaling changes how the existing image is presented.
- Pixel resampling creates a new pixel array.

This lesson uses view scaling, leaving the source DICOM pixel data unchanged.

For clinical software, display behavior may also be affected by monitor calibration, interpolation policy, grayscale presentation requirements, modality characteristics, and regulatory expectations. This educational viewer is not a certified diagnostic application.

---

## What I Learned

In this lesson, I learned that:

- `QGraphicsScene` manages the image item.
- `QGraphicsPixmapItem` stores the displayed pixmap.
- `QGraphicsView` applies zoom and pan transformations.
- `ScrollHandDrag` provides built-in left-button panning.
- `AnchorUnderMouse` keeps the pointer area stable while zooming.
- `wheelEvent()` can distinguish zoom-in and zoom-out directions.
- a fixed zoom step produces predictable scaling.
- minimum and maximum limits keep zoom within a practical range.
- `fitInView()` fits the complete image while preserving its aspect ratio.
- `resetTransform()` removes earlier view transformations.
- `transform().m11()` provides the current horizontal scale.
- a custom Qt signal can report the zoom percentage to the main window.
- view transformations do not modify the original DICOM pixel array.
- coordinate mapping will become important for later inspection and measurement tools.

The most important idea from this lesson is:

> **Zoom and pan should change how a DICOM image is viewed without changing the original medical-image pixel data.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 6 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/8b9456c)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I converted the currently windowed DICOM image to 8-bit grayscale and exported it as a PNG file:

[Exporting Windowed DICOM Images to PNG with Python](/medical-imaging/dicom/2026/08/29/exporting-windowed-dicom-images-to-png-with-python.html)

---

## Next Step

Now that the viewer supports zoom and pan, the next step is to control how DICOM pixel values are mapped to visible grayscale values.

In the next lesson, we will explore:

- Window Center and Window Width
- grayscale intensity mapping
- interactive Window/Level controls
- safe handling of narrow window widths
- updating the displayed image without changing the source pixels

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
