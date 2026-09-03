---
layout: post
title: "Verifying DICOM Connectivity with C-ECHO and pynetdicom"
date: 2026-09-03 22:50:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, python, pynetdicom, c-echo, networking, medical-imaging]
description: "Learn how to verify DICOM network connectivity by establishing an Association and sending a C-ECHO request with Python and pynetdicom."
---

After refactoring our Python DICOM viewer into focused modules, we now have a clean foundation for adding PACS networking features.

The first network operation we will implement is **C-ECHO**. It verifies whether two DICOM Application Entities can establish an Association and exchange a Verification request and response.

C-ECHO is often compared to the standard network `ping` command, but it checks more than basic IP connectivity. A successful C-ECHO confirms that the remote port is reachable, the DICOM Association is accepted, and the Verification SOP Class can be negotiated.

In this post, we will:

- learn the roles of SCU and SCP
- configure AE Titles, an IP address, and a port
- request the Verification Presentation Context
- establish and release a DICOM Association
- send a C-ECHO request with `pynetdicom`
- connect the network function to a PyQt5 interface
- interpret a successful `0x0000` response
- inspect the Association and DIMSE logs

---

## What C-ECHO Does

C-ECHO is part of the DICOM Verification Service Class. It is used to determine whether a remote DICOM Application Entity is available and able to communicate using the DICOM protocol.

A successful test verifies several layers:

1. The remote IP address can be reached.
2. The configured TCP port is listening.
3. The remote Application Entity accepts the Association.
4. The Verification SOP Class is supported.
5. A C-ECHO request and response can be exchanged.

This makes C-ECHO a useful first diagnostic step before testing storage, query, or retrieval operations.

---

## SCU and SCP Roles

DICOM network services use two fundamental roles:

| Role | Full name | Responsibility |
| --- | --- | --- |
| SCU | Service Class User | Requests a DICOM service |
| SCP | Service Class Provider | Accepts and processes the request |

In our test environment, the PACS DICOM Toolkit sends the request and therefore acts as the **Verification SCU**. The `echoscp` application receives the request and acts as the **Verification SCP**.

```text
PACS_TOOLKIT                         ANY-SCP
Verification SCU                Verification SCP
      |                                |
      |------ Association Request ---->|
      |<----- Association Accepted ----|
      |--------- C-ECHO Request ------>|
      |<-------- C-ECHO Response ------|
      |------ Association Release ---->|
```

---

## DICOM Network Configuration

The local test uses the following configuration:

| Setting | Value | Purpose |
| --- | --- | --- |
| Local AE Title | `PACS_TOOLKIT` | Identifies the requesting application |
| Remote AE Title | `ANY-SCP` | Identifies the receiving application |
| Remote IP | `127.0.0.1` | Uses the local computer for testing |
| Remote Port | `11112` | TCP port used by the Verification SCP |

An **AE Title** identifies a DICOM Application Entity at the application layer. It is not a hostname, IP address, or user account.

The IP address identifies the computer, while the port identifies the listening network service on that computer. All three values must be configured correctly for a DICOM Association to succeed.

---

## Installing pynetdicom

The networking implementation uses `pynetdicom`, a Python library that provides DICOM networking services.

It is added to `requirements.txt`:

```text
PyQt5
pydicom
numpy
Pillow
pynetdicom
```

The project test was performed with:

```text
pynetdicom 3.0.4
```

---

## Implementing the Verification SCU

A new module named `dicom_network.py` contains the DICOM networking logic.

```python
from pynetdicom import AE
from pynetdicom.sop_class import Verification


def verify_connection(
    local_ae_title,
    remote_ae_title,
    remote_ip,
    remote_port,
):
    """Send a DICOM C-ECHO request to a remote Application Entity."""
    association = None

    try:
        local_ae_title = local_ae_title.strip()
        remote_ae_title = remote_ae_title.strip()
        remote_ip = remote_ip.strip()
        remote_port = int(remote_port)

        ae = AE(ae_title=local_ae_title)
        ae.add_requested_context(Verification)

        ae.acse_timeout = 5
        ae.dimse_timeout = 5
        ae.network_timeout = 5

        association = ae.associate(
            remote_ip,
            remote_port,
            ae_title=remote_ae_title,
        )

        if not association.is_established:
            return False, "DICOM Association failed."

        status = association.send_c_echo()

        if not status or not hasattr(status, "Status"):
            return False, "No valid C-ECHO response was received."

        status_code = int(status.Status)

        if status_code == 0x0000:
            return True, "C-ECHO Success: 0x0000"

        return False, f"C-ECHO Failed: 0x{status_code:04X}"

    except (OSError, TypeError, ValueError) as error:
        return False, f"Network error: {error}"

    finally:
        if association is not None and association.is_established:
            association.release()
```

