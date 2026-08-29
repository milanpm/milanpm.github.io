---
layout: post
title: "Anonymizing DICOM Files with Python and pydicom"
date: 2026-08-29 20:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, pydicom, Anonymization, Medical Imaging]
---

# Anonymizing DICOM Files with Python and pydicom

A DICOM file may contain sensitive patient information in addition to medical-image pixel data.

Before a DICOM file is used for education, software testing, research, or public demonstration, identifying information must be handled carefully.

In this lesson, I extended the Python DICOM viewer with a basic anonymization feature.

The application can now:

- create a copy of the original DICOM dataset
- replace Patient Name and Patient ID
- clear selected identifying attributes
- remove private tags
- record that patient identity was removed
- save the anonymized dataset as a new DICOM file
- preserve the original source file
- report success or failure through the PyQt5 interface

---

## 1. Why DICOM Anonymization Matters

DICOM files can contain attributes such as:

- patient name
- patient ID
- birth date
- sex
- address
- telephone number
- accession number
- institution information
- referring physician
- performing physician
- operator name
- private vendor data

Even when the pixel data looks harmless, the accompanying metadata may identify a patient.

DICOM anonymization therefore requires more than changing the displayed patient name.

The dataset itself must be processed before it is shared.

---

## 2. Anonymization Workflow

The basic workflow in this project is:

```text
Original DICOM file
        ↓
Read with pydicom
        ↓
Create a deep copy
        ↓
Replace Patient Name and Patient ID
        ↓
Clear selected identifying fields
        ↓
Remove private tags
        ↓
Add de-identification indicators
        ↓
Save as a new DICOM file
```

The original file is not overwritten automatically.

This is an important safety decision because anonymization should not destroy the only available source file.

---

## 3. The Anonymizer Module

The anonymization logic is separated into `anonymizer.py`.

```python
from copy import deepcopy
from pathlib import Path

import pydicom
from pydicom.dataset import Dataset
```

The module provides two main functions:

| Function | Responsibility |
| --- | --- |
| `anonymize_dataset()` | Creates and anonymizes a dataset copy |
| `save_anonymized_dicom()` | Reads, anonymizes, and saves a DICOM file |

Keeping this logic outside the user-interface code makes it easier to test and reuse.

---

## 4. Identifying Fields

The project defines a list of metadata fields that should be cleared.

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

These keywords represent common DICOM attributes that may contain identifying or institution-related information.

Using a list avoids repeating the same removal logic for every field.

It also makes the basic anonymization policy visible and easy to extend.

---

## 5. Protecting the Original Dataset

The function starts by creating a deep copy.

```python
anonymized = deepcopy(dataset)
```

A normal assignment would not create an independent dataset:

```python
anonymized = dataset
```

Both names would refer to the same object.

Changing `anonymized` could then change the original in memory.

A deep copy creates an independent object hierarchy so that the anonymization function can modify the copy while preserving the loaded source dataset.

---

## 6. Replacing Patient Name and Patient ID

The two primary patient identifiers are replaced with a fixed value.

```python
anonymized.PatientName = "ANONYMOUS"
anonymized.PatientID = "ANONYMOUS"
```

Replacing the values instead of deleting them can make the anonymized result easier to inspect.

The output explicitly shows that anonymization was applied:

```text
Patient Name: ANONYMOUS
Patient ID:   ANONYMOUS
```

However, replacing these two fields alone is not sufficient because identifying information may exist elsewhere in the dataset.

---

## 7. Clearing Optional Identifying Fields

Each configured field is checked before modification.

```python
for field in IDENTIFYING_FIELDS:
    if hasattr(anonymized, field):
        setattr(anonymized, field, "")
```

`hasattr()` prevents an error when a DICOM file does not contain a particular attribute.

`setattr()` allows the field name to be supplied dynamically.

Conceptually:

```text
For every configured field:
    If the dataset contains it:
        Replace its value with an empty value
```

This approach handles DICOM datasets whose metadata contents differ.

---

## 8. Removing Private Tags

DICOM allows manufacturers and organizations to store private attributes.

These fields may contain:

- vendor-specific acquisition information
- internal identifiers
- device configuration
- processing parameters
- undocumented text values

The project removes private elements with:

```python
anonymized.remove_private_tags()
```

Private tags cannot automatically be assumed to be safe.

Removing them is a useful basic precaution when creating a de-identified copy.

