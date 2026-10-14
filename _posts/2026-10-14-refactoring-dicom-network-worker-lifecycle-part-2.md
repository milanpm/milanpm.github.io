---
layout: post
title: "Refactoring DICOM Query/Retrieve Worker Lifecycles in PyQt5 — Part 2"
date: 2026-10-14 23:50:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, python, pyqt5, qthread, refactoring, c-find, c-move, c-get]
description: "Apply a shared PyQt5 worker lifecycle to hierarchical C-FIND, C-MOVE, and C-GET, then verify retrieval, failure recovery, and cancellation followed by another operation."
---

In Part 1, I introduced `start_network_operation()` for C-ECHO and C-STORE. This session extended the same lifecycle to hierarchical C-FIND, C-MOVE, and C-GET.

The README records hierarchical C-FIND as Day 22 Part 2 and C-MOVE/C-GET as Part 3. This blog post covers both changes as the second article in the lifecycle refactoring series.

**Status:** Shared startup applied to the targeted operations; local integration checks completed within the scope described below.

**Learning date:** October 14, 2026.

## Learning Objectives

- Apply the shared worker startup method to Query/Retrieve operations.
- Preserve operation-specific validation and result handlers.
- Prevent another operation from starting before worker cleanup completes.
- Verify successful retrieval, association failure recovery, and cancellation followed by another operation.

## Keep Validation Close to Each Operation

The operations share a worker lifecycle, but their inputs differ.

| Operation | Required inputs |
|---|---|
| Study C-FIND | Study search criteria |
| Series C-FIND | Study Instance UID |
| Instance C-FIND | Study and Series Instance UIDs |
| Study C-MOVE/C-GET | Study Instance UID |
| Series C-MOVE/C-GET | Study and Series Instance UIDs |

Each GUI method continues to prepare and validate its inputs. The shared method handles worker creation, signal connections, control updates, and startup.

C-MOVE also keeps its local Storage SCP prerequisite. C-GET keeps its storage directory argument.

## Use the Worker Reference as the Guard

The updated operation methods check:

```python
if self.network_worker is not None:
    return
```

Checking only `isRunning()` leaves a gap: thread execution may have stopped while the GUI has not yet processed its completion handler.

Keeping the reference until cleanup finishes prevents another operation from replacing the tracked worker during that interval.

The shared startup method uses the same guard.

## Apply the Helper to Hierarchical C-FIND

Study, Series, and Instance searches now delegate startup to `start_network_operation()`.

Each search retains its own query arguments and result handler. Series and Instance searches also validate network settings before startup.

This separates two responsibilities:

- The search method defines the query and validates its inputs.
- The shared helper starts and tracks the worker.

## Apply the Helper to C-MOVE

After validating the Storage SCP prerequisite, connection settings, and required UIDs, C-MOVE starts through the shared helper:

```python
self.start_network_operation(
    operation_name,
    move_instances,
    result_handler=(
        lambda result, level=query_level:
        self.handle_move_result(result, level)
    ),
    enable_progress=True,
    enable_cancellation=True,
    local_ae_title=settings["local_ae_title"],
    remote_ae_title=settings["remote_ae_title"],
    remote_ip=settings["remote_ip"],
    remote_port=settings["remote_port"],
    move_destination_ae_title=settings["local_ae_title"],
    query_level=query_level,
    study_instance_uid=study_instance_uid,
    series_instance_uid=series_instance_uid,
)
```

The lambda captures the requested query level for the result handler. Progress reporting and cancellation remain enabled.

## Apply the Helper to C-GET

C-GET follows the same startup pattern while retaining its storage directory:

```python
storage_dir = Path.cwd() / "received"

self.start_network_operation(
    operation_name,
    get_instances,
    result_handler=(
        lambda result, level=query_level:
        self.handle_get_result(result, level)
    ),
    enable_progress=True,
    enable_cancellation=True,
    local_ae_title=settings["local_ae_title"],
    remote_ae_title=settings["remote_ae_title"],
    remote_ip=settings["remote_ip"],
    remote_port=settings["remote_port"],
    query_level=query_level,
    study_instance_uid=study_instance_uid,
    series_instance_uid=series_instance_uid,
    storage_dir=storage_dir,
)
```

After these changes, `NetworkWorker(` appears only once in `src/main.py`, inside the shared startup method.

