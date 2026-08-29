---
layout: post
title: "Displaying DICOM Metadata with Python and pydicom"
date: 2026-08-29 21:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, pydicom, Metadata, PyQt5, Medical Imaging]
---

# Displaying DICOM Metadata with Python and pydicom

A DICOM file is more than a medical image.

It is a structured dataset containing patient, study, image, acquisition, and display information.

In this lesson, I extended the Python DICOM viewer so that metadata extraction is handled by a dedicated function.

The viewer can now display:

- Patient Name
- Patient ID
- Study Date
- Modality
- Rows and Columns
- Window Center
- Window Width
- Pixel Spacing
- pixel-value range

This change also improves the software design by separating metadata extraction from the PyQt5 interface.

---

## 1. Image Data and Metadata

A DICOM dataset contains different kinds of information.

```text
DICOM dataset
├── Patient information
├── Study information
├── Series information
├── Equipment information
├── Image geometry
├── Display parameters
└── Pixel Data
```

Pixel Data represents the image.

Metadata explains the context and interpretation of that image.

A viewer needs both.

---

## 2. Why Metadata Matters

DICOM metadata can answer questions such as:

- Who does the image belong to?
- When was the study performed?
- Which imaging modality created it?
- What are the image dimensions?
- What physical distance does each pixel represent?
- Which window settings should be used?
- Which study and series contain the image?

Without metadata, an image may be visually available but medically and technically incomplete.

---

## 3. Metadata Added in This Lesson

The extraction function returns nine values.

| Display name | DICOM keyword | Purpose |
| --- | --- | --- |
| Patient Name | `PatientName` | Patient name stored in the dataset |
| Patient ID | `PatientID` | Patient identifier |
| Study Date | `StudyDate` | Date of the imaging study |
| Modality | `Modality` | Imaging equipment type |
| Rows | `Rows` | Image height in pixels |
| Columns | `Columns` | Image width in pixels |
| Window Center | `WindowCenter` | Midpoint of the display window |
| Window Width | `WindowWidth` | Width of the display window |
| Pixel Spacing | `PixelSpacing` | Physical spacing between pixels |

These values provide a useful summary without displaying every DICOM element.

---

## 4. Separating Metadata Extraction

The viewer previously read several fields directly inside `main.py`.

In this lesson, metadata access moves to `dicom_loader.py`.