It may also remove useful vendor-specific information, so production policies must determine which private elements may safely be retained.

---

## 9. Recording De-identification Status

The anonymized dataset records that identifying information was removed.

```python
anonymized.PatientIdentityRemoved = "YES"
anonymized.DeidentificationMethod = "Basic metadata anonymization"
```

These correspond to the concepts:

```text
Patient Identity Removed → de-identification was performed
De-identification Method → description of the applied method
```

This provides useful provenance for someone inspecting the output file.

The description deliberately says `Basic metadata anonymization` because this implementation is a learning example rather than a complete clinical de-identification system.

---

## 10. Complete Dataset Function

The complete dataset-level anonymization function is:

```python
def anonymize_dataset(dataset: Dataset) -> Dataset:
    """Return a copy with patient-identifying information and private tags removed."""
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

The function accepts a `pydicom.dataset.Dataset` and returns a new anonymized dataset.

It does not save the result by itself.

This keeps in-memory processing separate from file-system operations.

---

## 11. Saving an Anonymized DICOM File

The file-level function accepts input and output paths.

```python
def save_anonymized_dicom(input_path: str, output_path: str) -> Path:
    """Anonymize a DICOM file and save it as a new file."""
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

The processing sequence is:

1. validate the source path
2. read the source dataset
3. create an anonymized copy
4. create the destination directory when necessary
5. save the new DICOM file
6. return the saved path

---

## 12. Creating the Destination Directory

The output directory is created with:

```python
destination.parent.mkdir(parents=True, exist_ok=True)
```

The options mean:

```text
parents=True  → create missing parent directories
exist_ok=True → do not fail if the directory already exists
```

This makes the save function more robust when the selected destination directory does not yet exist.

---

## 13. Saving Without Overwriting the Loaded Dataset

The anonymized copy is saved with:

```python
anonymized.save_as(str(destination))
```

Because the function operates on a deep copy, the viewer's loaded dataset remains unchanged.

The separation is:

```text
Original dataset → remains loaded in the viewer
Anonymized copy  → saved to the selected output path
```

This allows the user to continue inspecting the original image and metadata after creating the anonymized file.

---

## 14. Adding the Anonymization Button

The PyQt5 viewer adds a new button.

```python
self.anonymize_button = QPushButton("Save Anonymized DICOM")
self.anonymize_button.setEnabled(False)
self.anonymize_button.clicked.connect(self.save_anonymized)
```

The button starts disabled.

It becomes enabled only after a DICOM file is loaded successfully:

```python
self.anonymize_button.setEnabled(True)
```

This prevents an invalid save operation before the application has a source file.

---

## 15. Remembering the Source Path

The viewer stores the path of the currently loaded DICOM file.

```python
self.current_file_path = None
```

After loading succeeds:

```python
self.current_file_path = file_path
```

The save operation needs this path so that it can:

- read the original file
- propose an output filename
- pass the source path to the anonymizer

---

## 16. Creating a Safe Default Filename

The default output path adds `_anonymized` to the source filename.

```python
source_path = Path(self.current_file_path)

default_output_path = source_path.with_name(
    f"{source_path.stem}_anonymized{source_path.suffix}"
)
```

Example:

```text
Input:  test_image.dcm
Output: test_image_anonymized.dcm
```

This naming pattern reduces the risk of accidentally overwriting the original file.

---

## 17. Selecting the Output Location

The viewer opens a save dialog.

```python
output_path, _ = QFileDialog.getSaveFileName(
    self,
    "Save Anonymized DICOM",
    str(default_output_path),
    "DICOM Files (*.dcm);;All Files (*)",
)
```

If the user cancels, the method returns without changing anything:

```python
if not output_path:
    return
```

If the filename has no extension, `.dcm` is added:

```python
if not Path(output_path).suffix:
    output_path += ".dcm"
```

---

## 18. Reporting the Result

After saving succeeds, the viewer displays the output path.

```python
QMessageBox.information(
    self,
    "Anonymization Complete",
    f"Anonymized DICOM saved:\n{saved_path}",
)
```

If an exception occurs, an error dialog is shown:

```python
QMessageBox.critical(
    self,
    "Anonymization Error",
    str(error),
)
```

The interface therefore provides clear feedback instead of failing silently.

---

## 19. Verifying the Output

An anonymized file should be reopened and checked.

