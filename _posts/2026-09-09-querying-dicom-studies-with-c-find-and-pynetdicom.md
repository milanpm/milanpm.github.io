---
layout: post
title: "Querying DICOM Studies with C-FIND and pynetdicom"
date: 2026-09-09 23:30:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, pynetdicom, python, c-find, query-retrieve, pyqt5]
description: "Learn how to query DICOM studies with C-FIND and pynetdicom, distinguish matching keys from return keys, process pending responses, and display study results in a PyQt5 application."
---

In the previous post, we built a Storage SCP that could receive and save DICOM files.

Storage is only one part of a PACS workflow. Before retrieving an image, a DICOM application must first discover which patients, studies, series, and instances are available on the remote system.

In this post, we will use **C-FIND** and `pynetdicom` to query studies from a DICOM Query/Retrieve SCP. We will build a Study Root query, distinguish matching keys from return keys, process multiple responses, and display the results without blocking the PyQt5 interface.

## What C-FIND Does

C-FIND searches DICOM metadata. It does not transfer image pixel data.

| Service | Purpose |
|---|---|
| C-ECHO | Verify DICOM connectivity |
| C-STORE | Transfer and store a DICOM object |
| C-FIND | Search patient, study, series, or instance metadata |
| C-MOVE / C-GET | Retrieve selected DICOM objects |

A C-FIND SCU sends an Identifier dataset containing the query level, search conditions, and requested result fields. The SCP returns zero or more pending responses followed by a final response.

```text
C-FIND SCU                         C-FIND SCP
     |                                  |
     | Association request              |
     |--------------------------------->|
     | C-FIND request + Identifier      |
     |--------------------------------->|
     | Pending + result dataset         |
     |<---------------------------------|
     | Pending + result dataset         |
     |<---------------------------------|
     | Final success                    |
     |<---------------------------------|
     | Association release              |
     |--------------------------------->|
```

## Study Root Query/Retrieve Model

The Study Root information model organizes searchable objects into three levels:

```text
STUDY
└── SERIES
    └── IMAGE
```

This example searches at the `STUDY` level:

```python
query = Dataset()
query.QueryRetrieveLevel = "STUDY"
```

The requested presentation context must use the same information model:

```python
from pynetdicom.sop_class import (
    StudyRootQueryRetrieveInformationModelFind,
)

ae.add_requested_context(
    StudyRootQueryRetrieveInformationModelFind
)
```

If the SCU and SCP do not support a compatible Query/Retrieve model, the required presentation context will not be accepted.

## Matching Keys and Return Keys

The C-FIND Identifier contains two important types of keys.

### Matching Keys

A key with a supplied value is used as a search condition.

```python
query.PatientID = "TEST001"
query.PatientName = "TEST^*"
query.StudyDate = "20260828"
```

DICOM also supports matching patterns such as a patient-name wildcard or date range, depending on the SCP implementation.

```python
query.PatientName = "TEST^*"
query.StudyDate = "20260801-20260831"
```

### Return Keys

An empty value requests that the SCP include the corresponding attribute in each result.

```python
query.StudyInstanceUID = ""
query.StudyDescription = ""
query.AccessionNumber = ""
query.ModalitiesInStudy = ""
```

In this project, the patient and date fields come from the GUI. When a field is empty, it can also act as a requested return key.

```python
query.PatientID = patient_id.strip()
query.PatientName = patient_name.strip()
query.StudyDate = study_date.strip()
```

The exact matching behavior depends on the PACS implementation. A production integration should always be checked against the server's DICOM Conformance Statement.

## Implementing the Study Query

The following function creates the Application Entity, establishes the association, sends the C-FIND request, and collects the returned studies:

```python
def find_studies(
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
    patient_id="",
    patient_name="",
    study_date="",
):
    """Search studies from a remote PACS using DICOM C-FIND."""
    association = None

    try:
        local_ae_title = local_ae_title.strip()
        remote_ae_title = remote_ae_title.strip()
        remote_ip = remote_ip.strip()
        remote_port = int(remote_port)

        ae = AE(ae_title=local_ae_title)
        ae.add_requested_context(
            StudyRootQueryRetrieveInformationModelFind
        )

        ae.acse_timeout = 5
        ae.dimse_timeout = 10
        ae.network_timeout = 10

        association = ae.associate(
            remote_ip,
            remote_port,
            ae_title=remote_ae_title,
        )

        if not association.is_established:
            return False, [], "DICOM Association failed."

        query = Dataset()
        query.QueryRetrieveLevel = "STUDY"

        query.PatientID = patient_id.strip()
        query.PatientName = patient_name.strip()
        query.StudyDate = study_date.strip()

        query.StudyInstanceUID = ""
        query.StudyDescription = ""
        query.AccessionNumber = ""
        query.ModalitiesInStudy = ""

        results = []

        responses = association.send_c_find(
            query,
            StudyRootQueryRetrieveInformationModelFind,
        )

        for status, identifier in responses:
            if status is None:
                return (
                    False,
                    results,
                    "No valid C-FIND response was received.",
                )

            status_code = int(status.Status)

            if status_code in (0xFF00, 0xFF01):
                if identifier is not None:
                    results.append(identifier)
            elif status_code == 0x0000:
                break
            else:
                return (
                    False,
                    results,
                    f"C-FIND Failed: 0x{status_code:04X}",
                )

        return (
            True,
            results,
            f"C-FIND completed: {len(results)} study(s) found.",
        )

    except (OSError, TypeError, ValueError) as error:
        return False, [], f"C-FIND error: {error}"

    finally:
        if association is not None and association.is_established:
            association.release()
```