```python
def extract_metadata(dataset: Dataset) -> dict[str, str]:
    """Extract key metadata from a DICOM dataset."""
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

The function accepts a DICOM `Dataset` and returns a dictionary containing display-ready strings.

---

## 5. Why Return a Dictionary?

The returned object has a simple structure:

```python
{
    "Patient Name": "TEST^PATIENT",
    "Patient ID": "TEST001",
    "Study Date": "20260829",
    "Modality": "OT",
    "Rows": "512",
    "Columns": "512",
    "Window Center": "32768.0",
    "Window Width": "65536.0",
    "Pixel Spacing": "N/A",
}
```

A dictionary is useful because:

- each value has a descriptive key
- the UI does not need to know how `pydicom` retrieves attributes
- missing-value handling is centralized
- the function can be tested independently
- additional fields can be added later

This reduces the responsibility of the main window.

---

## 6. Safely Reading Optional Attributes

The function uses `Dataset.get()`:

```python
dataset.get("StudyDate", "N/A")
```

The second argument is the fallback value.

If the field exists, its value is returned.

If the field is missing, the result is:

```text
N/A
```

This is safer than assuming that every file contains every attribute.

DICOM files vary according to:

- modality
- SOP Class
- manufacturer
- acquisition workflow
- export process
- conformance implementation

A robust viewer must expect missing optional fields.

---

## 7. Converting Values to Strings

Every value is converted with `str()`:

```python
str(dataset.get("PixelSpacing", "N/A"))
```

DICOM values may be represented by specialized pydicom types rather than ordinary Python strings.

Examples include:

- `PersonName`
- `DA` for date
- `DSfloat` or `DSdecimal`
- `MultiValue`
- `UID`

Converting values to strings provides a consistent format for display in a `QLabel`.

The original typed values should still be used when numerical calculations are required.

---

## 8. Displaying Patient Information

Patient Name and Patient ID are displayed using the extracted dictionary.

```python
self.patient_id_label.setText(metadata["Patient ID"])
self.patient_name_label.setText(metadata["Patient Name"])
```

DICOM Patient Name commonly uses caret separators.

Example:

```text
TEST^PATIENT
```

The components may represent family name, given name, middle name, prefix, and suffix.

A production viewer may format these components for users, but the learning viewer displays the stored representation directly.

---

## 9. Displaying Study Date

The UI adds a new label:

```python
self.study_date_label = QLabel("-")
```

It is added to the form:

```python
metadata_layout.addRow(
    "Study Date:",
    self.study_date_label,
)
```

The value is updated with:

```python
self.study_date_label.setText(metadata["Study Date"])
```

DICOM dates normally use the `YYYYMMDD` format.

Example:

```text
20260829
```

This representation is compact and unambiguous, although a viewer may later format it as `2026-08-29`.

---

## 10. Displaying Modality

The Modality attribute identifies the type of equipment or imaging process.

```python
self.modality_label.setText(metadata["Modality"])
```

Common examples include:

| Code | Meaning |
| --- | --- |
| `CT` | Computed Tomography |
| `MR` | Magnetic Resonance |
| `CR` | Computed Radiography |
| `DX` | Digital Radiography |
| `US` | Ultrasound |
| `OT` | Other |

The generated test file uses:

```text
OT
```

---

## 11. Displaying Image Dimensions

DICOM stores image height and width separately:

```text
Rows    → height
Columns → width
```

The viewer formats them in the familiar width-by-height order:

```python
self.image_size_label.setText(
    f'{metadata["Columns"]} x {metadata["Rows"]}'
)
```

For the test image:

```text
512 x 512
```

It is important not to reverse Rows and Columns when displaying image dimensions.

---

## 12. Understanding Pixel Spacing

Pixel Spacing describes the physical distance between the centers of neighboring pixels.

It is commonly represented by two values:

```text
[row spacing, column spacing]
```

The unit is normally millimeters.

Example:

```text
[0.5, 0.5]
```

This means each pixel covers approximately:

```text
0.5 mm vertically
0.5 mm horizontally
```

Pixel Spacing is important for:

- distance measurement
- area measurement
- physical image scale
- medical annotations
- quantitative image analysis

The generated test file does not define `PixelSpacing`, so this viewer correctly displays:

```text
N/A
```

Later measurement features must handle this missing value carefully rather than assuming a physical scale.

---

## 13. Window Center and Window Width

The extraction function also includes:

```python
"Window Center": str(
    dataset.get("WindowCenter", "N/A")
),
"Window Width": str(
    dataset.get("WindowWidth", "N/A")
),
```

These values control the visible grayscale range.

They were already used by the viewer's image-rendering pipeline.

Including them in the metadata dictionary provides one consistent location for retrieving important display information.

A DICOM attribute may contain one value or multiple values, so numerical processing still requires appropriate conversion logic.

---

## 14. Updating the PyQt5 Interface

The viewer imports both loading and metadata functions:

```python
from dicom_loader import extract_metadata, load_dicom
```

After a file is loaded, `update_metadata()` calls:

```python
metadata = extract_metadata(self.dataset)
```

It then updates the UI:

```python
self.patient_id_label.setText(metadata["Patient ID"])
self.patient_name_label.setText(metadata["Patient Name"])
self.study_date_label.setText(metadata["Study Date"])
self.modality_label.setText(metadata["Modality"])

self.image_size_label.setText(
    f'{metadata["Columns"]} x {metadata["Rows"]}'
)

self.pixel_spacing_label.setText(
    metadata["Pixel Spacing"]
)
```

The UI now consumes prepared values instead of reading DICOM attributes directly.

---

## 15. Adding the New Labels

Two labels are added in this lesson:

```python
self.study_date_label = QLabel("-")
self.pixel_spacing_label = QLabel("-")
```

They are registered in the form layout:

```python
metadata_layout.addRow(
    "Study Date:",
    self.study_date_label,
)
metadata_layout.addRow(
    "Pixel Spacing:",
    self.pixel_spacing_label,
)
```

The right-hand information panel now provides more useful study and image-geometry context.

---

## 16. Metadata Extraction Pipeline

The complete flow is:

```text
DICOM file
    ↓