A simple verification script can compare important fields:

```python
import pydicom

original = pydicom.dcmread("samples/test_image.dcm")
anonymized = pydicom.dcmread(
    "samples/test_image_anonymized.dcm"
)

print("Original Patient Name:", original.PatientName)
print("Anonymous Patient Name:", anonymized.PatientName)

print("Original Patient ID:", original.PatientID)
print("Anonymous Patient ID:", anonymized.PatientID)

print(
    "Patient Identity Removed:",
    anonymized.PatientIdentityRemoved,
)
print(
    "De-identification Method:",
    anonymized.DeidentificationMethod,
)
```

Expected anonymized values:

```text
Anonymous Patient Name: ANONYMOUS
Anonymous Patient ID: ANONYMOUS
Patient Identity Removed: YES
De-identification Method: Basic metadata anonymization
```

Verification is essential because a successfully saved file is not automatically proof that all required identifying data was removed.

---

## 20. Pixel Data Is Preserved

![DICOM pixel data preserved after metadata anonymization](/assets/images/posts/post-08/dicom-test-image.png)

*The grayscale Pixel Data remains available because this lesson changes selected metadata rather than the medical-image pixels.*

This basic implementation changes selected metadata but does not modify `PixelData`.

The medical image therefore remains available in the anonymized file.

Conceptually:

| Component | Result |
| --- | --- |
| Selected patient metadata | Replaced or cleared |
| Private tags | Removed |
| Pixel Data | Preserved |
| Original source file | Preserved |
| New anonymized file | Created |

However, some medical images may contain burned-in patient information inside the pixels themselves.

Metadata anonymization cannot remove text that is embedded directly in an image.

---

## 21. Important Limitations

This project demonstrates basic metadata anonymization for learning and software testing.

It is not a complete clinical de-identification solution.

A production system must also consider:

- identifying values in sequences
- dates and times
- UIDs
- burned-in annotations
- overlays
- structured reports
- private elements that require policy-based handling
- consistent pseudonyms across related studies
- institution-specific requirements
- applicable privacy laws and regulations
- formal DICOM confidentiality profiles

The output should not be treated as safe for real clinical sharing without an appropriate de-identification policy and independent validation.

---

## 22. Anonymization vs. Pseudonymization

Anonymization and pseudonymization are related but different.

| Method | General idea |
| --- | --- |
| Anonymization | Removes or transforms identifying information so re-identification is not reasonably possible |
| Pseudonymization | Replaces identifiers while retaining a controlled way to reconnect records |

This project uses a fixed `ANONYMOUS` value and does not provide a secure identity-mapping system.

It is therefore best described as a basic metadata de-identification exercise.

---

## 23. Why Modular Design Helps

The anonymization logic is independent of PyQt5.

This means it can later be reused in:

- command-line tools
- batch-processing scripts
- automated tests
- PACS import pipelines
- research-data preparation tools
- server applications

The interface only selects paths and reports results.

The actual DICOM transformation remains in `anonymizer.py`.

---

## What I Learned

In this lesson, I learned that:

- DICOM files may contain sensitive information in many attributes.
- changing Patient Name alone is not sufficient.
- `deepcopy()` protects the original in-memory dataset.
- `hasattr()` safely checks optional DICOM attributes.
- `setattr()` updates attributes selected dynamically.
- `remove_private_tags()` removes private DICOM elements.
- `PatientIdentityRemoved` records de-identification status.
- `DeidentificationMethod` describes the applied process.
- anonymized data should be saved as a new file.
- a disabled button prevents saving before a file is loaded.
- output files must be reopened and verified.
- metadata anonymization does not remove burned-in pixel text.
- production de-identification requires a formal policy and validation.

The most important idea from this lesson is:

> **DICOM anonymization must protect the original file, process more than one patient field, and verify the saved result.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 2 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/847cc26)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I built a basic DICOM viewer with Python, pydicom, NumPy, and PyQt5:

[Building a Basic DICOM Viewer with Python and PyQt5](/medical-imaging/dicom/2026/08/29/building-a-basic-dicom-viewer-with-python.html)

---

## Next Step

Now that the viewer can create an anonymized DICOM copy, the next step is to inspect more information stored in the dataset.

In the next lesson, we will explore:

- DICOM tags
- Value Representations
- metadata values
- displaying a complete metadata table
- separating image display from metadata inspection

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