The `finally` block ensures that an established association is released even when an error or early return occurs.

## Processing C-FIND Responses

C-FIND can produce several responses for one request.

| Status | Meaning |
|---|---|
| `0xFF00` | Pending: a matching result was returned |
| `0xFF01` | Pending: optional-key warning with a result |
| `0x0000` | Success: the search is complete |
| `0xFE00` | Cancel: the request was cancelled |
| Other failure status | The query failed |

Each pending response may contain one matching Identifier. The final success response is a completion signal and does not represent another study.

```python
if status_code in (0xFF00, 0xFF01):
    if identifier is not None:
        results.append(identifier)
elif status_code == 0x0000:
    break
```

## Running C-FIND Without Blocking PyQt5

DICOM network operations are blocking, so running C-FIND directly in a button handler could freeze the GUI. The toolkit passes `find_studies()` to a reusable `QThread` worker:

```python
self.network_worker = NetworkWorker(
    "Study C-FIND",
    find_studies,
    local_ae_title=self.local_ae_input.text(),
    remote_ae_title=self.remote_ae_input.text(),
    remote_ip=self.remote_ip_input.text(),
    remote_port=self.remote_port_spin.value(),
    patient_id=self.find_patient_id_input.text(),
    patient_name=self.find_patient_name_input.text(),
    study_date=self.find_study_date_input.text(),
)

self.network_worker.progress.connect(
    self.update_network_progress
)
self.network_worker.result.connect(
    self.handle_study_find_result
)
self.network_worker.error.connect(
    self.handle_network_error
)
self.network_worker.finished.connect(
    self.finish_network_operation
)

self.network_worker.start()
```

This keeps the interface responsive while the association and query are in progress.

## Displaying the Results Safely

A PACS may omit an optional attribute. The result handler therefore uses `getattr()` with an empty default instead of assuming that every element exists.

```python
patient_id = getattr(dataset, "PatientID", "")
patient_name = getattr(dataset, "PatientName", "")
study_date = getattr(dataset, "StudyDate", "")
study_uid = getattr(dataset, "StudyInstanceUID", "")
```

The Study Instance UID is especially important because it becomes the unique selection key for later Series C-FIND, C-MOVE, and C-GET operations.

## Running the Query Test

The test Query/Retrieve SCP was started with these settings:

| Setting | Value |
|---|---|
| Local AE Title | `PACS_TOOLKIT` |
| Remote AE Title | `TEST_PACS` |
| Remote IP | `127.0.0.1` |
| Remote Port | `11112` |
| Query Level | `STUDY` |

All search fields were left empty so that the SCU requested all available studies and asked the SCP to return their attributes.

The application log confirmed successful asynchronous execution:

```text
Study C-FIND: Working...
SUCCESS: C-FIND completed: 1 study(s) found.
```

The same query was executed three times, and each run returned the same study successfully.

## Query Result

The SCP returned the following dataset:

```text
Study 1

Patient ID: TEST001
Patient Name: TEST^PATIENT
Study Date: 20260828
Study Description: Test Study
Accession Number: ACC001
Modalities: CT
Study Instance UID: 1.2.826.0.1.3680043.8.498.1001
```

The SCP log also confirmed that the request contained the expected Study Root Identifier:

```text
(0008,0052) Query/Retrieve Level    CS: 'STUDY'
(0010,0010) Patient's Name         PN: ''
(0010,0020) Patient ID             LO: ''
(0020,000D) Study Instance UID     UI: ''
```

The empty values show that these elements were requested as return keys in the unrestricted test query.

## Practical Considerations

- Validate AE titles, IP addresses, ports, and date formats before starting the worker.
- Set association and DIMSE timeouts so an unavailable PACS does not leave the application waiting indefinitely.
- Do not assume that every PACS returns every optional key.
- Treat the Study Instance UID as the stable identifier for downstream Query/Retrieve operations.
- Consult the PACS DICOM Conformance Statement before relying on wildcard, range, fuzzy, or optional-key behavior.
- Apply access controls and audit logging when querying real clinical systems.

## What We Learned

In this post, we:

- used the Study Root Query/Retrieve Information Model
- created a Study-level C-FIND Identifier
- distinguished matching keys from return keys
- handled pending and final C-FIND responses
- released the association safely
- ran the blocking query in a background `QThread`
- displayed optional DICOM attributes safely
- verified the request and result against a test Query/Retrieve SCP

C-FIND now allows the toolkit to discover studies before selecting a study for more detailed queries or image retrieval.

## Source Code

The complete project source code is available on GitHub:

[PACS-DICOM-Toolkit](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

The next step is to query the **Series** and **Instance** levels for a selected Study Instance UID.

This will extend the Query/Retrieve workflow from a study list to the individual DICOM objects available for retrieval.

---

**Previous:** [Receiving DICOM Files with a Storage SCP and pynetdicom]({% post_url 2026-09-08-receiving-dicom-files-with-a-storage-scp-and-pynetdicom %})
