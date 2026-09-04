---
layout: post
title: "Sending DICOM Files with C-STORE and pynetdicom"
date: 2026-09-04 23:20:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, pynetdicom, python, c-store, storage-scu, pyqt5]
description: "Learn how to send a DICOM file to a Storage SCP with C-STORE, negotiate the required presentation context, interpret the response status, and verify the received object."
---

In the previous post, we used **C-ECHO** to verify that two DICOM Application Entities could communicate.

C-ECHO confirms connectivity, but it does not transfer medical images. To send a DICOM object to a PACS or another DICOM application, we use the **C-STORE service**.

In this post, we will:

- understand the roles of a Storage SCU and Storage SCP
- read and validate a DICOM file before transmission
- request a presentation context for the file's SOP Class
- establish a DICOM Association
- send the dataset with `send_c_store()`
- interpret the returned status code
- run the operation without blocking the PyQt5 GUI
- verify that the received object matches the original

## C-ECHO vs. C-STORE

C-ECHO and C-STORE are both DIMSE services, but they answer different questions.

| Service | Purpose | Transfers a DICOM object? |
|---|---|---:|
| C-ECHO | Is the remote DICOM application reachable? | No |
| C-STORE | Can this DICOM object be transferred and processed? | Yes |

A successful C-ECHO does not guarantee a successful C-STORE. The remote application must also accept a suitable Storage SOP Class and Transfer Syntax.

## Storage SCU and Storage SCP

C-STORE communication has two roles.

| Role | Responsibility | This test |
|---|---|---|
| Storage SCU | Sends a DICOM object | PACS-DICOM-Toolkit |
| Storage SCP | Receives and processes the object | TEST_PACS |

The communication flow is:

```text
PACS-DICOM-Toolkit
    Storage SCU
        |
        | Association Request
        v
TEST_PACS
    Storage SCP
        |
        | Association Accept
        v
PACS-DICOM-Toolkit
        |
        | C-STORE-RQ + DICOM Dataset
        v
TEST_PACS
        |
        | Save DICOM File
        | C-STORE-RSP: 0x0000
        v
PACS-DICOM-Toolkit
```

## DICOM Network Configuration

The local test used the following settings:

| Setting | Value |
|---|---|
| Local AE Title | `PACS_TOOLKIT` |
| Remote AE Title | `TEST_PACS` |
| Remote IP | `127.0.0.1` |
| Remote Port | `11112` |

`PACS_TOOLKIT` acts as the Storage SCU. The local test server, `TEST_PACS`, acts as the Storage SCP.

## Implementing the C-STORE SCU

The networking logic is implemented in `src/dicom_network.py`.

```python
def send_dicom_file(
    file_path,
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
):
    """Send a DICOM file to a remote Storage SCP using C-STORE."""
    association = None

    try:
        local_ae_title = local_ae_title.strip()
        remote_ae_title = remote_ae_title.strip()
        remote_ip = remote_ip.strip()
        remote_port = int(remote_port)

        dataset = dcmread(file_path)

        if not hasattr(dataset, "SOPClassUID"):
            return False, "DICOM file does not contain SOPClassUID."

        if not hasattr(dataset, "SOPInstanceUID"):
            return False, "DICOM file does not contain SOPInstanceUID."

        ae = AE(ae_title=local_ae_title)

        ae.add_requested_context(dataset.SOPClassUID)

        ae.acse_timeout = 5
        ae.dimse_timeout = 10
        ae.network_timeout = 10

        association = ae.associate(
            remote_ip,
            remote_port,
            ae_title=remote_ae_title,
        )

        if not association.is_established:
            return False, "DICOM Association failed."

        status = association.send_c_store(dataset)

        if not status or not hasattr(status, "Status"):
            return False, "No valid C-STORE response was received."

        status_code = int(status.Status)

        if status_code == 0x0000:
            return True, "C-STORE Success: 0x0000"

        return False, f"C-STORE Failed: 0x{status_code:04X}"

    except (OSError, TypeError, ValueError) as error:
        return False, f"C-STORE error: {error}"

    finally:
        if association is not None and association.is_established:
            association.release()
```

Let us examine the important parts of this function.

## Reading the DICOM Dataset

The selected file is loaded with `pydicom`:

```python
dataset = dcmread(file_path)
```

The complete dataset is required because C-STORE transmits the DICOM object, including its pixel data.

## Validating the SOP UIDs

Before creating the Association, the function checks two identifiers:

```python
if not hasattr(dataset, "SOPClassUID"):
    return False, "DICOM file does not contain SOPClassUID."

if not hasattr(dataset, "SOPInstanceUID"):
    return False, "DICOM file does not contain SOPInstanceUID."
```

Their purposes are different:

| Identifier | Meaning |
|---|---|
| SOP Class UID | The type of DICOM object, such as a CT image or Secondary Capture image |
| SOP Instance UID | The unique identity of one specific DICOM object |

The SOP Class determines which Storage service must be negotiated. The SOP Instance UID identifies the individual object being transferred.

## Requesting a Presentation Context

The Application Entity is created with the configured local AE Title:

```python
ae = AE(ae_title=local_ae_title)
```

The SCU then requests a presentation context for the selected file's SOP Class:

```python
ae.add_requested_context(dataset.SOPClassUID)
```

This is more precise than requesting a fixed Storage SOP Class because the request is derived from the actual DICOM file.

During Association negotiation, the SCP decides whether it supports the requested Abstract Syntax and one of the proposed Transfer Syntaxes. C-STORE cannot be sent unless a compatible presentation context is accepted.

The current implementation uses the default Transfer Syntax list supplied by `pynetdicom`. Supporting compressed datasets such as JPEG or JPEG 2000 may require explicitly requesting their Transfer Syntax or transcoding the dataset.

## Configuring Network Timeouts

The application sets separate timeouts for the major network layers:

```python
ae.acse_timeout = 5
ae.dimse_timeout = 10
ae.network_timeout = 10
```

| Timeout | Purpose |
|---|---|
| ACSE timeout | Limits Association negotiation time |
| DIMSE timeout | Limits the wait for a DIMSE response |
| Network timeout | Limits low-level network inactivity |

Timeouts prevent the application from waiting indefinitely when the remote system is unavailable or stops responding.

## Establishing the Association

The SCU connects to the remote AE:

```python
association = ae.associate(
    remote_ip,
    remote_port,
    ae_title=remote_ae_title,
)
```

Before sending the file, we verify that the Association was established:

```python
if not association.is_established:
    return False, "DICOM Association failed."
```

An Association failure can be caused by an incorrect IP address, port, AE Title, firewall rule, or incompatible presentation context configuration.

## Sending the DICOM Object

The actual transfer is performed with one call:

```python
status = association.send_c_store(dataset)
```

`send_c_store()` sends a C-STORE request containing the dataset and waits for the C-STORE response from the SCP.

The response is checked before reading its status value:

```python
if not status or not hasattr(status, "Status"):
    return False, "No valid C-STORE response was received."
```

## Interpreting the Response Status

The returned status is converted to an integer:

```python
status_code = int(status.Status)
```

A status of `0x0000` means Success:

```python
if status_code == 0x0000:
    return True, "C-STORE Success: 0x0000"
```

Other values are displayed as failures:

```python
return False, f"C-STORE Failed: 0x{status_code:04X}"
```

Common C-STORE status categories include:

| Category | Meaning |
|---|---|
| Success | The SCP successfully processed the request |
| Warning | The object was stored, but a non-fatal condition occurred |
| Failure | The SCP could not process or store the object |
| Cancel | Not used as a normal final response for C-STORE |

The current implementation treats only `0x0000` as success. A future improvement could classify warning and failure ranges separately and display their descriptions.

## Releasing the Association

The Association is released in a `finally` block:

```python
finally:
    if association is not None and association.is_established:
        association.release()
```

This guarantees cleanup after either success or an exception.

## Running C-STORE Outside the GUI Thread

The GUI collects the connection settings and starts the operation with `NetworkWorker`:

```python
self.network_worker = NetworkWorker(
    "C-STORE",
    send_dicom_file,
    file_path,
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
)
```

The C-STORE button is disabled while the operation is running:

```python
self.active_network_button = self.store_button
self.store_button.setEnabled(False)
```

Running network operations in a worker thread prevents Association negotiation and DIMSE timeouts from freezing the PyQt5 event loop.

## Preparing the Test Storage SCP

The project includes `tools/find_scp.py`, which provides C-ECHO, C-STORE, C-FIND, C-MOVE, and C-GET services for local testing.

It is started with:

```bash
python tools/find_scp.py
```

The test server reports:

```text
Starting Query/Retrieve SCP
Services: C-ECHO, C-STORE, C-FIND, C-MOVE, C-GET
AE Title: TEST_PACS
Port: 11112
Move Destination: PACS_TOOLKIT -> 127.0.0.1:11113
```

For this test, the SCP explicitly supports Secondary Capture Image Storage:

```python
ae.add_supported_context(
    SecondaryCaptureImageStorage,
    scu_role=False,
    scp_role=True,
)
```

Therefore, we opened a compatible Secondary Capture DICOM object before clicking **C-STORE**.

## C-STORE Test Result

The Network tab displayed the successful response:

```text
Network: C-STORE Success: 0x0000
```

![Successful C-STORE transmission in PACS-DICOM-Toolkit](/assets/images/posts/post-13-dicom-c-store/c-store-success.png)

This result confirms that:

1. the Storage Association was established
2. a compatible presentation context was accepted
3. the DICOM dataset was transmitted
4. the SCP processed the request
5. the SCP returned a successful C-STORE response

## Saving the Received Dataset

The test SCP handles the incoming C-STORE request using `EVT_C_STORE`:

```python
def handle_store(event):
    """Handle an incoming C-STORE request."""
    try:
        dataset = event.dataset
        dataset.file_meta = event.file_meta

        project_dir = Path(__file__).resolve().parents[1]
        storage_dir = project_dir / "received" / "test_pacs"
        storage_dir.mkdir(parents=True, exist_ok=True)

        sop_instance_uid = str(dataset.SOPInstanceUID).strip()
        output_path = storage_dir / f"{sop_instance_uid}.dcm"

        dataset.save_as(
            output_path,
            enforce_file_format=True,
        )

        return 0x0000

    except Exception:
        return 0xC001
```

The received file is named with its SOP Instance UID and saved under `received/test_pacs/`.

The successful test created:

```text
received/test_pacs/
1.2.826.0.1.3680043.8.498.19812736760175547305782898469238143179.dcm
```

Using the SOP Instance UID as the filename preserves the object's DICOM identity and reduces arbitrary filename collisions.

## Verifying the Received Object

A successful response is important, but we also compared the original and received datasets.

The verification checked:

- SOP Class UID
- SOP Instance UID
- image dimensions
- SHA-256 digest of Pixel Data

The result was:

```text
Same SOP Class:     True
Same SOP Instance:  True
Same dimensions:    True
Same pixel data:    True
```

This confirms that the same DICOM instance was received and that its image data was unchanged.

We did not require the SHA-256 hash of the complete files to match. Rewriting the File Meta Information can change file-level bytes even when the DICOM identity and pixel data remain the same.

## Troubleshooting C-STORE

### Association Failed

Check the following settings:

- remote IP address
- remote port
- remote AE Title
- whether the Storage SCP is running
- local firewall rules

### No Accepted Presentation Context

Confirm that the SCP supports the dataset's SOP Class and Transfer Syntax. For example, a server configured only for Secondary Capture Image Storage will not automatically accept CT Image Storage.

### No Valid Response

The Association may have timed out, been aborted, or returned an invalid DIMSE response. Check both the SCU status message and the SCP log.

### Compressed DICOM Transfer Fails

Inspect `dataset.file_meta.TransferSyntaxUID`. Compressed data may require the matching Transfer Syntax to be proposed and accepted, or the dataset may need to be decompressed before transmission.

## What We Learned

In this post, we learned how to:

- distinguish C-ECHO connectivity from C-STORE transfer
- use Storage SCU and Storage SCP roles correctly
- validate SOP Class UID and SOP Instance UID
- request a presentation context based on the selected dataset
- establish and release a DICOM Association
- send a dataset with `association.send_c_store()`
- interpret the `0x0000` success response
- execute C-STORE in a background worker
- save an incoming object using its SOP Instance UID
- verify DICOM identity, dimensions, and Pixel Data after transfer

C-STORE is the foundation for transferring images between modalities, workstations, gateways, archives, and PACS servers.

## Source Code

The complete project is available on GitHub:

[PACS-DICOM-Toolkit](https://github.com/milanpm/PACS-DICOM-Toolkit)

The C-STORE SCU implementation is located in:

[src/dicom_network.py](https://github.com/milanpm/PACS-DICOM-Toolkit/blob/main/src/dicom_network.py)

The local test Storage SCP is located in:

[tools/find_scp.py](https://github.com/milanpm/PACS-DICOM-Toolkit/blob/main/tools/find_scp.py)

## Previous Step

**Previous:** [Verifying DICOM Connectivity with C-ECHO and pynetdicom]({% post_url 2026-09-03-verifying-dicom-connectivity-with-c-echo-and-pynetdicom %})

## Next Step

In the next post, we will reverse the direction and run PACS-DICOM-Toolkit as a Storage SCP that receives and saves DICOM objects.

**Next:** Receiving DICOM Files with a Storage SCP and pynetdicom
