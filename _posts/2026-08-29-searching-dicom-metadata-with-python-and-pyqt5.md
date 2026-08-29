---
layout: post
title: "Searching DICOM Metadata with Python and PyQt5"
date: 2026-08-29 22:10:00 +0900
categories: [medical-imaging, dicom]
tags: [Python, DICOM, PACS, pydicom, Metadata Search, PyQt5, Medical Imaging]
---

# Searching DICOM Metadata with Python and PyQt5

A DICOM dataset may contain dozens or hundreds of data elements.

Displaying a small metadata summary is useful, but developers and technical users often need to find a particular attribute quickly.

In this lesson, I added a metadata search feature to the Python DICOM viewer.

The viewer can now:

- accept a search term through a PyQt5 text field
- iterate through the loaded DICOM dataset
- search by human-readable tag name
- search by DICOM keyword
- ignore uppercase and lowercase differences
- display every matching tag and value
- report empty searches and zero-result searches
- prevent searching before a DICOM file is loaded

---

## 1. Why Search DICOM Metadata?

A DICOM dataset contains much more information than can fit in a small summary panel.

Examples include:

- patient attributes
- study attributes
- series attributes
- acquisition parameters
- equipment information
- image geometry
- display settings
- SOP identifiers
- pixel representation
- private elements

Finding one attribute manually becomes difficult as the dataset grows.

A search field makes metadata inspection faster and more practical.

---

## 2. Tag Name and Keyword

Each pydicom `DataElement` provides several useful properties.

```python
element.tag
element.name
element.keyword
element.value
```

Example:

```text
Tag:     (0010,0010)
Name:    Patient's Name
Keyword: PatientName
Value:   TEST^PATIENT
```

The name is intended to be readable.

The keyword is useful in Python code and DICOM documentation.

Searching both gives the user more flexibility.

---

## 3. Search Examples

The viewer performs partial text matching.

Example searches include:

| Search term | Possible matches |
| --- | --- |
| `patient` | Patient's Name, Patient ID |
| `study` | Study Date, Study Time, Study Instance UID |
| `window` | Window Center, Window Width |
| `rows` | Rows |
| `modality` | Modality |
| `SOP` | SOP Class UID, SOP Instance UID |

The search is case-insensitive.

The following inputs therefore behave the same way:

```text
patient
Patient
PATIENT
```

---

## 4. Building the Search Interface

The viewer adds a `QLineEdit` for user input.

```python
self.metadata_search_input = QLineEdit()
self.metadata_search_input.setPlaceholderText(
    "Enter tag name or keyword"
)
```

The placeholder explains what the user can enter.

A search button is connected to the search method:

```python
self.metadata_search_button = QPushButton("Search")
self.metadata_search_button.clicked.connect(
    self.search_metadata
)
```

The result area uses a read-only `QTextEdit`:

```python
self.metadata_search_result = QTextEdit()
self.metadata_search_result.setReadOnly(True)
self.metadata_search_result.setMinimumHeight(120)
```

`QTextEdit` is useful because a search may return multiple lines.

---

## 5. Arranging the Search Controls

The input field and button are placed in a horizontal layout.

```python
search_layout = QHBoxLayout()
search_layout.addWidget(self.metadata_search_input)
search_layout.addWidget(self.metadata_search_button)
```

The search section is then added below the summary metadata.

```python
control_layout.addWidget(QLabel("Metadata Search"))
control_layout.addLayout(search_layout)
control_layout.addWidget(self.metadata_search_result)
```

The interface now provides both:

```text
Selected metadata summary
        +
Searchable dataset inspection
```

---

## 6. Preventing Search Before Loading

The search method first checks whether a dataset is available.

```python
if self.dataset is None:
    self.metadata_search_result.setText(
        "Please open a DICOM file first."
    )
    return
```

Without this guard, the application would attempt to iterate over `None`.

The message tells the user exactly what action is required.

---

## 7. Reading and Normalizing the Search Term

The input text is normalized with:

```python
keyword = self.metadata_search_input.text().strip().lower()
```

Three operations occur:

```text
text()  → read the QLineEdit contents
strip() → remove leading and trailing whitespace
lower() → convert text to lowercase
```

For example:

```text
"  Patient  "
        ↓
"patient"
```

