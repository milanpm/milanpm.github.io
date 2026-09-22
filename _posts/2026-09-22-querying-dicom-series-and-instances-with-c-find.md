---
layout: post
title: "Querying DICOM Series and Instances with C-FIND"
date: 2026-09-22 23:10:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, pynetdicom, python, c-find, query-retrieve, pyqt5]
description: "Learn how to extend a Study-level DICOM C-FIND workflow to query Series and individual SOP Instances with pynetdicom, hierarchical matching keys, and PyQt5."
---

In the previous post, we used C-FIND to query studies from a DICOM Query/Retrieve SCP.

A Study-level result is only the beginning of the PACS navigation workflow. Before retrieving images, the application must identify the Series contained in the selected Study and then discover the individual SOP Instances contained in a selected Series.

In this post, we will extend the C-FIND implementation to the `SERIES` and `IMAGE` Query/Retrieve levels. We will use hierarchical matching keys, request Series and Instance attributes, process multiple pending responses, and integrate the workflow into the PyQt5 Network tab.

## Hierarchical Query/Retrieve Workflow

The Study Root Query/Retrieve model follows this hierarchy:

```text
STUDY
└── SERIES
    └── IMAGE
```

Each query result supplies the unique identifier required by the next level.

```text
Study C-FIND
    |
    | StudyInstanceUID
    v
Series C-FIND
    |
    | StudyInstanceUID
    | SeriesInstanceUID
    v
Image-level C-FIND
    |
    | SOPClassUID
    | SOPInstanceUID
    | InstanceNumber
    v
Individual DICOM Instances
```

Although the application searches for individual Instances, the standard DICOM Query/Retrieve level is named `IMAGE`.

## Matching Keys and Return Keys

A C-FIND Identifier contains matching keys and return keys.

A matching key has a value and restricts the search:

```python
query.StudyInstanceUID = study_instance_uid
```

A return key has an empty value and asks the SCP to include that attribute in each matching response:

```python
query.SeriesInstanceUID = ""
query.SeriesNumber = ""
query.SeriesDescription = ""
```

The keys change as the application moves down the hierarchy.

| Query Level | Matching Keys | Important Return Keys |
|---|---|---|
| Study | Patient ID, Patient Name, Study Date | Study Instance UID |
| Series | Study Instance UID | Series Instance UID, Series Number, Modality |
| Image | Study Instance UID, Series Instance UID | SOP Class UID, SOP Instance UID, Instance Number |

The Study Instance UID identifies the parent Study. The Series Instance UID identifies one Series inside that Study. The SOP Instance UID identifies one DICOM object inside the selected Series.

## Implementing the Series Query

The Series query receives the Study Instance UID selected from the Study-level results.

```python
def find_series(
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
    study_instance_uid,
    cancel_callback=None,
):
    """Search series in a study from a remote PACS using DICOM C-FIND."""
```

The function first normalizes and validates the Study Instance UID.

```python
study_instance_uid = study_instance_uid.strip()

if not study_instance_uid:
    return False, [], "Study Instance UID is required."
```

The Application Entity requests the Study Root C-FIND presentation context and establishes an association with the remote PACS.

```python
ae = AE(ae_title=local_ae_title)
ae.add_requested_context(
    StudyRootQueryRetrieveInformationModelFind
)

association = ae.associate(
    remote_ip,
    remote_port,
    ae_title=remote_ae_title,
)
```

The Identifier selects the `SERIES` level and uses the Study Instance UID as a matching key.

```python
query = Dataset()
query.QueryRetrieveLevel = "SERIES"

# Matching key
query.StudyInstanceUID = study_instance_uid
```

The empty attributes are return keys requested from the SCP.

```python
# Return keys
query.SeriesInstanceUID = ""
query.SeriesNumber = ""
query.SeriesDescription = ""
query.Modality = ""
query.NumberOfSeriesRelatedInstances = ""
```

These values allow the GUI to display a useful Series list and supply the Series Instance UID required by the next query.

## Implementing the Instance Query

The Instance query requires both the parent Study Instance UID and the selected Series Instance UID.

```python
def find_instances(
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
    study_instance_uid,
    series_instance_uid,
    cancel_callback=None,
):
    """Search instances in a series using DICOM C-FIND."""
```

Both UIDs are validated before the association is opened.

```python
if not study_instance_uid:
    return False, [], "Study Instance UID is required."

if not series_instance_uid:
    return False, [], "Series Instance UID is required."
```

The DICOM query uses the `IMAGE` level.

```python
query = Dataset()
query.QueryRetrieveLevel = "IMAGE"

# Matching keys
query.StudyInstanceUID = study_instance_uid
query.SeriesInstanceUID = series_instance_uid
```

The Instance-specific attributes are requested as return keys.

```python
# Return keys
query.SOPClassUID = ""
query.SOPInstanceUID = ""
query.InstanceNumber = ""
```

The SOP Class UID identifies the type of DICOM object, while the SOP Instance UID uniquely identifies the individual object.

## Processing Multiple C-FIND Responses

Series and Instance queries can return multiple matches. `send_c_find()` therefore returns an iterator of status and Identifier pairs.

```python
responses = association.send_c_find(
    query,
    StudyRootQueryRetrieveInformationModelFind,
)

for status, identifier in responses:
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
```

The important response statuses are:

| Status | Meaning |
|---|---|
| `0xFF00` | Pending response containing a matching Identifier |
| `0xFF01` | Pending response with optional-key limitations |
| `0x0000` | Final success response |

