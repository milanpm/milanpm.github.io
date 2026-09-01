---
layout: post
title: "Measuring Distance in DICOM Images Using Pixel Spacing"
date: 2026-09-01 21:30:00 +0900
categories: [medical-imaging]
tags: [dicom, pydicom, python, pyqt5, pixel-spacing, medical-imaging]
description: "Learn how to measure physical distances in DICOM images using Pixel Spacing, NumPy, pydicom, and PyQt5."
---

Medical images often contain measurements that must be expressed in physical units such as millimeters rather than pixels.

A line that is 100 pixels long does not necessarily represent 100 millimeters. The physical distance depends on the spatial calibration information stored in the DICOM metadata.

In this post, we will extend our Python DICOM viewer with a distance measurement tool that:

* Selects two points using `Shift + Left Click`
* Draws a measurement line between the points
* Reads the DICOM `PixelSpacing` attribute
* Calculates the physical distance in millimeters
* Falls back to pixel distance when spacing information is unavailable

---

## Why Pixel Distance Is Not Enough

A digital image is made of pixels. If two points are located at:

```text
Point 1: (x1, y1)
Point 2: (x2, y2)
```

their distance in pixel coordinates can be calculated using the Euclidean distance formula:

$$
d_{\text{pixel}}
=
\sqrt{
(x_2-x_1)^2+(y_2-y_1)^2
}
$$

This result only describes how far apart the points are in the pixel grid.

It does not describe their actual physical distance.

For example, a 100-pixel line may represent:

* 50 mm when each pixel represents 0.5 mm
* 80 mm when each pixel represents 0.8 mm
* 100 mm when each pixel represents 1.0 mm

To obtain a physical measurement, the viewer must use the pixel spacing stored in the DICOM dataset.

---

## Understanding DICOM Pixel Spacing

DICOM images may contain the `PixelSpacing` attribute:

```text
(0028,0030) Pixel Spacing
```

It normally contains two values:

```text
PixelSpacing = [row spacing, column spacing]
```

The values represent the physical distance between adjacent pixel centers in millimeters.

For example:

```text
PixelSpacing = [0.703125, 0.703125]
```

means:

* Vertical spacing: `0.703125 mm` per pixel
* Horizontal spacing: `0.703125 mm` per pixel

An important detail is the ordering:

```python
row_spacing = float(pixel_spacing[0])
column_spacing = float(pixel_spacing[1])
```

The row spacing is applied to the change in the Y direction, while the column spacing is applied to the change in the X direction.

---

## Physical Distance Formula

First, calculate the difference between the selected coordinates:

```python
dx = x2 - x1
dy = y2 - y1
```

Convert each difference to millimeters:

```python
physical_dx = dx * column_spacing
physical_dy = dy * row_spacing
```

The final distance is:

$$
d_{\text{mm}}
=
\sqrt{
(\Delta x \times \text{column spacing})^2
+
(\Delta y \times \text{row spacing})^2
}
$$

In Python:

```python
distance_mm = np.sqrt(
    (dx * column_spacing) ** 2
    + (dy * row_spacing) ** 2
)
```

Applying the row and column spacing independently is important because the pixels may not always be square.

---

## Adding a Measurement Signal

The `ImageView` class handles mouse interaction with the displayed image.

A custom PyQt signal sends the selected image coordinates to the main window:

```python
measurement_point_selected = pyqtSignal(int, int)
```

The signal is connected to the distance measurement method:

```python
self.image_view.measurement_point_selected.connect(
    self.add_measurement_point
)
```

This separates the responsibilities of the two classes:

* `ImageView` detects mouse input and displays graphics.
* The main window reads the DICOM dataset and calculates distance.

---

## Selecting Points with Shift + Left Click

Distance measurement begins when the user holds `Shift` and clicks the left mouse button:

```python
if (
    event.button() == Qt.LeftButton
    and event.modifiers() & Qt.ShiftModifier
    and not self.pixmap_item.pixmap().isNull()
):
    scene_pos = self.mapToScene(event.pos())
    image_pos = self.pixmap_item.mapFromScene(scene_pos)

    x = int(image_pos.x())
    y = int(image_pos.y())

    pixmap = self.pixmap_item.pixmap()

    if 0 <= x < pixmap.width() and 0 <= y < pixmap.height():
        self.measurement_point_selected.emit(x, y)

    event.accept()
    return
```

The mouse position initially belongs to the `QGraphicsView`.

It must be converted through the following coordinate systems:

```text
View coordinates
    ↓ mapToScene()
Scene coordinates
    ↓ mapFromScene()
Image coordinates
```

This conversion keeps the selected point aligned with the original DICOM image even when the user zooms or pans the view.

The boundary check prevents points outside the image from being selected:

```python
if 0 <= x < pixmap.width() and 0 <= y < pixmap.height():
```

---

## Storing the Measurement Points

The main window stores selected points in a list:

```python
self.measurement_points = []
```

The first selected point is added to the list and shown in the interface:

```python
self.measurement_points.append((x, y))

if len(self.measurement_points) == 1:
    self.distance_label.setText(
        f"Point 1: ({x}, {y})"
    )
    return
```

After the second point is selected, the viewer has enough information to calculate and display the distance.

If two points already exist, the next click starts a new measurement:

```python
if len(self.measurement_points) == 2:
    self.measurement_points.clear()
    self.image_view.clear_measurement()
```

This produces a simple measurement cycle:

```text
First click  → Store Point 1
Second click → Calculate and display distance
Third click  → Clear the old result and start again
```

---

## Calculating the Distance

The complete distance calculation method is:

```python
def add_measurement_point(self, x, y):
    """Display the distance between two points in pixels or millimeters."""
    if self.dataset is None:
        return

    if len(self.measurement_points) == 2:
        self.measurement_points.clear()
        self.image_view.clear_measurement()

    self.measurement_points.append((x, y))

    if len(self.measurement_points) == 1:
        self.distance_label.setText(
            f"Point 1: ({x}, {y})"
        )
        return

    (x1, y1), (x2, y2) = self.measurement_points

    self.image_view.show_measurement(
        (x1, y1),
        (x2, y2),
    )

    dx = x2 - x1
    dy = y2 - y1

    pixel_spacing = getattr(
        self.dataset,
        "PixelSpacing",
        None,
    )

    if pixel_spacing is None or len(pixel_spacing) < 2:
        distance = np.sqrt(dx ** 2 + dy ** 2)
        self.distance_label.setText(
            f"Distance: {distance:.2f} px"
        )
        return

    row_spacing = float(pixel_spacing[0])
    column_spacing = float(pixel_spacing[1])

    distance_mm = np.sqrt(
        (dx * column_spacing) ** 2
        + (dy * row_spacing) ** 2
    )

    self.distance_label.setText(
        f"Distance: {distance_mm:.2f} mm"
    )
```

The method first attempts to read `PixelSpacing` safely:

```python
pixel_spacing = getattr(
    self.dataset,
    "PixelSpacing",
    None,
)
```

Using `getattr()` with a default value avoids an exception when the attribute is missing.

---

## Falling Back to Pixel Distance

Not every DICOM image contains reliable spatial calibration information.

When `PixelSpacing` is unavailable or incomplete, the viewer calculates the distance in pixels:

```python
if pixel_spacing is None or len(pixel_spacing) < 2:
    distance = np.sqrt(dx ** 2 + dy ** 2)

    self.distance_label.setText(
        f"Distance: {distance:.2f} px"
    )
    return
```

The unit is explicitly displayed as `px`.

This distinction is essential. A pixel distance must not be presented as a physical measurement because there is no metadata available to support conversion to millimeters.

---

## Drawing the Measurement Line

The viewer draws a yellow line between the two selected points:

```python
def show_measurement(self, start, end):
    """Display a measurement line between two image coordinates."""
    self.clear_measurement()

    x1, y1 = start
    x2, y2 = end

    pen = QPen(Qt.yellow)
    pen.setWidth(2)

    self.measurement_line = QGraphicsLineItem(
        x1,
        y1,
        x2,
        y2,
    )
    self.measurement_line.setPen(pen)
    self.scene.addItem(self.measurement_line)
```

Two circular markers identify the start and end points:

```python
marker_size = 6
radius = marker_size / 2

self.measurement_start_marker = QGraphicsEllipseItem(
    x1 - radius,
    y1 - radius,
    marker_size,
    marker_size,
)
self.measurement_start_marker.setPen(pen)
self.scene.addItem(self.measurement_start_marker)

self.measurement_end_marker = QGraphicsEllipseItem(
    x2 - radius,
    y2 - radius,
    marker_size,
    marker_size,
)
self.measurement_end_marker.setPen(pen)
self.scene.addItem(self.measurement_end_marker)
```

Because these graphics are added to the same `QGraphicsScene` as the image, they remain aligned with the selected locations during zooming and panning.

