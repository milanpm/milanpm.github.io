---
layout: post
title: "Measuring ROI Statistics in DICOM Images with Python and PyQt5"
date: 2026-09-01 22:30:00 +0900
categories: [medical-imaging]
tags: [dicom, pydicom, python, pyqt5, numpy, roi, hounsfield-units, medical-imaging]
description: "Learn how to select a rectangular ROI in a DICOM image and calculate its mean, minimum, and maximum Hounsfield Unit values using Python, NumPy, and PyQt5."
---

A single pixel value can tell us about one location in a DICOM image, but many image-analysis tasks require information from an entire region.

A **Region of Interest (ROI)** is a selected area of an image used for measurement or analysis. In CT imaging, ROI statistics can summarize the Hounsfield Unit values inside a tissue region.

In this post, we will extend our Python DICOM viewer with a rectangular ROI tool that:

- Selects an ROI with `Ctrl + Left Drag`
- Draws a green rectangle over the image
- Converts view coordinates to image coordinates
- Extracts the selected region with NumPy slicing
- Displays the ROI width and height
- Calculates the mean, minimum, and maximum HU values
- Clears the ROI and its results safely

---

## What Is a Region of Interest?

An ROI is a limited part of an image chosen for closer inspection.

Instead of calculating statistics for the entire DICOM image, we select a rectangular region bounded by two points:

```text
Start point: (x1, y1)
End point:   (x2, y2)
```

The viewer extracts all pixels inside this rectangle and calculates descriptive statistics.

For an ROI containing values \(v_1, v_2, \ldots, v_N\), the mean is:

\[
\text{Mean} = \frac{1}{N}\sum_{i=1}^{N}v_i
\]

The minimum and maximum describe the lowest and highest HU values within the selected area.

---

## Why ROI Statistics Are Useful

ROI measurements are commonly used to:

- Compare image regions
- Estimate tissue attenuation in CT images
- Inspect uniformity and noise
- Identify unusually low or high values
- Support segmentation and quantitative image analysis
- Validate image-processing results

A mean value provides a summary of the selected region, while the minimum and maximum help reveal its range.

However, these statistics must always be interpreted together with acquisition conditions, image reconstruction, ROI placement, and DICOM calibration metadata.

---

## Converting Stored Pixels to Hounsfield Units

The stored DICOM pixel value is not always the final physical value used for CT interpretation.

The conversion to Hounsfield Units is:

\[
HU = \text{Stored Pixel Value} \times \text{Rescale Slope}
+ \text{Rescale Intercept}
\]

Our DICOM loader performs this conversion when the image is opened:

```python
pixel_array = dataset.pixel_array.astype(np.float32)

slope = float(getattr(dataset, "RescaleSlope", 1.0))
intercept = float(getattr(dataset, "RescaleIntercept", 0.0))

pixel_array = pixel_array * slope + intercept
```

The resulting `self.pixel_array` therefore contains the rescaled values used by the ROI calculation.

When the DICOM file does not contain these attributes, the defaults are:

```text
Rescale Slope     = 1.0
Rescale Intercept = 0.0
```

---

## Adding an ROI Signal

The `ImageView` class emits four integer coordinates after the selection is complete:

```python
roi_selected = pyqtSignal(int, int, int, int)
```

The main window connects the signal to the ROI calculation method:

```python
self.image_view.roi_selected.connect(
    self.update_roi_measurement
)
```

This design separates mouse interaction from DICOM data analysis:

- `ImageView` manages selection and drawing.
- The main window extracts the HU array and calculates statistics.

---

## Starting ROI Selection

The user begins an ROI selection by holding `Ctrl` and pressing the left mouse button:

```python
if (
    event.button() == Qt.LeftButton
    and event.modifiers() & Qt.ControlModifier
    and not self.pixmap_item.pixmap().isNull()
):
    image_point = self.get_image_point(event.pos())

    if image_point is not None:
        self.clear_roi()
        self.selecting_roi = True
        self.roi_start = image_point
        self.setDragMode(QGraphicsView.NoDrag)
        self.setCursor(Qt.CrossCursor)
        self.show_roi(image_point, image_point)

    event.accept()
    return
```

This code:

1. Confirms that a DICOM image is displayed.
2. Converts the mouse position to image coordinates.
3. Removes the previous ROI.
4. Stores the starting point.
5. Temporarily disables image panning.
6. Changes the cursor to a crosshair.
7. Draws the initial ROI rectangle.

---