Each pending Identifier is appended to the result list. The final success response indicates that the SCP has finished sending matches.

The association is released in a `finally` block so that cleanup also occurs after an error or early return.

```python
finally:
    if association is not None and association.is_established:
        association.release()
```

## Integrating the Hierarchy into PyQt5

The Network tab provides three related actions:

1. Search for Studies.
2. Search for Series using a Study Instance UID.
3. Search for Instances using Study and Series Instance UIDs.

The selected identifiers flow through the interface as follows:

```text
Study result
    -> Study Instance UID field
    -> Series result
    -> Series Instance UID field
    -> Instance result
```

When a Study query returns exactly one result, the application automatically copies its Study Instance UID into the Series query field.

The same behavior is available when a Series query returns exactly one result. If multiple Series are returned, the user selects or enters the required Series Instance UID before starting the Instance query.

The network operation runs in a background worker so the PyQt5 interface remains responsive while waiting for C-FIND responses.

## Test Query/Retrieve SCP

The local test SCP used the following connection settings:

| Setting | Value |
|---|---|
| Local AE Title | `PACS_TOOLKIT` |
| Remote AE Title | `TEST_PACS` |
| Remote IP | `127.0.0.1` |
| Remote Port | `11112` |
| Information Model | Study Root Query/Retrieve |

The test hierarchy contained one Study, two Series, and four Instances.

```text
Test Study
├── CT Axial
│   ├── Instance 1
│   ├── Instance 2
│   └── Instance 3
└── CT Scout
    └── Instance 1
```

## Series Query Result

The Series C-FIND operation returned two results.

```text
Series 1

Series Number: 1
Description: CT Axial
Modality: CT
Instances: 3
Series Instance UID: 1.2.826.0.1.3680043.8.498.2001

Series 2

Series Number: 2
Description: CT Scout
Modality: CT
Instances: 1
Series Instance UID: 1.2.826.0.1.3680043.8.498.2002
```

The response sequence consisted of two pending responses followed by final success.

```text
0xFF00 + CT Axial Identifier
0xFF00 + CT Scout Identifier
0x0000 + Final Success
```

## Instance Query Results

The CT Axial Series returned three Instances.

```text
Instance 1

Instance Number: 1
SOP Class UID: 1.2.840.10008.5.1.4.1.1.2
SOP Instance UID: 1.2.826.0.1.3680043.8.498.3001

Instance 2

Instance Number: 2
SOP Class UID: 1.2.840.10008.5.1.4.1.1.2
SOP Instance UID: 1.2.826.0.1.3680043.8.498.3002

Instance 3

Instance Number: 3
SOP Class UID: 1.2.840.10008.5.1.4.1.1.2
SOP Instance UID: 1.2.826.0.1.3680043.8.498.3003
```

The CT Scout Series returned one Instance.

```text
Instance 1

Instance Number: 1
SOP Class UID: 1.2.840.10008.5.1.4.1.1.2
SOP Instance UID: 1.2.826.0.1.3680043.8.498.4001
```

The SOP Class UID `1.2.840.10008.5.1.4.1.1.2` represents CT Image Storage.

## Handling a Zero-Result Query

An unknown Series Instance UID was also tested:

```text
1.2.826.0.1.3680043.8.498.9999
```

The query completed successfully with no matching Instances.

```text
SUCCESS: C-FIND completed: 0 instance(s) found.
No matching instances found.
```

A zero-result query is not a network or DIMSE failure. The SCP accepted and processed the request, found no matching objects, and returned final success without any pending result datasets.

```text
C-FIND request
    -> No pending responses
    -> 0x0000 Final Success
    -> Zero results
```

This distinction is important in production applications because "no match" should be presented to the user differently from an association failure, timeout, or C-FIND error status.

## Practical Considerations

- Treat Study, Series, and SOP Instance UIDs as identifiers rather than display labels.
- Validate the required parent UID before issuing a lower-level query.
- Do not assume that every PACS returns every optional Series or Instance attribute.
- Expect multiple pending responses and one final response.
- Handle a successful zero-result query separately from a failed operation.
- Set association, ACSE, DIMSE, and network timeouts.
- Keep blocking DICOM operations outside the GUI thread.
- Consult the PACS DICOM Conformance Statement for supported query keys and matching behavior.
- Apply authentication, authorization, audit logging, and patient-data safeguards in clinical environments.

## What We Learned

In this post, we:

- extended Study-level C-FIND to the `SERIES` and `IMAGE` levels
- used the Study Instance UID as the Series-level matching key
- used Study and Series Instance UIDs as Image-level matching keys
- requested Series and Instance attributes with empty return keys
- processed multiple pending responses and final success
- displayed two Series and four SOP Instances in PyQt5
- distinguished a successful zero-result query from an operation failure
- completed the hierarchical query path from Study to individual DICOM objects

The toolkit can now discover the exact Series and SOP Instances that should be retrieved from a PACS.

## Source Code

The complete project source code is available on GitHub:

[PACS-DICOM-Toolkit](https://github.com/milanpm/PACS-DICOM-Toolkit)

---

## Next Step

The next step is to retrieve the DICOM objects selected through the hierarchical query workflow using **C-MOVE**.

We will use the Study and Series Instance UIDs returned by C-FIND, configure a Storage SCP as the move destination, and verify that the requested DICOM files are received successfully.

---

**Previous:** [Querying DICOM Studies with C-FIND and pynetdicom]({% post_url 2026-09-09-querying-dicom-studies-with-c-find-and-pynetdicom %})
