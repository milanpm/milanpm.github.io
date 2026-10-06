---
layout: post
title: "Refactoring the C-ECHO and C-STORE Worker Lifecycle in PyQt5 — Part 1"
date: 2026-10-06 15:35:00 +0900
categories: [medical-imaging]
tags: [dicom, pacs, python, pyqt5, qthread, refactoring, c-echo, c-store]
description: "Centralize C-ECHO and C-STORE worker startup in PyQt5 and verify cleanup, control recovery, and repeated execution after association failure."
---

In Day 21, I added cooperative cancellation and network timeouts to the PACS-DICOM-Toolkit. In Day 22 Part 1, I focused on making the worker lifecycle easier to maintain by sharing the startup code used by C-ECHO and C-STORE.

**Status:** Part 1 completed; the full Day 22 refactoring is still in progress.
**Learning date:** October 6, 2026.

## Learning Objectives

- Separate operation-specific validation from worker startup.
- Centralize worker creation, signal connections, and UI state changes.
- Distinguish an operation result from thread completion.
- Verify that a failed operation can finish cleanly and allow another operation to start.

## Why Refactor the Lifecycle?

Each network operation repeated a similar sequence: disable controls, create a NetworkWorker, connect its signals, update the status label, and start the thread. Maintaining these steps in several methods made changes harder to apply consistently.

I extracted this shared sequence into `start_network_operation()`. The operation methods now specify what to run and how to display its result.

## A Shared Startup Method

```python
    def start_network_operation(
        self,
        operation_name,
        operation,
        *args,
        result_handler,
        enable_progress=False,
        enable_cancellation=True,
        **kwargs,
    ):
        """Create, connect, and start one network worker."""
        # Wait until the previous worker's cleanup is complete.
        if self.network_worker is not None:
            return False

        worker = NetworkWorker(
            operation_name,
            operation,
            *args,
            enable_progress=enable_progress,
            enable_cancellation=enable_cancellation,
            **kwargs,
        )

        worker.progress.connect(self.update_network_progress)
        worker.result.connect(result_handler)
        worker.cancelled.connect(self.handle_network_cancelled)
        worker.error.connect(self.handle_network_error)
        worker.finished.connect(self.finish_network_operation)

        self.network_worker = worker

        self.set_network_controls_enabled(False)
        self.network_status_label.setStyleSheet("")
        self.network_status_label.setText(
            f"Network: Starting {operation_name}..."
        )

        worker.start()
        return True
```

`operation` is the network function to execute. `*args` and `**kwargs` pass its arguments to the worker. `result_handler` identifies the GUI method that receives the operation result. For C-ECHO, this is `handle_echo_result`; for C-STORE, it is `handle_store_result`.

The startup guard checks whether a worker reference still exists. This prevents a new operation from starting before the previous worker's GUI cleanup has been processed, even if its thread has already stopped running.

## Applying the Method to C-ECHO and C-STORE

After validating the network settings, C-ECHO starts its worker through the shared method:

```python
        self.start_network_operation(
            "C-ECHO",
            verify_connection,
            settings["local_ae_title"],
            settings["remote_ae_title"],
            settings["remote_ip"],
            settings["remote_port"],
            result_handler=self.handle_echo_result,
        )
```

C-STORE also checks that a DICOM file is open, then supplies the file path and its own result handler:

```python
        self.start_network_operation(
            "C-STORE",
            send_dicom_file,
            self.current_file_path,
            settings["local_ae_title"],
            settings["remote_ae_title"],
            settings["remote_ip"],
            settings["remote_port"],
            result_handler=self.handle_store_result,
        )
```

The network functions and result-display methods keep their existing responsibilities. The shared method owns worker creation and startup.

## Result Handling and Thread Completion

An operation result and thread completion serve different purposes:

| Signal | Responsibility |
|---|---|
| `result` | Deliver the operation return value, including reported success or failure |
| `error` | Report an unexpected exception from the worker |
| `cancelled` | Report cancellation |
| `finished` | Notify the GUI that thread execution has ended |

A reported association failure can arrive through `result`; it does not necessarily trigger the worker's `error` signal.

The cleanup method handles the worker that emitted the completion signal:

```python
    def finish_network_operation(self):
        """Clean up the finished worker and restore controls."""
        worker = self.sender()

        if worker is None:
            return

        worker.deleteLater()

        if worker is not self.network_worker:
            return

        self.network_worker = None
        self.set_network_controls_enabled(True)
```

`self.sender()` identifies the signal sender. `deleteLater()` schedules deletion rather than deleting the object immediately. If the sender is the currently tracked worker, its reference is cleared and the controls are restored.

## Validation Results

The following checks were completed during this session:

| Check | Observed result |
|---|---|
| Python syntax compilation | Passed |
| `git diff --check` | Passed; only a Windows line-ending warning was shown |
| GUI launch | Passed |
| C-ECHO association failure | Failure message displayed; button restored |
| Repeated C-ECHO | A second operation started and returned a failure result |
| C-STORE association failure | Failure message displayed; button restored |
| C-STORE followed by C-ECHO | The next operation started successfully |

Example logs:

```text
[15:17:25] C-STORE: Working...
[15:17:27] FAILED: DICOM Association failed or timed out.
[15:18:01] C-ECHO: Working...
[15:18:04] FAILED: DICOM Association failed or timed out.
```

These results verify lifecycle recovery after a reported connection failure. They do not demonstrate successful association or DICOM transmission. The cause of the connection failure was not established during this session. Cancellation after this refactoring also remains to be tested.

## Source Code

Commit: [`c95222d — Refactor C-ECHO and C-STORE worker lifecycle`](https://github.com/milanpm/PACS-DICOM-Toolkit/commit/c95222d)

The commit changed one file, with 62 insertions and 72 deletions. The main benefit is having a shared location for startup behavior.

## Next Step

Continue Day 22 Part 2 by applying the shared method to Study, Series, and Instance C-FIND, C-MOVE, and C-GET. Preserve operation-specific validation, arguments, progress callbacks, and result handlers. Then test successful operations and cooperative cancellation against the test PACS.

**Key takeaway:** Operation methods decide what to execute. The shared startup method manages worker creation and launch, while the completion handler cleans up the finished worker and restores the UI.

---

**Previous:** [Querying DICOM Series and Instances with C-FIND]({% post_url 2026-09-22-querying-dicom-series-and-instances-with-c-find %})