## Converting Mouse Coordinates to Image Coordinates

The mouse position belongs to the `QGraphicsView`, but the ROI must match the original image coordinates.

The conversion follows this path:

```text
View coordinates
    ↓ mapToScene()
Scene coordinates
    ↓ mapFromScene()
Image coordinates
```

The helper method also supports coordinate clamping:

```python
def get_image_point(self, view_position, clamp=False):
    scene_pos = self.mapToScene(view_position)
    image_pos = self.pixmap_item.mapFromScene(scene_pos)

    x = int(image_pos.x())
    y = int(image_pos.y())

    if clamp:
        x = max(0, min(x, pixmap.width() - 1))
        y = max(0, min(y, pixmap.height() - 1))
        return x, y

    if 0 <= x < pixmap.width() and 0 <= y < pixmap.height():
        return x, y

    return None
```

Clamping keeps the coordinates inside the image if the pointer moves beyond an edge during the drag operation.

---

## Updating the Rectangle While Dragging

While the left mouse button is held, the current pointer position updates the visible rectangle:

```python
if self.selecting_roi and self.roi_start is not None:
    image_point = self.get_image_point(event.pos(), clamp=True)

    if image_point is not None:
        self.show_roi(self.roi_start, image_point)

    event.accept()
    return
```

The start point remains fixed while the end point follows the pointer.

This gives immediate visual feedback before the ROI statistics are calculated.

---

## Drawing the ROI Overlay

The two drag points may be selected in any direction. The coordinates are therefore normalized with `min()` and `max()`:

```python
def show_roi(self, start, end):
    """Display the rectangular ROI created by two image coordinates."""
    x1 = min(start[0], end[0])
    y1 = min(start[1], end[1])
    x2 = max(start[0], end[0])
    y2 = max(start[1], end[1])

    if self.roi_rectangle is None:
        pen = QPen(Qt.green)
        pen.setWidth(2)
        self.roi_rectangle = QGraphicsRectItem()
        self.roi_rectangle.setPen(pen)
        self.scene.addItem(self.roi_rectangle)

    self.roi_rectangle.setRect(
        QRectF(x1, y1, x2 - x1 + 1, y2 - y1 + 1)
    )
```

The green `QGraphicsRectItem` is added to the same scene as the DICOM image. It therefore stays aligned with the selected pixels while zooming or panning.

The `+1` includes both boundary pixels in the displayed ROI dimensions.

---

## Completing the Selection

When the user releases the left mouse button, the viewer completes the ROI:

```python
if event.button() == Qt.LeftButton and self.selecting_roi:
    image_point = self.get_image_point(event.pos(), clamp=True)
    start = self.roi_start

    self.selecting_roi = False
    self.roi_start = None
    self.setDragMode(QGraphicsView.ScrollHandDrag)
    self.unsetCursor()

    if start is not None and image_point is not None:
        self.show_roi(start, image_point)

        x1 = min(start[0], image_point[0])
        y1 = min(start[1], image_point[1])
        x2 = max(start[0], image_point[0])
        y2 = max(start[1], image_point[1])

        self.roi_selected.emit(x1, y1, x2, y2)

    event.accept()
    return
```

Normalizing the coordinates means the user can drag in any direction:

- Top-left to bottom-right
- Bottom-right to top-left
- Top-right to bottom-left
- Bottom-left to top-right

---

## Extracting the ROI with NumPy

The main window first limits all coordinates to the valid array boundaries:

```python
height, width = self.pixel_array.shape[:2]

x1 = max(0, min(x1, width - 1))
x2 = max(0, min(x2, width - 1))
y1 = max(0, min(y1, height - 1))
y2 = max(0, min(y2, height - 1))
```

The ROI is then extracted using NumPy slicing:

```python
roi = self.pixel_array[y1:y2 + 1, x1:x2 + 1]
```

NumPy arrays use row-major indexing:

```python
array[y, x]
```

This is why Y coordinates appear before X coordinates in the slice.

---

## Why the Slice Uses `+1`

The end index of a Python slice is excluded.

For example:

```python
values[10:20]
```

contains indices `10` through `19`, but not `20`.

Because the ROI selection treats both endpoints as part of the rectangle, the code uses:

```python
y1:y2 + 1
x1:x2 + 1
```

If the selected X coordinates are `100` and `199`, the ROI width becomes:

```text
199 - 100 + 1 = 100 pixels
```

---

## Calculating ROI Statistics

The complete calculation method is:

