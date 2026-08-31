---
layout: post
title: "Interactive DICOM Window Level Adjustment with Python and PyQt5"
date: 2026-08-31 23:20:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, PyQt5, Window Level, Window Width, pydicom, Medical Imaging]
description: "Learn how to implement interactive DICOM Window Center and Window Width controls in Python and PyQt5 using NumPy, DICOM rescale values, SpinBoxes, and right-mouse dragging."
---

DICOM images often contain a much wider range of pixel values than a standard 8-bit display can show at one time.

If the complete range is mapped directly to grayscale, clinically or technically important contrast may become difficult to see. A DICOM viewer therefore needs **Window Center** and **Window Width** controls that select which part of the pixel-value range is displayed.

In the previous lesson, I added zoom and pan navigation to the PyQt5 viewer. In this lesson, I will implement interactive Window/Level adjustment using:

- DICOM `WindowCenter` and `WindowWidth` values
- `RescaleSlope` and `RescaleIntercept`
- NumPy clipping and normalization
- PyQt5 SpinBoxes
- right-mouse dragging
- Qt signals and slots
- a Reset Window button

---

## 1. What Are Window Center and Window Width?

Windowing maps a selected range of medical-image values to visible grayscale values.

- **Window Center (WC)**, also called Window Level, is the midpoint of the displayed range.
- **Window Width (WW)** is the size of the displayed range.

The lower and upper boundaries can be calculated as:

```text
window_min = window_center - window_width / 2
window_max = window_center + window_width / 2
```

For example, if:

```text
Window Center = 40
Window Width  = 400
```

then:

```text
window_min = 40 - 400 / 2 = -160
window_max = 40 + 400 / 2 = 240
```

Values below the lower boundary become black, values above the upper boundary become white, and values inside the range are mapped to intermediate grayscale levels.

---

## 2. How Center and Width Affect the Image

Changing the two values produces different visual effects.

| Control | Primary effect |
|---|---|
| Increase Window Center | Shifts the displayed range toward higher values |
| Decrease Window Center | Shifts the displayed range toward lower values |
| Increase Window Width | Displays a wider range with lower contrast |
| Decrease Window Width | Displays a narrower range with higher contrast |

A narrow window emphasizes small differences inside a limited range. A wide window includes more values but compresses them into the same 256 grayscale levels.

Window settings change only the displayed representation. They do not replace or overwrite the original DICOM Pixel Data.

---

## 3. Applying Rescale Slope and Intercept

Before windowing, the loader converts the stored pixel values into modality values:

```python
pixel_array = dataset.pixel_array.astype(np.float32)

slope = float(getattr(dataset, "RescaleSlope", 1.0))
intercept = float(getattr(dataset, "RescaleIntercept", 0.0))
pixel_array = pixel_array * slope + intercept
```

The conversion is:

```text
modality_value = stored_value × RescaleSlope + RescaleIntercept
```

Default values are used when the attributes are absent:

```text
RescaleSlope     = 1.0
RescaleIntercept = 0.0
```

For many CT images, the resulting modality values represent Hounsfield Units. However, this should not be assumed for every DICOM modality or object. The meaning depends on the dataset and its metadata.

Converting to `float32` prevents arithmetic problems and preserves negative values during rescaling and windowing.

---

## 4. Implementing the Windowing Function

The windowing calculation is separated into `windowing.py`:

```python
import numpy as np


def apply_window(
    pixel_array: np.ndarray,
    window_center: float,
    window_width: float,
) -> np.ndarray:
    """Apply Window Center/Width and convert the result to an 8-bit image."""
    if window_width <= 0:
        raise ValueError("Window width must be greater than 0.")

    window_min = window_center - window_width / 2
    window_max = window_center + window_width / 2

    windowed = np.clip(pixel_array, window_min, window_max)
    windowed = (windowed - window_min) / (window_max - window_min)
    windowed = (windowed * 255).astype(np.uint8)

    return windowed
```

This function performs three main operations:

1. Clips values to the selected window.
2. Normalizes the clipped values to the range `0.0–1.0`.
3. Converts the normalized values to `0–255` unsigned integers.

---

## 5. Validating Window Width

Window Width must be greater than zero:

```python
if window_width <= 0:
    raise ValueError("Window width must be greater than 0.")
```

A zero width would make the normalization denominator zero:

```python
window_max - window_min
```

A negative width also has no useful meaning in this implementation. Validation prevents division-by-zero errors and invalid ranges.

---

## 6. Clipping and Normalizing with NumPy

`np.clip()` limits every pixel to the selected boundaries:

```python
windowed = np.clip(pixel_array, window_min, window_max)
```

The result is normalized:

```python
windowed = (windowed - window_min) / (window_max - window_min)
```

After normalization:

