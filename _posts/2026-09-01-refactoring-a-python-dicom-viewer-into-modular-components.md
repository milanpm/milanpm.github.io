---
layout: post
title: "Refactoring a Python DICOM Viewer into Modular Components"
date: 2026-09-01 23:30:00 +0900
categories: [medical-imaging]
tags: [dicom, python, pyqt5, pydicom, refactoring, software-design, medical-imaging]
description: "Learn how to refactor a Python DICOM viewer into focused modules for DICOM loading, windowing, anonymization, image interaction, and UI coordination."
---

As a DICOM viewer gains features, keeping every responsibility in one Python file becomes increasingly difficult.

Our viewer began with basic DICOM loading and display. It later added metadata extraction, anonymization, PNG export, zoom and pan, Window Center and Width controls, pixel and HU inspection, distance measurement, and ROI statistics.

All of these features work together, but they do not belong to the same responsibility.

In this post, we will refactor the viewer into focused modules:

- `dicom_loader.py` for DICOM reading and metadata extraction
- `windowing.py` for Window Center and Width processing
- `anonymizer.py` for DICOM de-identification
- `image_view.py` for image interaction and overlays
- `main.py` for UI construction and application coordination

This structure makes the project easier to understand, test, maintain, and extend.

---

## Why Refactoring Became Necessary

A single-file application can be useful during early development. The entire program is visible in one place, and a small prototype can be changed quickly.

The disadvantages become clearer as features accumulate:

- The main window class becomes too large.
- UI code is mixed with DICOM processing logic.
- Mouse-event code is mixed with file operations.
- Reusable functions are difficult to test independently.
- A change in one feature can affect unrelated code.
- New contributors need more time to understand the program.

The goal of refactoring is not simply to reduce the number of lines in `main.py`.

The real goal is to give each module a clear reason to change.

---

## Separation of Concerns

Separation of concerns means grouping code according to its responsibility.

For this DICOM viewer, the main concerns are:

| Concern | Module | Responsibility |
| --- | --- | --- |
| DICOM data | `dicom_loader.py` | Read a file, validate pixel data, convert pixels, extract metadata |
| Display processing | `windowing.py` | Apply Window Center and Width and create an 8-bit image |
| Privacy | `anonymizer.py` | Remove or replace identifying DICOM information |
| Image interaction | `image_view.py` | Zoom, pan, mouse events, distance lines, and ROI overlays |
| Application flow | `main.py` | Build the UI, connect signals, maintain state, and coordinate modules |

Each module exposes a small interface that the main application can call.

---

## Refactored Project Structure

The core viewer structure is:

```text
PACS-DICOM-Toolkit/
├── src/
│   ├── main.py
│   ├── dicom_loader.py
│   ├── windowing.py
│   ├── anonymizer.py
│   ├── image_view.py
│   └── dicom_network.py
└── README.md
```

The `dicom_network.py` module was added later for C-ECHO, C-STORE, C-FIND, C-MOVE, and Storage SCP functionality. Its addition demonstrates an important benefit of the refactoring: a new technical area can be introduced without placing all network logic inside the viewer code.

---

## Module Dependencies

The main window coordinates the focused modules:

```text
main.py
├── dicom_loader.py
├── windowing.py
├── anonymizer.py
├── image_view.py
└── dicom_network.py
```

The main module imports the functions and classes it needs:

```python
from anonymizer import save_anonymized_dicom
from dicom_loader import extract_metadata, load_dicom
from image_view import ImageView
from windowing import apply_window
```

This is easier to understand than embedding every implementation directly in `DicomViewer`.

---

## `dicom_loader.py`: Loading and Preparing DICOM Data

The loader is responsible for reading the DICOM file and returning data that the rest of the application can use.

```python
from pathlib import Path

import numpy as np
import pydicom
from pydicom.dataset import Dataset
```

The `load_dicom()` function validates the path and confirms that pixel data exists:

```python
def load_dicom(file_path: str) -> tuple[Dataset, np.ndarray]:
    """Read a DICOM file and return its dataset and pixel array."""
    path = Path(file_path)

    if not path.is_file():
        raise FileNotFoundError(f"DICOM file not found: {path}")

    dataset = pydicom.dcmread(str(path))

    if "PixelData" not in dataset:
        raise ValueError("The selected DICOM file has no pixel data.")
```

It converts stored pixels to a floating-point array and applies rescaling:

```python
pixel_array = dataset.pixel_array.astype(np.float32)

slope = float(getattr(dataset, "RescaleSlope", 1.0))
intercept = float(getattr(dataset, "RescaleIntercept", 0.0))

pixel_array = pixel_array * slope + intercept

return dataset, pixel_array
```

The returned objects have distinct roles:

- `dataset` provides DICOM elements and metadata.
- `pixel_array` provides rescaled numeric values for display and measurement.

Keeping this logic in the loader prevents the main window from needing to know every detail of file validation and HU conversion.