---

## Creating the Application Entity

The local AE Title is assigned when the `AE` object is created:

```python
ae = AE(ae_title=local_ae_title)
```

The application must then request a Presentation Context for the Verification SOP Class:

```python
ae.add_requested_context(Verification)
```

A Presentation Context describes the DICOM service and transfer syntax that the requesting application wants to use. If the remote SCP does not accept a suitable context, the C-ECHO operation cannot proceed.

---

## Configuring Timeouts

Three five-second timeouts prevent the interface from waiting indefinitely when a remote service does not respond:

```python
ae.acse_timeout = 5
ae.dimse_timeout = 5
ae.network_timeout = 5
```

| Timeout | Purpose |
| --- | --- |
| ACSE timeout | Limits Association-control operations |
| DIMSE timeout | Limits DICOM message operations such as C-ECHO |
| Network timeout | Limits low-level network waiting |

These values are appropriate for a local learning environment. Production systems may require different values depending on network conditions and operational requirements.

---

## Establishing the Association

The SCU requests an Association with the remote Application Entity:

```python
association = ae.associate(
    remote_ip,
    remote_port,
    ae_title=remote_ae_title,
)
```

Before sending a DIMSE command, the program confirms that the Association was established:

```python
if not association.is_established:
    return False, "DICOM Association failed."
```

An Association failure is different from a failed C-ECHO response. If the Association cannot be established, no C-ECHO request is sent and no DIMSE status code is received.

---

## Sending C-ECHO and Reading the Status

Once the Association is established, the request is sent:

```python
status = association.send_c_echo()
```

The returned status object should contain a `Status` field:

```python
if not status or not hasattr(status, "Status"):
    return False, "No valid C-ECHO response was received."
```

The success status is `0x0000`:

```python
status_code = int(status.Status)

if status_code == 0x0000:
    return True, "C-ECHO Success: 0x0000"
```

Formatting the value as four hexadecimal digits makes DICOM status codes easier to recognize:

```python
return False, f"C-ECHO Failed: 0x{status_code:04X}"
```

---

## Releasing the Association Safely

The Association is released in a `finally` block:

```python
finally:
    if association is not None and association.is_established:
        association.release()
```

This ensures that an established Association is released even when a response is invalid or an exception occurs after the connection has been created.

---

## Connecting C-ECHO to the PyQt5 Interface

The main window imports the network function:

```python
from dicom_network import verify_connection
```

The Network interface contains fields for the local and remote configuration:

```python
self.local_ae_input = QLineEdit("PACS_TOOLKIT")
self.local_ae_input.setMaxLength(16)

self.remote_ae_input = QLineEdit("ANY-SCP")
self.remote_ae_input.setMaxLength(16)

self.remote_ip_input = QLineEdit("127.0.0.1")

self.remote_port_spin = QSpinBox()
self.remote_port_spin.setRange(1, 65535)
self.remote_port_spin.setValue(11112)

self.echo_button = QPushButton("Send C-ECHO")
self.network_status_label = QLabel("Network: Not tested")
```

The button is connected to the UI handler:

```python
self.echo_button.clicked.connect(self.send_c_echo)
```

The handler disables the button during the operation, calls `verify_connection()`, and displays the returned result. A successful result is shown in green, while a failed result is shown in red.

---

## Starting a Verification SCP

The `echoscp` command supplied with `pynetdicom` provides a convenient local Verification SCP.

The basic command starts the service but produces little terminal output:

```bash
echoscp 11112 -aet ANY-SCP
```