---

## Clearing a Previous Measurement

Before drawing a new line, the previous graphics must be removed from the scene:

```python
def clear_measurement(self):
    """Remove the current distance measurement graphics."""
    for item in (
        self.measurement_line,
        self.measurement_start_marker,
        self.measurement_end_marker,
    ):
        if item is not None:
            self.scene.removeItem(item)

    self.measurement_line = None
    self.measurement_start_marker = None
    self.measurement_end_marker = None
```

The main window also resets the selected points and label:

```python
def clear_measurements(self):
    """Clear distance and ROI measurement state."""
    self.measurement_points.clear()
    self.image_view.clear_measurement()
    self.distance_label.setText("Distance: -")
    self.clear_roi_measurement()
```

Resetting both the logical state and the visible graphics prevents an old measurement from remaining on the screen.

---

## Example Calculation

Suppose the user selects these points:

```text
Point 1: (100, 150)
Point 2: (300, 350)
```

The coordinate differences are:

```text
dx = 300 - 100 = 200 pixels
dy = 350 - 150 = 200 pixels
```

Assume the DICOM metadata contains:

```text
PixelSpacing = [0.5, 0.8]
```

Then:

```text
Horizontal distance = 200 × 0.8 = 160 mm
Vertical distance   = 200 × 0.5 = 100 mm
```

The physical distance is:

$$
\sqrt{160^2+100^2}
=
188.68\text{ mm}
$$

The viewer displays:

```text
Distance: 188.68 mm
```

Notice that using only one spacing value would produce an incorrect result when the row and column spacing are different.

---

## Using the Distance Tool

The interaction is intentionally simple:

1. Open a DICOM image.
2. Hold the `Shift` key.
3. Left-click the first measurement point.
4. Hold `Shift` and left-click the second point.
5. Read the result displayed in millimeters or pixels.
6. Shift-click again to begin a new measurement.

The viewer controls are:

| Action                         | Control                  |
| ------------------------------ | ------------------------ |
| Zoom                           | Mouse wheel              |
| Pan                            | Left mouse drag          |
| Adjust Window Center and Width | Right mouse drag         |
| Inspect Pixel and HU values    | Mouse movement           |
| Measure distance               | Shift + Left Click twice |
| Select an ROI                  | Ctrl + Left Drag         |

---

## Result

After two points are selected, the viewer:

* Draws a yellow line between the points
* Displays circular start and end markers
* Reads the DICOM pixel spacing
* Converts the coordinate differences to millimeters
* Shows the calculated distance
* Uses pixels when physical spacing is unavailable

![DICOM distance measurement result](/assets/images/posts/post-09-dicom-distance/dicom-distance-measurement.png)

---

## Important Considerations

This implementation is useful for learning and viewer prototyping, but medical measurements require careful interpretation.

### Pixel Spacing Availability

Some DICOM files may not contain `PixelSpacing`.

Other spacing-related attributes may exist depending on the modality and image type. Their meanings are not always interchangeable.

### Image Calibration

The spacing metadata must correctly describe the displayed image. Resized, derived, reconstructed, or secondary-capture images may require additional validation.

### Measurement Plane

This example measures distance within a single two-dimensional image plane.

It does not calculate three-dimensional distance across different slices.

### Diagnostic Use

A learning project should not be treated as a validated diagnostic medical device. Clinical software requires additional verification, calibration, testing, risk management, and regulatory compliance.

---

## What We Learned

In this post, we learned how to:

* Convert mouse positions to DICOM image coordinates
* Select two points with PyQt5
* Calculate Euclidean distance in pixels
* Interpret DICOM `PixelSpacing`
* Apply row and column spacing correctly
* Calculate physical distance in millimeters
* Draw measurement lines and endpoint markers
* Handle missing spacing metadata safely
* Reset measurement graphics and state

This feature connects DICOM metadata with interactive image analysis.

Instead of treating the DICOM image as a simple bitmap, the viewer now uses spatial information to produce meaningful physical measurements.

---

## Source Code

The complete source code is available on GitHub:

[View the PACS-DICOM-Toolkit repository on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

In the next post, we will add rectangular ROI measurement to the DICOM viewer.

The ROI tool will allow users to select an image region and calculate:

* ROI width and height
* Mean Hounsfield Unit
* Minimum Hounsfield Unit
* Maximum Hounsfield Unit

**Next: Measuring ROI Statistics in DICOM Images with Python and PyQt5**
