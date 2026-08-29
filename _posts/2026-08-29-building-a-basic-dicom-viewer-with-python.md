---
layout: post
title: "Building a Basic DICOM Viewer with Python and PyQt5"
date: 2026-08-29 19:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, PyQt5, pydicom, Medical Imaging]
---

# Building a Basic DICOM Viewer with Python and PyQt5

DICOM is the standard file format and communication framework used for medical images such as X-rays, CT scans, and MRI studies.

Unlike ordinary image files, a DICOM file contains both pixel data and medical metadata.

In this first PACS/DICOM project, I built a basic desktop viewer that can:

- open a DICOM file
- read its dataset with `pydicom`
- extract the pixel data as a NumPy array
- display basic patient and image information
- apply Window Center and Window Width
- convert medical pixel data into an 8-bit grayscale image
- display the result through a PyQt5 interface

This project is the starting point for building a more complete PACS and DICOM toolkit.

![Grayscale test image displayed by the Python DICOM viewer](/assets/images/posts/post-08/dicom-test-image.png)

*The 512 × 512 grayscale test image used to verify DICOM loading, windowing, and display.*

---

## 1. What Is a DICOM File?

DICOM stands for:

```text
Digital Imaging and Communications in Medicine
```

A DICOM file is different from a common PNG or JPEG image.

It may contain:

- patient information
- study information
- series information
- imaging modality
- image dimensions
- acquisition parameters
- Window Center and Window Width
- raw medical-image pixel data

This combination of metadata and pixel data makes DICOM useful for medical imaging systems, including PACS.

---

## 2. Project Structure

The first version of the project separates DICOM loading, image windowing, and the user interface.

```text
PACS-DICOM-Toolkit/
├── samples/
│   └── test_image.dcm
├── src/
│   ├── dicom_loader.py
│   ├── main.py
│   └── windowing.py
├── tests/
│   └── create_test_dicom.py
└── requirements.txt
```

Each module has a specific responsibility.

| File | Responsibility |
| --- | --- |
| `dicom_loader.py` | Reads the DICOM file and extracts its pixel array |
| `windowing.py` | Converts medical pixel values into an 8-bit display image |
| `main.py` | Builds the PyQt5 viewer and connects the components |
| `create_test_dicom.py` | Creates a sample DICOM file for testing |
| `requirements.txt` | Lists the required Python packages |

Separating these responsibilities makes the application easier to understand and extend.

---

## 3. Installing the Required Packages

The first viewer uses four Python packages.

```text
PyQt5
pydicom
numpy
Pillow
```

They can be installed with:

```bash
pip install -r requirements.txt
```

Their roles are:

| Package | Purpose |
| --- | --- |
| `PyQt5` | Desktop graphical user interface |
| `pydicom` | Reading and working with DICOM datasets |
| `NumPy` | Pixel-array processing |
| `Pillow` | General image-processing support |

---

## 4. Loading a DICOM File

DICOM loading is handled in `dicom_loader.py`.

```python
from pathlib import Path

import numpy as np
import pydicom
from pydicom.dataset import Dataset


def load_dicom(file_path: str) -> tuple[Dataset, np.ndarray]:
    """DICOM 파일을 읽고 데이터셋과 픽셀 배열을 반환합니다."""
    path = Path(file_path)

    if not path.is_file():
        raise FileNotFoundError(f"DICOM file not found: {path}")

    dataset = pydicom.dcmread(str(path))

    if "PixelData" not in dataset:
        raise ValueError("The selected DICOM file has no pixel data.")

    pixel_array = dataset.pixel_array.astype(np.float32)

    return dataset, pixel_array
```

The function returns two objects:

```text
dataset     → DICOM metadata and attributes
pixel_array → image pixels represented as a NumPy array
```

This separation is important because the viewer needs both the medical information and the image data.

---

## 5. Reading the Dataset with pydicom

The following line reads the DICOM file:

```python
dataset = pydicom.dcmread(str(path))
```

The returned dataset provides access to DICOM attributes such as:

```python
dataset.PatientID
dataset.PatientName
dataset.Modality
dataset.WindowCenter
dataset.WindowWidth
```

Not every DICOM file contains every attribute.

For that reason, the viewer safely reads optional fields with `getattr()`:

```python
getattr(self.dataset, "PatientID", "Unknown")
```

If `PatientID` is unavailable, the interface displays `Unknown` instead of raising an exception.

---

## 6. Extracting the Pixel Array

The DICOM image is extracted with:

```python
pixel_array = dataset.pixel_array.astype(np.float32)
```

`dataset.pixel_array` converts the encoded DICOM Pixel Data into a NumPy array.

The array contains the numerical intensity values recorded by the imaging system.

The values are converted to `float32` so that later calculations do not suffer from unintended integer truncation.

For a two-dimensional image, the array shape is:

```text
(rows, columns)
```

This is different from the displayed image size, which is normally written as:

```text
width × height
columns × rows
```

---

## 7. Checking for Pixel Data