For concise runtime messages, use verbose mode:

```bash
echoscp -v 11112 -aet ANY-SCP
```

For detailed Association and DIMSE logs, use debug mode:

```bash
echoscp -d 11112 -aet ANY-SCP
```

In the tested `pynetdicom 3.0.4` environment, the installed `echoscp` command is used directly.

---

## Test Result

The application was configured with `PACS_TOOLKIT` as the local AE, `ANY-SCP` as the remote AE, `127.0.0.1` as the remote IP, and port `11112`.

After selecting **Send C-ECHO**, the interface displayed:

```text
Network: C-ECHO Success: 0x0000
```

![Successful DICOM C-ECHO verification](/assets/images/posts/post-12-dicom-c-echo/dicom-c-echo-success.png)

The green status confirms that the Association was established and a successful C-ECHO response was received.

---

## Reading the Debug Log

The debug log provides more detail about the protocol exchange.

Important entries included:

```text
Calling Application Name:    PACS_TOOLKIT
Called Application Name:     ANY-SCP
Abstract Syntax:              Verification SOP Class
Context ID:                   1 (Accepted)
Accepted Transfer Syntax:     Implicit VR Little Endian
Accepting Association
Received Echo Request (MsgID 1)
Message Type:                 C-ECHO RQ
Data Set:                     None
Association Released
```

![pynetdicom C-ECHO Association and DIMSE debug log](/assets/images/posts/post-12-dicom-c-echo/dicom-c-echo-terminal-log.png)

The log shows that Presentation Context ID 1 was accepted for the Verification SOP Class. Although several transfer syntaxes were proposed, the Association selected **Implicit VR Little Endian**.

The C-ECHO request contains no dataset because it is a verification command rather than a storage operation.

---

## Troubleshooting C-ECHO

| Symptom | Likely cause |
| --- | --- |
| Association failed | SCP is not running, IP or port is incorrect, or a firewall blocks the connection |
| Association rejected | Called AE Title is incorrect or the SCP rejects the request |
| No valid response | Connection was interrupted or the peer did not return a valid status |
| C-ECHO failed with a status code | The Association succeeded, but the DIMSE operation did not complete successfully |
| No terminal log appears | `echoscp` was started without `-v` or `-d` |

When troubleshooting, first confirm that the SCP is listening on the expected port. Then verify the Calling and Called AE Titles and inspect the Association negotiation log.

---

## What We Learned

In this step, we learned that:

- C-ECHO verifies DICOM application-level connectivity.
- The SCU requests a service and the SCP provides it.
- AE Titles identify DICOM Application Entities.
- IP addresses and ports identify the network endpoint.
- A Presentation Context must be accepted before C-ECHO can be sent.
- `0x0000` indicates a successful C-ECHO response.
- Association failure and DIMSE failure occur at different stages.
- Debug logs reveal the negotiated SOP Class and transfer syntax.
- Associations should be released even when an error occurs.

C-ECHO gives us a reliable foundation for the next network operation: transferring a DICOM dataset with C-STORE.

---

## Source Code

The implementation used in this post is available in the PACS-DICOM-Toolkit repository:

- [PACS-DICOM-Toolkit on GitHub](https://github.com/milanpm/PACS-DICOM-Toolkit)
- [C-ECHO implementation commit](https://github.com/milanpm/PACS-DICOM-Toolkit/commit/2fa4aeb06dee10fd556a4de1d8fcd92e03dae619)

---

## Previous Step

In the previous post, we separated the growing viewer into focused modules for DICOM loading, windowing, anonymization, image interaction, and UI coordination.

**Previous: [Refactoring a Python DICOM Viewer into Modular Components](/medical-imaging/2026/09/01/refactoring-a-python-dicom-viewer-into-modular-components.html)**

---

## Next Step

In the next post, we will extend `dicom_network.py` with Storage SCU functionality and send a DICOM file to a Storage SCP using C-STORE.

**Next: Sending DICOM Files with C-STORE and pynetdicom**

---

> **Disclaimer:** This project is intended for education and local testing. Production medical systems require appropriate security, privacy, validation, access control, audit logging, and regulatory compliance.