```python
def update_roi_measurement(self, x1, y1, x2, y2):
    """Display the Mean/Min/Max HU values of a rectangular ROI."""
    if self.pixel_array is None:
        return

    height, width = self.pixel_array.shape[:2]
    x1 = max(0, min(x1, width - 1))
    x2 = max(0, min(x2, width - 1))
    y1 = max(0, min(y1, height - 1))
    y2 = max(0, min(y2, height - 1))

    roi = self.pixel_array[y1:y2 + 1, x1:x2 + 1]

    if roi.size == 0:
        self.roi_label.setText("ROI: -")
        return

    self.roi_label.setText(
        f"ROI: {roi.shape[1]} x {roi.shape[0]} px\n"
        f"Mean: {np.mean(roi):.2f} HU\n"
        f"Min: {np.min(roi):.2f} HU\n"
        f"Max: {np.max(roi):.2f} HU"
    )
```

The statistics are calculated with:

```python
np.mean(roi)
np.min(roi)
np.max(roi)
```

The displayed ROI size uses:

```python
roi.shape[1]  # width
roi.shape[0]  # height
```

---

## Handling an Empty ROI

Although the coordinates are normalized and clamped, the method still checks the extracted array:

```python
if roi.size == 0:
    self.roi_label.setText("ROI: -")
    return
```

This defensive check prevents NumPy reduction functions from operating on an empty array.

---

## Clearing the ROI

The ROI overlay is removed from the graphics scene with:

```python
def clear_roi(self):
    """Remove the current ROI rectangle overlay."""
    if self.roi_rectangle is not None:
        self.scene.removeItem(self.roi_rectangle)
        self.roi_rectangle = None
```

The main window also resets the result label:

```python
def clear_roi_measurement(self):
    """Clear the ROI rectangle and measurement result."""
    self.image_view.clear_roi()
    self.roi_label.setText("ROI: Ctrl + Left Drag")
```

This method is connected to the `Clear ROI` button.

---

## Using the ROI Tool

1. Open a DICOM image.
2. Hold the `Ctrl` key.
3. Press and hold the left mouse button at the ROI start point.
4. Drag across the region to be measured.
5. Release the mouse button.
6. Read the ROI size and HU statistics in the Viewer panel.
7. Select `Clear ROI` to remove the result.

The Viewer displays a result similar to:

```text
ROI: 120 x 85 px
Mean: 42.31 HU
Min: -18.00 HU
Max: 96.00 HU
```

---

## Result

After an ROI is selected, the viewer:

- Draws a green rectangular overlay
- Keeps the rectangle inside the image boundaries
- Extracts the corresponding HU array
- Displays the width and height in pixels
- Calculates the mean HU value
- Displays the minimum and maximum HU values

![DICOM rectangular ROI statistics result](/assets/images/posts/post-10-dicom-roi/dicom-roi-statistics.png)

---

## Important Considerations

### ROI Placement

The statistics depend directly on the selected location and size. Including surrounding structures, image edges, or artifacts can significantly change the result.

### CT and Other Modalities

Hounsfield Units are specifically associated with calibrated CT image values. For another modality, the rescaled values may have a different physical or modality-specific meaning.

### Multi-frame and Color Images

This implementation assumes a two-dimensional grayscale pixel array. Multi-frame, RGB, palette-color, and other image formats require additional dimension and color handling.

### Diagnostic Use

This project demonstrates DICOM image-processing concepts. It is not a validated diagnostic medical device. Clinical measurements require appropriate testing, calibration, quality assurance, risk management, and regulatory compliance.

---

## What We Learned

In this post, we learned how to:

- Select a rectangular region with PyQt5 mouse events
- Convert view coordinates to image coordinates
- Clamp a selection to image boundaries
- Normalize coordinates for every drag direction
- Draw a `QGraphicsRectItem` ROI overlay
- Extract a two-dimensional region with NumPy slicing
- Include both ROI boundary pixels correctly
- Calculate mean, minimum, and maximum HU values
- Clear the visual overlay and calculated results

The ROI tool is an important step from simple image display toward quantitative DICOM image analysis.

---

## Source Code

The complete source code is available on GitHub:

[View the PACS-DICOM-Toolkit repository on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

In the next post, we will refactor the DICOM viewer into separate modules for loading, windowing, anonymization, and image interaction.

This will make the project easier to maintain, test, and extend with additional DICOM networking features.

**Next: Refactoring a Python DICOM Viewer into Modular Components**