Some DICOM objects do not contain images.

Examples may include:

- structured reports
- presentation states
- metadata-only objects
- non-image DICOM instances

The loader therefore checks for Pixel Data:

```python
if "PixelData" not in dataset:
    raise ValueError("The selected DICOM file has no pixel data.")
```

This prevents the viewer from attempting to display a non-image DICOM object.

---

## 8. Why Windowing Is Necessary

Medical-image pixel values often have a much wider range than an ordinary 8-bit monitor image.

An 8-bit grayscale display uses values from:

```text
0 to 255
```

The DICOM pixel array may contain a much larger range.

Windowing selects a useful intensity interval and maps it into the display range.

The two main parameters are:

```text
Window Center → the midpoint of the visible intensity range
Window Width  → the size of the visible intensity range
```

The minimum and maximum values are calculated as:

```text
window_min = center - width / 2
window_max = center + width / 2
```

---

## 9. Applying Window Center and Window Width

Windowing is implemented in `windowing.py`.

```python
import numpy as np


def apply_window(
    pixel_array: np.ndarray,
    window_center: float,
    window_width: float,
) -> np.ndarray:
    """Window Level/Width를 적용하여 8-bit 영상으로 변환합니다."""
    if window_width <= 0:
        raise ValueError("Window width must be greater than 0.")

    window_min = window_center - window_width / 2
    window_max = window_center + window_width / 2

    windowed = np.clip(pixel_array, window_min, window_max)
    windowed = (windowed - window_min) / (window_max - window_min)
    windowed = (windowed * 255).astype(np.uint8)

    return windowed
```

The processing sequence is:

```text
Original pixel values
        ↓
Clip values to the selected window
        ↓
Normalize the values to 0.0–1.0
        ↓
Scale the values to 0–255
        ↓
Convert the result to uint8
```

The result can then be displayed as an ordinary grayscale image.

---

## 10. Clipping Pixel Values

The viewer clips pixel values with:

```python
windowed = np.clip(pixel_array, window_min, window_max)
```

Values below `window_min` become the minimum value.

Values above `window_max` become the maximum value.

Conceptually:

```text
Below the window → black
Inside the window → grayscale
Above the window → white
```

Changing the window changes which structures are visible in the medical image.

---

## 11. Building the PyQt5 Interface

The main window is implemented as a `QMainWindow`.

```python
class DicomViewer(QMainWindow):
    def __init__(self):
        super().__init__()

        self.dataset = None
        self.pixel_array = None

        self.setWindowTitle("PACS DICOM Toolkit")
        self.resize(900, 700)
```

The viewer stores:

```text
self.dataset     → currently loaded DICOM dataset
self.pixel_array → currently loaded pixel data
```

Before a file is opened, both values are `None`.

---

## 12. Opening a DICOM File

The Open DICOM button is connected to the file-selection method.

```python
self.open_button = QPushButton("Open DICOM")
self.open_button.clicked.connect(self.open_dicom)
```

The file dialog filters DICOM files:

```python
file_path, _ = QFileDialog.getOpenFileName(
    self,
    "Open DICOM File",
    str(Path.cwd() / "samples"),
    "DICOM Files (*.dcm);;All Files (*)",
)
```

After a file is selected, the viewer:

1. loads the dataset and pixel array
2. updates the displayed metadata
3. determines the initial window settings
4. renders the image

```python
self.dataset, self.pixel_array = load_dicom(file_path)
self.update_metadata()
self.set_initial_window()
self.update_image()
```

---

## 13. Displaying Basic DICOM Information

The first viewer displays five useful values:

- Patient ID
- Patient Name
- Modality
- Image Size
- Pixel Range

Example:

```python
self.patient_id_label.setText(
    str(getattr(self.dataset, "PatientID", "Unknown"))
)
self.patient_name_label.setText(
    str(getattr(self.dataset, "PatientName", "Unknown"))
)
self.modality_label.setText(
    str(getattr(self.dataset, "Modality", "Unknown"))
)
```

The image size and pixel range are obtained from the NumPy array:

```python
rows, columns = self.pixel_array.shape

self.image_size_label.setText(f"{columns} x {rows}")
self.pixel_range_label.setText(
    f"{self.pixel_array.min():.0f} ~ {self.pixel_array.max():.0f}"
)
```

These values help verify that the selected DICOM file was loaded correctly.

---

## 14. Selecting the Initial Window

The viewer first calculates fallback values from the pixel range:

```python
pixel_min = float(self.pixel_array.min())
pixel_max = float(self.pixel_array.max())

default_center = (pixel_min + pixel_max) / 2
default_width = max(pixel_max - pixel_min, 1)
```

It then checks whether the DICOM dataset contains Window Center and Window Width:

```python
window_center = float(
    getattr(self.dataset, "WindowCenter", default_center)
)
window_width = float(
    getattr(self.dataset, "WindowWidth", default_width)
)
```

This provides two advantages:

- stored DICOM window settings are used when available
- calculated fallback values are used when the tags are missing