This makes the search more forgiving.

---

## 8. Handling an Empty Search

After trimming whitespace, the method checks whether anything remains.

```python
if not keyword:
    self.metadata_search_result.setText(
        "Please enter a search keyword."
    )
    return
```

This handles:

- an empty input
- a string containing only spaces
- a cleared search field

An explicit message is more useful than returning an unexplained empty result.

---

## 9. Iterating Through the Dataset

A pydicom `Dataset` can be iterated directly.

```python
for element in self.dataset:
```

Each iteration returns a `DataElement`.

The viewer can then inspect:

```python
element.tag
element.name
element.keyword
element.value
```

The current implementation searches the top-level elements contained directly in the dataset.

---

## 10. Normalizing DICOM Names and Keywords

The element name and keyword are converted to lowercase.

```python
tag_name = element.name.lower()
tag_keyword = element.keyword.lower()
```

The user's normalized input can then be compared without case sensitivity.

Example:

```text
Input:           patient
Element name:    patient's name
Element keyword: patientname
```

Both strings contain the search term.

---

## 11. Partial-Match Search

The matching condition is:

```python
if keyword in tag_name or keyword in tag_keyword:
```

This uses substring matching.

The search term does not need to equal the complete name or keyword.

Examples:

```text
"patient" matches "Patient's Name"
"patient" matches "PatientName"
"window"  matches "Window Center"
"window"  matches "WindowWidth"
```

Partial matching is convenient when the exact DICOM keyword is unknown.

---

## 12. Formatting Results

Each match is formatted with:

```python
results.append(
    f"{element.tag} {element.name}: {element.value}"
)
```

A result may look like:

```text
(0010,0010) Patient's Name: TEST^PATIENT
```

The output contains:

1. the DICOM tag number
2. the human-readable element name
3. the stored value

This is more informative than displaying only the value.

---

## 13. Displaying Multiple Matches

Matches are collected in a list.

```python
results = []
```

When matches exist, they are joined with newline characters:

```python
self.metadata_search_result.setText(
    "\n".join(results)
)
```

Example:

```text
(0010,0010) Patient's Name: TEST^PATIENT
(0010,0020) Patient ID: TEST001
```

A multi-line result is easy to inspect in the read-only text area.

---

## 14. Handling Zero Results

If the list remains empty, the viewer displays:

```python
self.metadata_search_result.setText(
    "No matching metadata found."
)
```

A zero-result search is a normal application state.

It does not mean the DICOM file failed to load.

It only means that no top-level element name or keyword contains the supplied text.

---

## 15. Complete Search Method

The complete implementation is:

```python
def search_metadata(self):
    """Search DICOM metadata by tag name or keyword."""
    if self.dataset is None:
        self.metadata_search_result.setText(
            "Please open a DICOM file first."
        )
        return

    keyword = (
        self.metadata_search_input
        .text()
        .strip()
        .lower()
    )

    if not keyword:
        self.metadata_search_result.setText(
            "Please enter a search keyword."
        )
        return

    results = []

    for element in self.dataset:
        tag_name = element.name.lower()
        tag_keyword = element.keyword.lower()

        if keyword in tag_name or keyword in tag_keyword:
            results.append(
                f"{element.tag} "
                f"{element.name}: "
                f"{element.value}"
            )

    if results:
        self.metadata_search_result.setText(
            "\n".join(results)
        )
    else:
        self.metadata_search_result.setText(
            "No matching metadata found."
        )
```

The method combines input validation, dataset traversal, filtering, formatting, and result display.

---

## 16. Search Processing Flow

The search pipeline is:

```text
User enters a term
        ↓
Trim whitespace
        ↓
Convert to lowercase
        ↓
Check that a dataset is loaded
        ↓
Iterate through Data Elements
        ↓
Compare name and keyword
        ↓
Collect matching results
        ↓
Display matches or zero-result message
```

Each validation condition returns early, keeping the main search loop simple.

---

## 17. Searching the Test Dataset

The generated test DICOM contains fields such as:

```text
Patient Name
Patient ID
Modality
Study Date
Study Time
Rows
Columns
Window Center
Window Width
SOP Class UID
SOP Instance UID
```

A search for:

```text
window
```