## Successful Query and Retrieval Checks

The local test configuration used:

| Setting | Value |
|---|---|
| Local AE | PACS_TOOLKIT |
| Remote AE | TEST_PACS |
| Remote address | 127.0.0.1:11112 |
| Local Storage SCP port | 11113 |

The query returned one study with two series. The selected series contained three instances.

| Check | Observed result |
|---|---|
| C-ECHO | Success, 0x0000 |
| Study C-FIND | 1 study |
| Series C-FIND | 2 series |
| Instance C-FIND | 3 instances in the selected series |
| Study C-GET | 4 completed, 0 failed, 0 warning |
| Series C-GET | 3 completed, 0 failed, 0 warning |
| Study C-MOVE | 4 completed, 0 failed, 0 warning |
| Series C-MOVE | 3 completed, 0 failed, 0 warning |
| Missing Series Instance UID | Request blocked by validation |

The selected series returned three instances through C-FIND and completed three transfers through both retrieval methods.

Successive operations could start after the preceding operation ended.

## Association Failure Recovery

I stopped the test PACS, then ran Study C-GET and Study C-MOVE.

```text
[23:37:05] Study C-GET: Working...
[23:37:07] FAILED: DICOM Association failed or timed out.
[23:37:28] Study C-MOVE: Working...
[23:37:31] FAILED: DICOM Association failed or timed out.
```

After restarting the test PACS:

```text
[23:37:52] C-ECHO: Working...
[23:37:52] SUCCESS: C-ECHO Success: 0x0000
```

These checks demonstrate that a reported association failure did not leave the GUI unable to start another operation.

A failed association may show zero completed and zero failed sub-operations because transfer never started. Those counters alone do not indicate operation success.

## Manual Cancellation Checks

The four-instance dataset transferred too quickly for reliable manual cancellation.

I added a one-second test delay before each instance transfer in the C-MOVE and C-GET handlers, with cancellation checks before and after the delay:

```python
for instance in matching_instances:
    if event.is_cancelled:
        yield 0xFE00, None
        return

    # Test-only delay to allow manual cancellation.
    sleep(1.0)

    if event.is_cancelled:
        yield 0xFE00, None
        return

    # Existing logging remains here.
    yield 0xFF00, instance
```

The delay is part of the local test server and slows its retrieval responses.

The GUI logs showed:

```text
[23:43:54] C-GET Progress: 2 completed, 2 remaining, 0 failed, 0 warning
[23:43:54] Study C-GET: Cancellation requested...
[23:43:55] CANCELLED: Study C-GET cancelled
[23:44:00] C-ECHO: Working...
[23:44:00] SUCCESS: C-ECHO Success: 0x0000
[23:44:13] Study C-MOVE: Working...
[23:44:14] Study C-MOVE: Cancellation requested...
[23:44:15] CANCELLED: Study C-MOVE cancelled
[23:44:18] C-ECHO: Working...
[23:44:18] SUCCESS: C-ECHO Success: 0x0000
```

These results verify GUI cancellation reporting and the ability to start another operation after cleanup.

Server-side receipt and handling of C-CANCEL were not independently verified. The GUI logs do not establish exactly how many files the server sent before cancellation took effect.

## Validation Scope

Python syntax compilation and `git diff --check` passed. Git reported Windows line-ending conversion warnings.

The runtime checks covered the local test PACS and its small dataset. They do not establish behavior against every PACS implementation or under large-volume transfer.

C-STORE was refactored in Part 1 and was not separately retested during this session.

## Source Code

- [49bd4a4 — Refactor hierarchical C-FIND worker lifecycle](https://github.com/milanpm/PACS-DICOM-Toolkit/commit/49bd4a4)
- [7ffe873 — Refactor C-MOVE and C-GET worker lifecycle](https://github.com/milanpm/PACS-DICOM-Toolkit/commit/7ffe873)

## Next Step

Independently verify server-side cancellation handling and extend integration checks beyond the current small local dataset.

**Key takeaway:** Each operation keeps its validation, arguments, and result presentation. One shared method manages worker startup, while the completion handler clears the worker reference and restores the controls.

---

**Previous:** [Refactoring the C-ECHO and C-STORE Worker Lifecycle in PyQt5 — Part 1]({% post_url 2026-10-06-refactoring-dicom-network-worker-lifecycle-part-1 %})
