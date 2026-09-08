---
layout: post
title: "Receiving DICOM Files with a Storage SCP and pynetdicom"
date: 2026-09-08 14:30:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, python, pydicom, pynetdicom, c-store, storage-scp]
description: "Learn how to build a DICOM Storage SCP with pynetdicom, handle incoming C-STORE requests, save received datasets, and verify the stored files with pydicom."
---

In the previous post, we sent a DICOM file from a C-STORE Service Class User (SCU).

A sender alone, however, cannot complete a DICOM storage workflow. A receiving application must listen for associations, negotiate supported storage services, process C-STORE requests, and save the received datasets.

In this post, we will build a **Storage Service Class Provider (SCP)** with `pynetdicom`. The server will receive DICOM objects, save them using their SOP Instance UIDs, and return a success or failure status to the sender.

## C-STORE SCU and Storage SCP

C-STORE communication involves two applications.

| Component | Role |
|---|---|
| C-STORE SCU | Requests storage and sends a DICOM object |
| Storage SCP | Accepts the request and stores the received object |

The communication flow is:

1. The SCU requests an association.
2. The SCU and SCP negotiate presentation contexts.
3. The SCU sends a C-STORE request containing a DICOM dataset.
4. The SCP processes and saves the dataset.
5. The SCP returns a C-STORE status.
6. The association is released.

```text
C-STORE SCU
     |
     | Association request
     | C-STORE request
     | DICOM dataset
     v
Storage SCP
     |
     | Save dataset
     | Return status 0x0000
     v
received/<SOPInstanceUID>.dcm
```

## Storage SCP Configuration

The Storage SCP in this example uses the following settings:

| Setting | Value |
|---|---|
| AE Title | `STORAGE_SCP` |
| Host | `127.0.0.1` |
| Port | `11113` |
| Storage directory | `received/` |

The server and sender run on the same computer for this test. In an external network, the sender must use the receiver's reachable IP address and an allowed TCP port.

## Handling Incoming C-STORE Requests

The Storage SCP processes incoming objects through an event handler.

```python
def handle_store(event, storage_dir):
    try:
        dataset = event.dataset
        dataset.file_meta = event.file_meta

        storage_dir.mkdir(parents=True, exist_ok=True)

        output_path = (
            storage_dir
            / f"{dataset.SOPInstanceUID}.dcm"
        )

        dataset.save_as(
            output_path,
            enforce_file_format=True,
        )

        print(f"Received DICOM: {output_path}")
        return 0x0000

    except Exception as exc:
        print(f"Failed to store DICOM: {exc}")
        return 0xC211
```

The event object provides the received dataset and its file meta information:

```python
dataset = event.dataset
dataset.file_meta = event.file_meta
```

`event.dataset` contains the main DICOM dataset. `event.file_meta` contains file-format information derived from the negotiated presentation context, including the transfer syntax.

Attaching the file meta information before saving allows the received object to be reopened as a standard DICOM file.

## Saving with the SOP Instance UID

Each DICOM instance is identified by a unique SOP Instance UID.

```python
output_path = storage_dir / f"{dataset.SOPInstanceUID}.dcm"
```

The resulting directory has the following form:

```text
received/
├── 1.2.826.0.1.3680043.8.498.3001.dcm
├── 1.2.826.0.1.3680043.8.498.3002.dcm
└── 1.2.826.0.1.3680043.8.498.4001.dcm
```

Each received object is stored with a stable, unique filename.

A production system should define how duplicate SOP Instance UIDs are handled.

## Starting the Storage SCP

The server requires an Application Entity, supported presentation contexts, and a C-STORE event handler.

```python
from pathlib import Path

from pynetdicom import AE, AllStoragePresentationContexts, evt
```

The server registers the supported storage SOP Classes and connects `EVT_C_STORE` to the handler.

```python
def start_storage_scp(ae_title, host, port, storage_dir):
    storage_dir = Path(storage_dir)
    ae = AE(ae_title=ae_title)

    for context in AllStoragePresentationContexts:
        ae.add_supported_context(context.abstract_syntax)

    handlers = [
        (evt.EVT_C_STORE, handle_store, [storage_dir])
    ]

    return ae.start_server(
        (host, port),
        block=False,
        evt_handlers=handlers,
    )
```