```text
window_min → 0.0
window_center → approximately 0.5
window_max → 1.0
```

Finally, the values are mapped to 8-bit grayscale:

```python
windowed = (windowed * 255).astype(np.uint8)
```

The output is suitable for `QImage.Format_Grayscale8`.

This is a clear educational linear-windowing implementation. Production DICOM software may need the precise DICOM VOI LUT rules, including the specified boundary behavior, VOI LUT Sequence support, and alternative VOI LUT functions.

---

## 7. Adding Window Controls to the Interface

Two SpinBoxes provide precise numeric control:

```python
# Window Center
self.window_center_spin = QSpinBox()
self.window_center_spin.setRange(-65535, 65535)
self.window_center_spin.setValue(32768)
self.window_center_spin.valueChanged.connect(
    self.on_window_controls_changed
)

# Window Width
self.window_width_spin = QSpinBox()
self.window_width_spin.setRange(1, 131070)
self.window_width_spin.setValue(65536)
self.window_width_spin.valueChanged.connect(
    self.on_window_controls_changed
)
```

Window Center allows negative values because rescaled modality values can be negative. Window Width begins at `1`, preventing a zero-width window.

The controls are placed in a form layout:

```python
window_layout = QFormLayout()
window_layout.addRow("Window Center:", self.window_center_spin)
window_layout.addRow("Window Width:", self.window_width_spin)
```

Whenever either value changes, the viewer recalculates the displayed image.

---

## 8. Reading Initial Values from the DICOM Dataset

When a file is loaded, the viewer first calculates fallback values from the pixel range:

```python
pixel_min = float(self.pixel_array.min())
pixel_max = float(self.pixel_array.max())

default_center = (pixel_min + pixel_max) / 2
default_width = max(pixel_max - pixel_min, 1)
```

It then attempts to use the DICOM Window Center and Window Width attributes:

```python
window_center = self.get_numeric_value(
    getattr(
        self.dataset,
        "WindowCenter",
        default_center,
    ),
    default_center,
)

window_width = self.get_numeric_value(
    getattr(
        self.dataset,
        "WindowWidth",
        default_width,
    ),
    default_width,
)
```

This provides two levels of fallback:

1. Use the DICOM window attributes when available and valid.
2. Otherwise derive a window from the minimum and maximum modality values.

---

## 9. Handling Single and Multiple DICOM Values

DICOM Window Center and Window Width may contain one value or multiple alternative values. The helper selects the first numeric value:

```python
@staticmethod
def get_numeric_value(value, default):
    """Convert a single value or the first of multiple DICOM values to a number."""
    try:
        if isinstance(value, (list, tuple)):
            value = value[0]

        if hasattr(value, "__len__") and not isinstance(
            value,
            (str, bytes),
        ):
            value = value[0]

        return float(value)

    except (TypeError, ValueError, IndexError):
        return float(default)
```

If conversion fails, the calculated fallback is returned. Selecting the first item is sufficient for this lesson, although a future viewer could present every available window preset to the user.

---

## 10. Avoiding Duplicate Signal Processing

The initial values must be placed in both SpinBoxes. Setting a SpinBox programmatically normally emits `valueChanged`, which could trigger unnecessary image updates before initialization is complete.

Signals are temporarily blocked:

```python
self.window_center_spin.blockSignals(True)
self.window_width_spin.blockSignals(True)

self.window_center_spin.setValue(
    round(window_center)
)
self.window_width_spin.setValue(
    max(round(window_width), 1)
)

self.window_center_spin.blockSignals(False)
self.window_width_spin.blockSignals(False)
```

The values are then stored as the reset defaults:

```python
self.image_view.set_window_values(
    self.window_center_spin.value(),
    self.window_width_spin.value(),
    set_default=True,
)
self.update_window_label()
```

This prevents duplicate work and makes the initialization flow predictable.

---

## 11. Storing Window State in ImageView

`ImageView` stores the current and default values:

```python
self.window_center = 0.0
self.window_width = 1.0
self.default_window_center = 0.0
self.default_window_width = 1.0
```

The setter enforces a minimum width:

```python
def set_window_values(self, center, width, set_default=False):
    """Store the current Window Center and Window Width values."""
    self.window_center = float(center)
    self.window_width = max(float(width), 1.0)

    if set_default:
        self.default_window_center = self.window_center
        self.default_window_width = self.window_width
```

The default values represent the window selected when the current DICOM file was opened.

---

## 12. Starting Right-Mouse Window Adjustment

The viewer uses the right mouse button to start interactive adjustment:

```python
def mousePressEvent(self, event):
    """Start Window adjustment when the right mouse button is pressed."""
    if (
        event.button() == Qt.RightButton
        and not self.pixmap_item.pixmap().isNull()
    ):
        self.adjusting_window = True
        self.window_drag_start = event.pos()
        self.drag_start_center = self.window_center
        self.drag_start_width = self.window_width
        self.setDragMode(QGraphicsView.NoDrag)
        self.setCursor(Qt.SizeAllCursor)
        event.accept()
        return

    super().mousePressEvent(event)
```

At the beginning of the drag, the code records:

- the mouse position
- the current Window Center
- the current Window Width

The normal left-button pan mode is temporarily disabled, and the cursor changes to indicate an adjustment operation.

Other mouse events are passed to the base class, preserving the existing pan behavior.

---

## 13. Converting Mouse Movement into WC and WW

While the right button is held, mouse displacement is measured from the starting position:

```python
def mouseMoveEvent(self, event):
    """Adjust WW and WC using horizontal and vertical right-button dragging."""
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

    super().mouseMoveEvent(event)
```

The interaction mapping is:

| Drag direction | Result |
|---|---|
| Right | Increase Window Width |
| Left | Decrease Window Width |
| Up | Increase Window Center |
| Down | Decrease Window Center |

The sensitivity depends on the initial width:

```python
scale = max(abs(self.drag_start_width) / 500.0, 1.0)
```

A larger initial width therefore changes faster, while the minimum sensitivity remains `1.0`.

The width is clamped to at least `1.0`:

```python
width = max(1.0, calculated_width)
```

Each movement emits the new values, allowing the main window to refresh the controls and image immediately.

---

## 14. Ending the Adjustment

Releasing the right button restores the normal interaction mode:

```python
def mouseReleaseEvent(self, event):
    """End Window adjustment when the right mouse button is released."""
    if event.button() == Qt.RightButton and self.adjusting_window:
        self.adjusting_window = False
        self.window_drag_start = None
        self.setDragMode(QGraphicsView.ScrollHandDrag)
        self.unsetCursor()
        event.accept()
        return

    super().mouseReleaseEvent(event)
```

The temporary state is cleared, left-button panning is restored, and the cursor returns to normal.

---

## 15. Synchronizing the SpinBoxes and Image

The custom view declares a signal containing two floating-point values:

```python
window_changed = pyqtSignal(float, float)
```

The main window connects it to the controls:

```python
self.image_view.window_changed.connect(
    self.update_window_controls
)
```

The slot clamps and rounds the incoming values:

```python
def update_window_controls(self, center, width):
    """Apply Window values from ImageView to the SpinBoxes."""
    center = max(
        self.window_center_spin.minimum(),
        min(round(center), self.window_center_spin.maximum()),
    )
    width = max(
        self.window_width_spin.minimum(),
        min(round(width), self.window_width_spin.maximum()),
    )

    self.window_center_spin.blockSignals(True)
    self.window_width_spin.blockSignals(True)
    self.window_center_spin.setValue(center)
    self.window_width_spin.setValue(width)
    self.window_center_spin.blockSignals(False)
    self.window_width_spin.blockSignals(False)

    self.image_view.set_window_values(center, width)
    self.update_window_label()
    self.update_image()
```

Blocking signals here is essential. Without it, updating the SpinBoxes from a mouse-drag signal could emit `valueChanged` again and cause redundant updates or a feedback loop.

---

## 16. Responding to SpinBox Changes

Manual numeric changes travel in the opposite direction:

```python
def on_window_controls_changed(self, _value=None):
    """Apply changed SpinBox values to ImageView and the displayed image."""
    center = self.window_center_spin.value()
    width = self.window_width_spin.value()

    self.image_view.set_window_values(center, width)
    self.update_window_label()
    self.update_image()
```

This provides two synchronized input methods:

```text
SpinBox change → ImageView state → Label → Display image
Mouse drag → Signal → SpinBoxes → Label → Display image
```

---

## 17. Updating the Displayed Image

The original modality-value array is passed to `apply_window()`:

```python
def update_image(self):
    """Update the image using the current Window Center and Window Width."""
    if self.pixel_array is None:
        return

    windowed = apply_window(
        pixel_array=self.pixel_array,
        window_center=self.window_center_spin.value(),
        window_width=self.window_width_spin.value(),
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
```

`np.ascontiguousarray()` ensures a predictable memory layout for `QImage`. Calling `.copy()` gives the Qt image its own data instead of leaving it dependent on the temporary NumPy buffer.

Only the displayed 8-bit image is recreated. The stored source array remains available for future window changes.

---

## 18. Displaying the Current Values

A label provides immediate feedback:

```python
self.window_label = QLabel("WC: -  WW: -")
```

It is updated with:

```python
def update_window_label(self):
    """Display the current Window Center and Window Width."""
    self.window_label.setText(
        f"WC: {self.window_center_spin.value()}  "
        f"WW: {self.window_width_spin.value()}"
    )
```