should find:

```text
Window Center
Window Width
```

A search for:

```text
patient
```

should find:

```text
Patient's Name
Patient ID
```

The exact formatting of numerical and UID values is handled by pydicom.

---

## 18. Data-Privacy Consideration

Metadata search can expose patient-identifying values.

For example, searching for `patient` may display:

- Patient Name
- Patient ID
- other patient-related fields

A production viewer should consider:

- user authorization
- audit logging
- screen privacy
- whether sensitive values should be masked
- whether an anonymized dataset should be used
- whether results may be copied or exported

A convenient search feature must still respect the sensitivity of medical information.

---

## 19. Current Limitation: Top-Level Search

The loop:

```python
for element in self.dataset:
```

searches elements directly contained in the top-level dataset.

DICOM Sequences may contain nested datasets.

The current version does not recursively traverse those nested items.

A later recursive implementation could inspect:

```text
Dataset
└── Sequence element
    ├── Item 1 dataset
    │   └── Nested elements
    └── Item 2 dataset
        └── Nested elements
```

Sequence traversal requires care because DICOM metadata can contain multiple nesting levels.

---

## 20. Current Limitation: Large Values

The result currently includes:

```python
element.value
```

This is suitable for ordinary text and numerical metadata.

It is not suitable for every DICOM value.

Searching for `pixel` could match `PixelData` and attempt to display a very large byte value.

A safer future formatter should:

- replace Pixel Data with a summary
- truncate long text
- summarize binary values
- limit the number of displayed characters
- identify sequences without expanding everything

Example safe output:

```text
(7FE0,0010) Pixel Data: <524288 bytes>
```

This would keep the user interface responsive and readable.

---

## 21. Possible Search Improvements

The search feature can later support:

- pressing Enter to search
- searching tag numbers such as `0010,0010`
- searching values
- exact-match mode
- regular expressions
- recursive sequence traversal
- result highlighting
- result counts
- type-aware value formatting
- exclusion of Pixel Data
- exporting search results
- searching only selected tag groups
- clearing results when a new file opens

These improvements can turn the basic search box into a complete DICOM metadata browser.

---

## 22. Why This Feature Matters for PACS Development

Metadata search is useful when developing and troubleshooting:

- PACS viewers
- DICOM import pipelines
- modality integrations
- anonymization tools
- storage systems
- query/retrieve workflows
- image-processing applications
- conformance tests

It helps answer practical questions quickly:

```text
Does this file contain Pixel Spacing?
Which SOP Class does it use?
Is Window Center present?
What is the Study Instance UID?
Which modality created the image?
```

This is especially useful when different devices produce datasets with different optional fields.

---

## What I Learned

In this lesson, I learned that:

- a DICOM Dataset can be iterated as Data Elements.
- each element provides a tag, name, keyword, and value.
- tag names are human-readable.
- DICOM keywords are useful for programming.
- `strip()` removes unnecessary input whitespace.
- `lower()` enables case-insensitive matching.
- the `in` operator provides partial-text matching.
- a list can collect multiple metadata results.
- `QTextEdit` can display multi-line results.
- empty and zero-result searches should be handled explicitly.
- top-level iteration does not recursively search Sequence items.
- large binary values should not be printed directly.
- metadata search must consider patient privacy.

The most important idea from this lesson is:

> **A useful DICOM metadata search must combine flexible matching with safe handling of missing, nested, sensitive, and potentially large values.**

---

## Source Code

The source code for this lesson is available on GitHub:

[View the Day 4 source code on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit/tree/4cbab9d)

Current project repository:

[View PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Previous Step

In the previous lesson, I separated metadata extraction from the PyQt5 interface:

[Displaying DICOM Metadata with Python and pydicom](/medical-imaging/dicom/2026/08/29/displaying-dicom-metadata-with-python-and-pydicom.html)

---

## Next Step

Now that the viewer can search DICOM metadata, the next step is to export the displayed medical image to a common image format.

In the next lesson, we will explore:

- applying the current Window Center and Window Width
- converting DICOM pixels to 8-bit grayscale
- creating a Pillow image
- selecting a PNG output path
- saving the displayed result as a PNG file

---

*This post is part of my journey in medical imaging, DICOM, and PACS software development.*