pydicom.dcmread()
    ↓
Dataset
    ↓
extract_metadata()
    ↓
Dictionary of display strings
    ↓
PyQt5 labels
```

Image processing follows a separate path:

```text
Dataset
    ↓
Pixel Data
    ↓
NumPy array
    ↓
Windowing
    ↓
QImage and QPixmap
```

Separating these paths keeps metadata handling independent from image rendering.

---

## 17. The Generated Test Dataset

The project includes a script that creates a 512×512 DICOM test image.

Important metadata includes:

```python
dataset.PatientName = "TEST^PATIENT"
dataset.PatientID = "TEST001"
dataset.Modality = "OT"
dataset.StudyDate = now.strftime("%Y%m%d")
dataset.Rows = 512
dataset.Columns = 512
dataset.WindowCenter = 32768
dataset.WindowWidth = 65536
```

The pixel data contains a horizontal 16-bit grayscale gradient.

![512 by 512 grayscale test image used by the DICOM viewer](/assets/images/posts/post-08/dicom-test-image.png)

*The generated DICOM uses a horizontal intensity gradient to verify loading, windowing, and metadata display.*

Because `PixelSpacing` is not assigned by the generator, the metadata extractor returns `N/A` for that field.

This provides a useful test of missing-field handling.

---

## 18. Metadata Display vs. Complete Metadata Browser

This lesson displays a selected summary of important values.

It is not yet a complete metadata browser.

A complete metadata viewer should later provide:

- tag number
- DICOM keyword
- attribute name
- Value Representation
- value length
- formatted value
- nested sequence handling
- search and filtering
- private-tag indication

The summary panel is still useful because it exposes frequently needed information without overwhelming the user.

---

## 19. Design Improvement

Before this change, `main.py` handled both:

- reading DICOM attributes
- displaying the extracted values

After this change:

| Module | Responsibility |
| --- | --- |
| `dicom_loader.py` | Load the dataset and extract selected metadata |
| `main.py` | Display prepared metadata in the user interface |
| `windowing.py` | Convert pixel values for grayscale display |
| `anonymizer.py` | Create an anonymized dataset copy |

This modular structure makes the project easier to extend toward a full PACS/DICOM toolkit.

---

## 20. Possible Improvements

The metadata layer can be improved by:

- formatting DICOM dates
- formatting Person Name values
- preserving typed numerical values
- supporting multi-valued attributes
- including Study Instance UID
- including Series Instance UID
- including SOP Instance UID
- adding Series Description
- displaying body-part information
- handling nested sequences
- returning structured metadata objects
- adding metadata search

The next practical feature is a searchable metadata table.

---

## What I Learned

In this lesson, I learned that:

- DICOM files contain structured metadata and Pixel Data.
- metadata provides patient, study, modality, and geometry context.
- `Dataset.get()` safely handles missing attributes.
- `N/A` is a useful display fallback.
- pydicom values can be converted to strings for UI display.
- Rows represent image height.
- Columns represent image width.
- Pixel Spacing represents physical pixel dimensions.
- Pixel Spacing may be absent.
- metadata extraction should be separated from UI code.
- a dictionary provides a simple interface between modules.
- selected metadata is useful even before building a complete tag browser.

The most important idea from this lesson is:

> **A DICOM viewer should treat metadata extraction as a separate responsibility from image rendering and user-interface display.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 3 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/8e0c7f8)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I added basic DICOM anonymization:

[Anonymizing DICOM Files with Python and pydicom](/medical-imaging/dicom/2026/08/29/anonymizing-dicom-files-with-python-and-pydicom.html)

---

## Next Step

Now that the viewer can display selected metadata, the next step is to find attributes inside a complete DICOM dataset.

In the next lesson, we will explore:

- metadata tables
- DICOM tag numbers
- Value Representations
- attribute names
- keyword-based search
- filtering metadata rows

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