---

## Extracting Frequently Used Metadata

The loader module also provides a small metadata interface:

```python
def extract_metadata(dataset: Dataset) -> dict[str, str]:
    """Extract frequently used metadata from a DICOM dataset."""
    return {
        "Patient Name": str(dataset.get("PatientName", "N/A")),
        "Patient ID": str(dataset.get("PatientID", "N/A")),
        "Study Date": str(dataset.get("StudyDate", "N/A")),
        "Modality": str(dataset.get("Modality", "N/A")),
        "Rows": str(dataset.get("Rows", "N/A")),
        "Columns": str(dataset.get("Columns", "N/A")),
        "Window Center": str(dataset.get("WindowCenter", "N/A")),
        "Window Width": str(dataset.get("WindowWidth", "N/A")),
        "Pixel Spacing": str(dataset.get("PixelSpacing", "N/A")),
    }
```

Using `dataset.get()` with `"N/A"` makes the UI resilient when an optional element is missing.

The main window can now update its labels without repeating DICOM lookup logic:

```python
metadata = extract_metadata(self.dataset)

self.patient_id_label.setText(metadata["Patient ID"])
self.patient_name_label.setText(metadata["Patient Name"])
self.study_date_label.setText(metadata["Study Date"])
self.modality_label.setText(metadata["Modality"])
self.pixel_spacing_label.setText(metadata["Pixel Spacing"])
```

---

## `windowing.py`: Isolating Display Transformation

DICOM pixel data often contains a much wider numeric range than an 8-bit display can show directly.

The `apply_window()` function converts a selected HU range into an 8-bit grayscale image:

```python
import numpy as np


def apply_window(
    pixel_array: np.ndarray,
    window_center: float,
    window_width: float,
) -> np.ndarray:
    """Apply Window Center/Width and return an 8-bit image."""
    if window_width <= 0:
        raise ValueError("Window width must be greater than 0.")

    window_min = window_center - window_width / 2
    window_max = window_center + window_width / 2

    windowed = np.clip(pixel_array, window_min, window_max)
    windowed = (windowed - window_min) / (window_max - window_min)
    windowed = (windowed * 255).astype(np.uint8)

    return windowed
```

The processing steps are:

1. Calculate the lower and upper window boundaries.
2. Clip values outside the selected range.
3. Normalize the range to `0.0–1.0`.
4. Scale the normalized values to `0–255`.
5. Convert the result to `uint8`.

Because this is a pure data-processing function, it can be tested without starting PyQt5.

---

## `anonymizer.py`: Separating Privacy-Sensitive Logic

DICOM anonymization deserves its own module because it changes patient-related metadata and must not be mixed casually with display logic.

The module defines the identifying fields to clear:

```python
IDENTIFYING_FIELDS = [
    "PatientBirthDate",
    "PatientSex",
    "PatientAddress",
    "PatientTelephoneNumbers",
    "OtherPatientIDs",
    "OtherPatientNames",
    "InstitutionName",
    "InstitutionAddress",
    "ReferringPhysicianName",
    "PerformingPhysicianName",
    "OperatorsName",
    "AccessionNumber",
]
```

The dataset is copied before modification:

```python
def anonymize_dataset(dataset: Dataset) -> Dataset:
    """Return an anonymized copy of a DICOM dataset."""
    anonymized = deepcopy(dataset)

    anonymized.PatientName = "ANONYMOUS"
    anonymized.PatientID = "ANONYMOUS"

    for field in IDENTIFYING_FIELDS:
        if hasattr(anonymized, field):
            setattr(anonymized, field, "")

    anonymized.remove_private_tags()
    anonymized.PatientIdentityRemoved = "YES"
    anonymized.DeidentificationMethod = "Basic metadata anonymization"

    return anonymized
```

Using `deepcopy()` protects the currently loaded dataset from being changed unexpectedly.

The file-saving function handles paths separately:

```python
def save_anonymized_dicom(input_path: str, output_path: str) -> Path:
    source = Path(input_path)
    destination = Path(output_path)

    if not source.is_file():
        raise FileNotFoundError(f"DICOM file not found: {source}")

    dataset = pydicom.dcmread(str(source))
    anonymized = anonymize_dataset(dataset)

    destination.parent.mkdir(parents=True, exist_ok=True)
    anonymized.save_as(str(destination))

    return destination
```

This learning implementation performs basic metadata anonymization. Production de-identification requires a formal policy, DICOM confidentiality profiles, validation, and handling of identifying information that may appear in pixels, sequences, private elements, or other attributes.

---

## `image_view.py`: Encapsulating Image Interaction

The custom `ImageView` class inherits from `QGraphicsView`:

```python
class ImageView(QGraphicsView):
```

It owns interaction-specific state and behavior, including:

- Mouse-wheel zoom
- Left-button panning
- Right-drag Window Center and Width adjustment
- Pixel-position reporting
- Distance measurement point selection
- ROI drag selection
- Measurement line and marker overlays
- ROI rectangle overlays
- View reset and fit-to-view behavior