Using `block=False` allows the GUI event loop to continue while the Storage SCP listens for incoming associations. The returned server can be stopped later with:

```python
server.shutdown()
```

## Running the Transfer Test

The Storage SCP was started at `127.0.0.1:11113` with the AE title `STORAGE_SCP`. A separate C-STORE SCU then sent a test DICOM object.

```text
Received DICOM:
received/1.2.826.0.1.3680043.8.498.19812736760175547305782898469238143179.dcm
```

The handler returned `0x0000`, indicating that the C-STORE operation completed successfully.

## Verifying the Received File

The stored object was reopened with `pydicom` to verify that it remained a readable DICOM file.

```python
from pathlib import Path

import pydicom


files = list(Path("received").glob("*.dcm"))

if not files:
    raise SystemExit("No received DICOM files found.")

path = max(files, key=lambda item: item.stat().st_mtime)
dataset = pydicom.dcmread(path)

print("File:", path)
print("Patient ID:", getattr(dataset, "PatientID", "N/A"))
print("Patient Name:", getattr(dataset, "PatientName", "N/A"))
print("Modality:", getattr(dataset, "Modality", "N/A"))
print("SOP Instance UID:", getattr(dataset, "SOPInstanceUID", "N/A"))
print("Transfer Syntax UID:", dataset.file_meta.TransferSyntaxUID)
print("Has Pixel Data:", "PixelData" in dataset)
```

The verification result was:

```text
Patient ID: TEST001
Patient Name: TEST^PATIENT
Modality: OT
Transfer Syntax UID: 1.2.840.10008.1.2
Has Pixel Data: True
Validation: PASS
```

The transfer syntax UID `1.2.840.10008.1.2` represents **Implicit VR Little Endian**. The file was successfully reopened, and its pixel data was preserved.

## Missing Study and Series UIDs

The synthetic test object did not contain `StudyInstanceUID` or `SeriesInstanceUID`. These attributes were absent from the source dataset; the Storage SCP did not remove them.

A typical clinical DICOM hierarchy is:

```text
StudyInstanceUID
└── SeriesInstanceUID
    └── SOPInstanceUID
```

Production systems should validate required attributes according to the applicable DICOM Information Object Definition.

## C-STORE Status Codes

| Status | Example | Meaning |
|---|---:|---|
| Success | `0x0000` | The object was stored successfully |
| Warning | `0xB000` | The object was stored with a warning |
| Failure | `0xA700` | Storage resources were unavailable |
| Failure | `0xA900` | The dataset did not match the requested SOP Class |
| Failure | `0xCxxx` | An implementation-specific processing error occurred |

The example returns success only after the dataset has been saved. If an exception occurs, it returns `0xC211`.

## Practical Considerations

A production Storage SCP requires more than writing datasets to a directory. Important considerations include:

- validating SOP Class and SOP Instance UIDs
- defining a duplicate-instance policy
- checking available storage capacity
- organizing objects by patient, study, and series
- recording association and storage logs
- protecting patient information
- restricting accepted AE titles and network addresses
- backing up stored objects
- registering instances in a database

The implementation in this post intentionally focuses on the essential receiving workflow.

## What We Learned

In this post, we:

- created a DICOM Storage SCP with `pynetdicom`
- registered storage presentation contexts
- handled incoming requests with `EVT_C_STORE`
- attached file meta information to the received dataset
- used the SOP Instance UID as the output filename
- returned C-STORE success and failure statuses
- ran the server without blocking the GUI
- reopened the received file with `pydicom`
- confirmed that the pixel data was preserved

A C-STORE SCU and Storage SCP now form a complete DICOM file-transfer path.

## Source Code

The complete project source code is available on GitHub:

[PACS-DICOM-Toolkit](https://github.com/milanpm/PACS-DICOM-Toolkit)

---


## Next Step

The next step is to use **C-FIND** to query a remote PACS for DICOM studies.

This will move the project from basic connectivity and storage operations to DICOM Query/Retrieve.

---

**Previous:** [Sending DICOM Files with C-STORE and pynetdicom]({% post_url 2026-09-04-sending-dicom-files-with-c-store-and-pynetdicom %})