Showing the exact values helps users understand the effect of mouse movement and reproduce a specific display setting.

---

## 19. Resetting the Window

The reset button is connected directly to `ImageView`:

```python
self.reset_window_button = QPushButton("Reset Window")
self.reset_window_button.clicked.connect(
    self.image_view.reset_window
)
```

`reset_window()` restores the values stored during file loading:

```python
def reset_window(self):
    """Restore the Window values selected when the DICOM file was opened."""
    self.set_window_values(
        self.default_window_center,
        self.default_window_width,
    )
    self.window_changed.emit(
        self.window_center,
        self.window_width,
    )
```

Emitting `window_changed` reuses the same synchronization path as right-mouse dragging.

---

## 20. Testing the Window/Level Controls

Run the application:

```bash
python src/main.py
```

Open a synthetic or anonymized DICOM file and test the following:

1. Confirm that initial WC and WW values appear.
2. Change Window Center with its SpinBox.
3. Change Window Width with its SpinBox.
4. Hold the right mouse button over the image.
5. Drag horizontally and confirm that WW changes.
6. Drag vertically and confirm that WC changes.
7. Confirm that the image updates continuously.
8. Confirm that left-button panning still works afterward.
9. Click **Reset Window**.
10. Confirm that the initial values and appearance return.

Expected controls:

```text
Mouse wheel         → Zoom
Left mouse drag     → Pan
Right drag left/right → Window Width
Right drag up/down    → Window Center
Reset View          → Reset zoom and pan
Reset Window        → Restore initial WC and WW
```

---

## 21. Common Problems

### The image is entirely black or white

Check whether the window range is appropriate for the rescaled pixel values. Also verify `RescaleSlope` and `RescaleIntercept`.

### Window Width reaches zero

Enforce a minimum of `1` in the SpinBox, state setter, drag calculation, and windowing function.

### Mouse dragging causes repeated updates

Use `blockSignals(True)` while assigning values received from `ImageView`, then restore signals afterward.

### Window values are missing

Calculate fallbacks from the minimum and maximum pixel values.

### A DICOM value cannot be converted to float

Handle single values, multi-values, and conversion errors through a helper function.

### Zoom or pan resets after every Window change

Update the pixmap without automatically calling `fit_to_view()`. In this version, `set_pixmap()` accepts a `fit` argument so routine window updates can preserve the existing view transformation.

---

## 22. Important DICOM Considerations

This implementation is intentionally focused and educational. A broader DICOM viewer may also need to support:

- multiple Window Center/Width presets
- `WindowCenterWidthExplanation`
- VOI LUT Sequence
- `VOILUTFunction`
- MONOCHROME1 inversion
- modality-specific processing
- Presentation LUT processing
- color images
- multi-frame objects
- Real World Value Mapping
- floating-point pixel data
- exact DICOM linear-window boundary rules

Windowing is part of a larger DICOM grayscale display pipeline. The order of modality transformation, VOI transformation, presentation processing, and display rendering matters.

This educational viewer is not a certified diagnostic application.

---

## What I Learned

In this lesson, I learned that:

- Window Center determines the midpoint of the displayed value range.
- Window Width determines the size of that range.
- narrower windows generally create stronger contrast inside a smaller range.
- `RescaleSlope` and `RescaleIntercept` convert stored pixels to modality values.
- rescaled CT values commonly represent Hounsfield Units, but this is not universal.
- `np.clip()` restricts values to the selected window.
- normalization converts the selected range to `0.0–1.0`.
- an 8-bit display image uses values from `0–255`.
- DICOM window attributes may be absent or contain multiple values.
- pixel-range fallbacks make the viewer more robust.
- `blockSignals()` prevents redundant signal handling.
- horizontal right dragging controls Window Width.
- vertical right dragging controls Window Center.
- a custom signal synchronizes `ImageView` and the main window.
- resetting restores the values chosen when the file was opened.
- changing the display window does not modify the original Pixel Data.
- a production DICOM display pipeline requires additional standard-specific handling.

The most important idea from this lesson is:

> **Window Center and Window Width select how modality values are mapped to visible grayscale without changing the original DICOM Pixel Data.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 7 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/b398689)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I added mouse-wheel zoom, left-button pan, fit-to-view, and zoom-percentage feedback:

[DICOM Image Zoom and Pan with Python and PyQt5](/medical-imaging/dicom/2026/08/31/dicom-image-zoom-and-pan-with-python-and-pyqt5.html)

---

## Next Step

Now that the viewer can navigate and adjust image contrast, the next step is to inspect individual image locations.

In the next lesson, we will explore:

- mapping mouse positions to image coordinates
- validating pixel boundaries
- reading stored and rescaled pixel values
- displaying Hounsfield Units when appropriate
- preserving coordinate accuracy during zoom and pan

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