Custom signals communicate user actions without requiring `ImageView` to perform DICOM calculations:

```python
zoom_changed = pyqtSignal(int)
window_changed = pyqtSignal(float, float)
pixel_position_changed = pyqtSignal(int, int)
measurement_point_selected = pyqtSignal(int, int)
roi_selected = pyqtSignal(int, int, int, int)
```

This signal-based design keeps the graphics widget reusable and reduces direct coupling with the main window.

---

## `main.py`: Coordinating the Application

After refactoring, `main.py` still has an important role.

The `DicomViewer` class:

- Builds the Viewer, Metadata, and Network interfaces
- Stores the current dataset and HU array
- Connects buttons and custom signals
- Opens file dialogs
- Calls the focused processing modules
- Converts processed arrays to `QImage` and `QPixmap`
- Updates labels and controls
- Handles user-facing error messages
- Coordinates DICOM network operations added later

For example, opening a DICOM file becomes an orchestration task:

```python
dataset, pixel_array = load_dicom(file_path)

self.dataset = dataset
self.pixel_array = pixel_array
self.current_file_path = file_path

self.update_metadata()
self.set_initial_window()
self.update_image()
self.image_view.fit_to_view()
```

The main window decides **when** these actions happen, while the modules decide **how** their focused work is performed.

---

## Data Flow Through the Viewer

When a user opens and views a DICOM file, data moves through the application as follows:

```text
DICOM file
    ↓ load_dicom()
Dataset + rescaled pixel array
    ├── extract_metadata() → UI metadata labels
    ├── apply_window() → 8-bit display image
    ├── pixel/ROI tools → HU measurements
    └── anonymizer → anonymized DICOM copy
```

The displayed 8-bit image and the measurement array remain separate:

- Windowing changes visualization.
- Measurements use the rescaled numeric pixel array.

This distinction prevents display normalization from corrupting HU measurements.

---

## Error Handling Across Module Boundaries

Focused modules raise meaningful exceptions:

```python
raise FileNotFoundError(f"DICOM file not found: {path}")
```

```python
raise ValueError("The selected DICOM file has no pixel data.")
```

```python
raise ValueError("Window width must be greater than 0.")
```

The UI layer catches these exceptions and presents them to the user with a message box.

This keeps reusable modules independent of PyQt5 while allowing the GUI to remain user-friendly.

---

## Benefits of the Refactored Design

### Readability

A developer looking for windowing logic can open `windowing.py` instead of searching through the complete UI class.

### Testability

Functions such as `apply_window()`, `load_dicom()`, and `anonymize_dataset()` can be tested independently.

### Reusability

The same loader or anonymizer could later be used by a command-line program, batch processor, or web service.

### Safer Changes

Modifying ROI drawing is less likely to affect DICOM file loading because those responsibilities belong to different modules.

### Easier Extension

The later addition of `dicom_network.py` shows how the project can grow with C-ECHO, C-STORE, Storage SCP, C-FIND, and C-MOVE features while keeping network protocols separated from core image processing.

---

## What Refactoring Does Not Solve Automatically

Splitting files does not guarantee good architecture.

Modules can still be tightly coupled if they share too much state or depend on one another in both directions. A large `main.py` can also continue growing as more UI and workflow logic is added.

Future improvements may include:

- Moving network widgets into a dedicated UI component
- Moving Viewer and Metadata tabs into separate widgets
- Adding data classes for application state
- Adding automated tests for pure processing functions
- Introducing logging instead of relying only on message boxes
- Adding type checks and continuous integration

Refactoring is an iterative process rather than a one-time cleanup.

---

## Result

The viewer now has clearer boundaries between:

- DICOM file access
- Metadata extraction
- HU conversion
- Window Center and Width processing
- Anonymization
- Image interaction
- UI coordination
- DICOM networking

![Modular structure of the Python DICOM viewer](/assets/images/posts/post-11-dicom-refactoring/dicom-viewer-modular-structure.png)

---

## What We Learned

In this post, we learned how to:

- Recognize when a single-file GUI application needs refactoring
- Separate code by responsibility
- Keep DICOM loading independent from the UI
- Isolate Window Center and Width processing
- Separate privacy-sensitive anonymization logic
- Encapsulate mouse interaction in a custom graphics view
- Use PyQt signals to reduce coupling
- Keep the main window focused on coordination
- Prepare the codebase for future DICOM network features

The application still presents one integrated DICOM viewer to the user, but internally each component now has a clearer purpose.

---

## Source Code

The complete source code is available on GitHub:

[View the PACS-DICOM-Toolkit repository on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

With the viewer core organized into focused modules, the next stage is DICOM networking.

We will begin by verifying communication between DICOM Application Entities using C-ECHO.

**Next: Verifying DICOM Connectivity with C-ECHO and pynetdicom**