---

## 15. Converting the NumPy Array to QImage

After windowing, the image is stored as a two-dimensional `uint8` NumPy array.

```python
windowed = apply_window(
    self.pixel_array,
    self.window_center_spin.value(),
    self.window_width_spin.value(),
)
```

The array is made contiguous in memory:

```python
windowed = np.ascontiguousarray(windowed)
```

It is then converted into a PyQt5 `QImage`:

```python
height, width = windowed.shape

image = QImage(
    windowed.data,
    width,
    height,
    windowed.strides[0],
    QImage.Format_Grayscale8,
).copy()
```

`QImage.Format_Grayscale8` tells Qt that each pixel is represented by one 8-bit grayscale value.

The `.copy()` call ensures that the `QImage` owns a safe copy of the image data.

---

## 16. Displaying the Image

The `QImage` is converted into a `QPixmap`:

```python
pixmap = QPixmap.fromImage(image)
```

The image is scaled to fit the display label:

```python
pixmap = pixmap.scaled(
    self.image_label.size(),
    Qt.KeepAspectRatio,
    Qt.SmoothTransformation,
)
```

`Qt.KeepAspectRatio` prevents the medical image from being stretched horizontally or vertically.

Finally, the pixmap is displayed:

```python
self.image_label.setPixmap(pixmap)
```

---

## 17. Interactive Window Controls

The viewer provides spin boxes for Window Center and Window Width.

```python
self.window_center_spin.valueChanged.connect(self.update_image)
self.window_width_spin.valueChanged.connect(self.update_image)
```

Whenever either value changes, `update_image()` runs again.

This allows the user to adjust image contrast interactively without reopening the file.

The original pixel array remains unchanged.

Only the displayed 8-bit image is recalculated.

---

## 18. Error Handling

DICOM loading is protected by a `try` and `except` block.

```python
try:
    self.dataset, self.pixel_array = load_dicom(file_path)
    self.update_metadata()
    self.set_initial_window()
    self.update_image()
except Exception as error:
    QMessageBox.critical(self, "DICOM Load Error", str(error))
```

Possible errors include:

- missing files
- invalid DICOM data
- missing Pixel Data
- unsupported pixel-data encoding
- invalid Window Width
- unreadable datasets

Instead of terminating silently, the application displays an error message to the user.

---

## 19. Running the Viewer

The application starts by creating a `QApplication`.

```python
def main():
    app = QApplication(sys.argv)
    viewer = DicomViewer()
    viewer.show()
    sys.exit(app.exec_())
```

The entry-point condition prevents the viewer from starting automatically if the file is imported as a module:

```python
if __name__ == "__main__":
    main()
```

Run the viewer from the project directory:

```bash
python src/main.py
```

The application opens a desktop window where a DICOM file can be selected and displayed.

---

## 20. Processing Pipeline

The complete image-display pipeline is:

```text
DICOM file
    ↓
pydicom.dcmread()
    ↓
DICOM Dataset
    ↓
dataset.pixel_array
    ↓
NumPy float32 array
    ↓
Window Center / Window Width
    ↓
8-bit grayscale NumPy array
    ↓
QImage
    ↓
QPixmap
    ↓
PyQt5 QLabel
```

Understanding this pipeline is essential for building more advanced medical-image viewers.

---

## 21. Why This First Project Matters

This basic viewer provides the foundation for later PACS/DICOM features.

The same dataset and pixel array can later support:

- complete metadata browsing
- metadata search
- DICOM anonymization
- PNG export
- zoom and pan
- pixel-value inspection
- distance measurement
- ROI measurement
- PACS network verification with C-ECHO
- DICOM transmission with C-STORE
- study search with C-FIND
- image retrieval with C-MOVE or C-GET

The first viewer therefore establishes both the image-processing pipeline and the desktop application structure.

---

## What I Learned

In this lesson, I learned that:

- DICOM files contain both medical metadata and pixel data.
- `pydicom.dcmread()` reads a DICOM dataset.
- `dataset.pixel_array` converts Pixel Data into a NumPy array.
- some DICOM objects may not contain Pixel Data.
- medical pixel values must be converted for ordinary display.
- Window Center defines the midpoint of the visible range.
- Window Width defines the size of the visible range.
- `np.clip()` limits pixels to the selected window.
- normalized values can be scaled to the 8-bit range.
- `QImage.Format_Grayscale8` displays an 8-bit grayscale array.
- `Qt.KeepAspectRatio` prevents image distortion.
- PyQt5 signals can update the image interactively.
- separating loading, windowing, and UI code improves maintainability.

The most important idea from this lesson is:

> **A DICOM viewer must interpret both the dataset and its pixel values before the medical image can be displayed correctly.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 1 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/2df921f)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

Now that the application can load and display a DICOM image, the next step is to protect patient information.

In the next lesson, we will explore:

- DICOM patient identifiers
- sensitive metadata fields
- replacing patient information
- saving an anonymized DICOM copy
- verifying that the original file remains unchanged

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
