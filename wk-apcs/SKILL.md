---
name: wk-apcs
description: Use when work must run in a specific thread's context, when a thread hangs inside a critical or guarded region, or when a queued kernel or user APC never arrives. Covers KAPC, KeInitializeApc, KeInsertQueueApc, delivery conditions, KeEnterCriticalRegion and KeEnterGuardedRegion, alertable waits, APC disabling, and a safe kernel-APC-to-system-thread example.
---

# Asynchronous procedure calls: thread-targeted execution and its hazards

An APC is a callback bound to one specific thread rather than to a processor. The kernel runs it later, in that thread's context, at a point where the thread is allowed to be interrupted: returning from a wait, dropping to `PASSIVE_LEVEL`, or re-entering user mode. That makes APCs the only supported way to execute code in someone else's thread context, and it makes them dangerous, because the target thread controls when, and whether, the callback ever runs. The rule that prevents most APC disasters: keep the target thread referenced and the `KAPC` alive until the kernel routine has run or the thread has terminated, and never wait for an APC while you are blocking its delivery.

## When to use

- Work must execute in a particular thread, for example the thread that issued an I/O request.
- A driver needs to run code in a system thread it owns without polling.
- File-system or filter code must wrap an operation in a critical or guarded region correctly.
- A thread is stuck and the stack shows a wait inside `KeEnterCriticalRegion` or at `APC_LEVEL`.
- A user-mode APC queued with `QueueUserAPC` or an overlapped `ReadFileEx` never runs.
- A driver is deciding between a kernel APC, a work item, and a dedicated system thread.
- Do not use for: generic deferred work with no thread affinity (`wk-dpcs`, `wk-work-items`); lock selection (`wk-spinlocks`, `wk-synchronization`); thread creation and lifetime (`wk-kernel-threads`).

## Prerequisites and related skills

- Kernel C with `<ntddk.h>`; WDK 10. The APC interfaces described here are stable from Windows 7 onward; user-APC injection behavior is not a supported contract at any version.
- The target thread must be known and referenced. `PsCreateSystemThread` plus `ObReferenceObjectByHandle` on `*PsThreadType` is the normal way to obtain one you control.
- `KAPC` storage must be non-paged and must outlive the queue operation, because the rundown routine can touch it during thread termination.
- Sibling skills: `wk-irql` (`APC_LEVEL` and `PASSIVE_LEVEL` constraints), `wk-kernel-threads` (creating and stopping the target thread), `wk-spinlocks` (what a fast mutex raises), `wk-work-items` (the usual alternative), `wk-synchronization` (events and waits).

## Mental model

Every thread owns two APC queues, one for kernel-mode APCs and one for user-mode APCs, and each entry is a `KAPC` that names the routine to run. The queues are drained by the kernel at defined delivery points, not by the inserting driver. Between the insert and the delivery, the APC sits in the list and the thread it targets may be running anything.

```text
  caller (any thread, IRQL <= DISPATCH)
     |  KeInitializeApc: bind the Apc to TargetThread
     |  KeInsertQueueApc(&Apc, arg1, arg2, 0)
     v
  TargetThread->ApcState.ApcListHead[KernelMode]      <-- pending, not running
     |
     |  delivery point: wait satisfied, IRQL lowered to PASSIVE, return to user
     v
  KernelRoutine at APC_LEVEL        may free the KAPC, may suppress the next phase
     |
     v
  NormalRoutine at PASSIVE_LEVEL    may page, allocate, take a fast mutex, signal
```

Three kinds of APC share this machinery:

| Type | Runs at | Mode | Delivered when | Used by |
|---|---|---|---|---|
| User APC | `PASSIVE_LEVEL` | user | thread performs an alertable wait | `QueueUserAPC`, `ReadFileEx`, `WriteFileEx` |
| Normal kernel APC | `PASSIVE_LEVEL` | kernel | thread reaches a delivery point, not in a critical region | drivers, I/O manager internals |
| Special kernel APC | `APC_LEVEL` | kernel | thread is not in a guarded region | I/O completion, thread suspension, kernel internals |

Delivery conditions decide whether an APC ever runs. Check all of them before assuming your callback executes:

| Condition | User APC | Normal kernel APC | Special kernel APC |
|---|---|---|---|
| Current IRQL below `APC_LEVEL` | required | required | required |
| Thread performs an alertable wait | required | not required | not required |
| Thread inside `KeEnterCriticalRegion` | blocked | blocked | still delivered |
| Thread inside `KeEnterGuardedRegion` | blocked | blocked | blocked |
| IRQL raised to `APC_LEVEL` with `KeRaiseIrql` | blocked | blocked | blocked |
| Thread has exited | rundown runs | rundown runs | rundown runs |

Two consequences are easy to miss. First, a *special* kernel APC pierces a critical region, which is exactly why the I/O manager can complete I/O while a file system holds one. Second, raising to `APC_LEVEL` blocks everything, including special APCs, so holding that level across a blocking call can stall the I/O system on that thread.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `KeInitializeApc(Apc, Thread, State, Kernel, Rundown, Normal, Mode, Context)` | Bind a `KAPC` to a thread | any | Re-initialize before every insert |
| `KeInsertQueueApc(Apc, Arg1, Arg2, Boost)` | Queue for delivery | `<= DISPATCH` | `FALSE` if already queued or thread exiting |
| `KeEnterCriticalRegion()` | Block normal and user APCs | `<= APC_LEVEL` | Counter, not a flag; must balance |
| `KeLeaveCriticalRegion()` | Undo one level | `<= APC_LEVEL` | — |
| `KeEnterGuardedRegion()` | Block every APC | `<= APC_LEVEL` | Counter; must balance |
| `KeLeaveGuardedRegion()` | Undo one level | `<= APC_LEVEL` | — |
| `KeAreApcsDisabled()` | Query both counters | any | `TRUE` inside either region |
| `KeTestAlertThread(Mode)` | Is an APC pending for this mode | `PASSIVE` | Query only; does not deliver |
| `KeAlertThread(Thread, Mode)` | Wake a thread from an alertable wait | `<= DISPATCH` | The opposite of making a thread alertable |
| `KeWaitForSingleObject` with `Alertable = TRUE` | Alertable wait | `<= DISPATCH` | `Alertable` meaningful only at `PASSIVE` |
| `KeDelayExecutionThread(Mode, Alertable, Interval)` | Sleep, optionally alertable | `PASSIVE` | Returns `STATUS_USER_APC` when alerted |
| `FsRtlEnterFileSystem()` / `FsRtlExitFileSystem()` | File-system APC guard | `PASSIVE` | Wraps `KeEnterCriticalRegion` plus bookkeeping |

`KAPC` fields a driver actually uses:

| Field | Meaning |
|---|---|
| `KernelRoutine` | Runs first, at `APC_LEVEL`, in the target thread |
| `RundownRoutine` | Runs at `DISPATCH` when the thread dies with the APC still queued |
| `NormalRoutine` | Runs at `PASSIVE_LEVEL` after the kernel routine, unless suppressed |
| `NormalContext` | Pointer handed to both routines |
| `SystemArgument1`, `SystemArgument2` | Values from `KeInsertQueueApc` |
| `ApcStateIndex` | `OriginalApcEnvironment`, `AttachedApcEnvironment`, or `CurrentApcEnvironment` |
| `ApcMode` | `KernelMode` or `UserMode` |
| `Thread`, `ApcListEntry`, `Inserted` | Managed by the kernel; never touch |

The routine signatures are fixed:

```c
VOID KernelRoutine(_In_ PKAPC Apc,
                   _Inout_ PKNORMAL_ROUTINE *NormalRoutine,
                   _Inout_ PVOID *NormalContext,
                   _Inout_ PVOID *SystemArgument1,
                   _Inout_ PVOID *SystemArgument2);

VOID NormalRoutine(_In_ PVOID NormalContext,
                   _In_ PVOID SystemArgument1,
                   _In_ PVOID SystemArgument2);

VOID RundownRoutine(_In_ PKAPC Apc);
```

The kernel routine owns the lifetime decision: setting `*NormalRoutine = NULL` skips the passive phase, and freeing a pool-allocated `KAPC` there is correct because the kernel routine is the last consumer on the normal path. If the thread terminates first, the rundown routine runs instead and must free the same memory. Whichever runs first wins; both must be safe.

## Step-by-step

1. **Decide that a thread is genuinely required.** If any thread will do, a work item or a DPC is cheaper and has no lifetime coupling to a thread object. Choose an APC only when the work must observe a specific thread's context, alert state, or attached process.
2. **Acquire and hold a thread reference.** `PsCreateSystemThread` for a thread you own, or `ObReferenceObjectByHandle` plus `ObDereferenceObject` for one you found. The reference must be held until the APC has run or the thread has terminated.
3. **Choose the environment.** `OriginalApcEnvironment` binds to the process the thread was created in and is right when the thread never attaches elsewhere. `CurrentApcEnvironment` is only valid when the caller is attached to the target process.
4. **Allocate the `KAPC`.** Embed it in the device extension, or allocate it from non-paged pool with `ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, Tag)`. Never put it in paged pool and never on the stack.
5. **Initialize on every submission.** `KeInitializeApc` must run again for each queue operation; re-initializing a `KAPC` that is still queued corrupts the list.
6. **Insert and check the result.** `KeInsertQueueApc` returning `FALSE` means the thread is exiting or the APC is already queued. Release the pending state and return `STATUS_UNSUCCESSFUL`; do not retry in a loop.
7. **Synchronize with an event.** The insert is asynchronous. The kernel routine or the normal routine must signal a `KEVENT`, and the submitter waits on it with a bounded timeout. Every path that waits must handle `STATUS_TIMEOUT` without corrupting the `KAPC`.
8. **Tear down in the right order.** Stop submitting, wake and wait for the target thread to terminate, then close the thread handle and dereference the thread object. Waiting for termination is what proves the kernel routine or the rundown has finished with the `KAPC`.

## Complete example

A driver that owns a system thread and delivers one normal kernel APC to it per request. The `KAPC` lives in the device extension, so there is no pool lifetime to manage; the thread reference and the thread-termination wait make unload safe.

```c
//
// apcdemo.c -- deliver a normal kernel APC to a driver-owned system thread.
//
#include <ntddk.h>

#define APCD_TAG      'dpcA'
#define APCD_DEVNAME  L"\\Device\\ApcDemo"
#define APCD_LINK     L"\\DosDevices\\ApcDemo"

#define IOCTL_APCD_SUBMIT \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0xA00, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_APCD_RESULT \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0xA01, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _APCD_EXTENSION {
    KAPC            Apc;              /* embedded: no pool lifetime to track */
    PKTHREAD        WorkerThread;     /* referenced thread object */
    HANDLE          WorkerHandle;
    KEVENT          KickEvent;        /* auto-reset: wakes the worker */
    KEVENT          DoneEvent;        /* notification: set by the APC */
    KEVENT          ExitedEvent;      /* set by the worker just before it dies */
    volatile LONG   Stop;
    volatile LONG   Busy;             /* 1 while a job owns the KAPC */
    volatile LONG   LastValue;
    volatile LONG   Completed;
    volatile LONG   KernelRuns;
} APCD_EXTENSION, *PAPCD_EXTENSION;

/*
 * Phase 1: APC_LEVEL, in the target thread. Non-paged work only; this is the
 * last point at which a pool-allocated KAPC could be freed on the normal path.
 */
static VOID ApcKernelRoutine(_In_ PKAPC Apc,
                             _Inout_ PKNORMAL_ROUTINE *NormalRoutine,
                             _Inout_ PVOID *NormalContext,
                             _Inout_ PVOID *SystemArgument1,
                             _Inout_ PVOID *SystemArgument2)
{
    PAPCD_EXTENSION ext = (PAPCD_EXTENSION)(*NormalContext);

    UNREFERENCED_PARAMETER(Apc);
    UNREFERENCED_PARAMETER(SystemArgument2);

    NT_ASSERT(KeGetCurrentIrql() == APC_LEVEL);
    InterlockedIncrement(&ext->KernelRuns);

    /* Leaving *NormalRoutine intact keeps the PASSIVE-level phase scheduled. */
    UNREFERENCED_PARAMETER(NormalRoutine);
    UNREFERENCED_PARAMETER(SystemArgument1);
}

/*
 * Phase 2: PASSIVE_LEVEL, still in the target thread. Paging, allocation and
 * short waits are legal here, which is the reason to have a second phase.
 */
static VOID ApcNormalRoutine(_In_ PVOID NormalContext,
                             _In_ PVOID SystemArgument1,
                             _In_ PVOID SystemArgument2)
{
    PAPCD_EXTENSION ext = (PAPCD_EXTENSION)NormalContext;
    ULONG           value = (ULONG)(ULONG_PTR)SystemArgument1;

    UNREFERENCED_PARAMETER(SystemArgument2);

    NT_ASSERT(KeGetCurrentIrql() == PASSIVE_LEVEL);

    InterlockedExchange(&ext->LastValue, (LONG)value);
    InterlockedIncrement(&ext->Completed);

    /*
     * Signal before releasing the busy flag. A new submit clears DoneEvent and
     * re-initializes the KAPC, so it must not be able to run before this one
     * has published its result and fired the event.
     */
    KeSetEvent(&ext->DoneEvent, IO_NO_INCREMENT, FALSE);
    InterlockedExchange(&ext->Busy, 0);
}

static VOID ApcWorkerThread(_In_ PVOID Context)
{
    PAPCD_EXTENSION ext = (PAPCD_EXTENSION)Context;

    for (;;) {
        KeWaitForSingleObject(&ext->KickEvent, Executive, KernelMode,
                              FALSE, NULL);

        /*
         * Returning from this wait is a delivery point: a pending kernel APC
         * runs before the loop observes the stop flag, so a request submitted
         * just before unload still completes instead of being abandoned.
         */
        if (InterlockedCompareExchange(&ext->Stop, 0, 0) != 0) {
            break;
        }
    }

    /*
     * Last touch of the device extension. From here the worker executes only
     * kernel code, which is what makes the ExitedEvent a valid teardown barrier
     * on paths that have no referenced thread object yet.
     */
    KeSetEvent(&ext->ExitedEvent, IO_NO_INCREMENT, FALSE);
    PsTerminateSystemThread(STATUS_SUCCESS);
}

static NTSTATUS ApcSubmit(_Inout_ PAPCD_EXTENSION Ext, _In_ ULONG Value)
{
    if (InterlockedCompareExchange(&Ext->Stop, 0, 0) != 0) {
        return STATUS_DELETE_PENDING;
    }
    if (InterlockedCompareExchange(&Ext->Busy, 1, 0) != 0) {
        return STATUS_DEVICE_BUSY;
    }

    KeResetEvent(&Ext->DoneEvent);

    KeInitializeApc(&Ext->Apc, Ext->WorkerThread, OriginalApcEnvironment,
                    ApcKernelRoutine, NULL, ApcNormalRoutine, KernelMode, Ext);

    if (!KeInsertQueueApc(&Ext->Apc, (PVOID)(ULONG_PTR)Value, NULL, 0)) {
        InterlockedExchange(&Ext->Busy, 0);
        return STATUS_UNSUCCESSFUL;
    }

    /* The event only makes the worker reach a delivery point; it is not the APC. */
    KeSetEvent(&Ext->KickEvent, IO_NO_INCREMENT, FALSE);
    return STATUS_SUCCESS;
}

static NTSTATUS ApcdCreateClose(_In_ PDEVICE_OBJECT DeviceObject,
                                _Inout_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS ApcdControl(_In_ PDEVICE_OBJECT DeviceObject, _Inout_ PIRP Irp)
{
    PAPCD_EXTENSION    ext = (PAPCD_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    ULONG              code = stack->Parameters.DeviceIoControl.IoControlCode;
    ULONG              outLen = stack->Parameters.DeviceIoControl.OutputBufferLength;
    ULONG              inLen = stack->Parameters.DeviceIoControl.InputBufferLength;
    PULONG             buffer = (PULONG)Irp->AssociatedIrp.SystemBuffer;
    LARGE_INTEGER      timeout;
    NTSTATUS           status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG              info = 0;

    if (KeGetCurrentIrql() != PASSIVE_LEVEL) {
        status = STATUS_INVALID_DEVICE_STATE;
        goto done;
    }

    switch (code) {
    case IOCTL_APCD_SUBMIT:
        if (buffer == NULL || inLen < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        status = ApcSubmit(ext, *buffer);
        break;

    case IOCTL_APCD_RESULT:
        if (buffer == NULL || outLen < sizeof(ULONG) * 2) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        timeout.QuadPart = -5 * 1000 * 10000LL;      /* 5 seconds, relative */
        status = KeWaitForSingleObject(&ext->DoneEvent, Executive, KernelMode,
                                       FALSE, &timeout);
        if (!NT_SUCCESS(status)) {
            break;                                   /* timeout or alert */
        }
        buffer[0] = (ULONG)InterlockedCompareExchange(&ext->LastValue, 0, 0);
        buffer[1] = (ULONG)InterlockedCompareExchange(&ext->Completed, 0, 0);
        info = sizeof(ULONG) * 2;
        status = STATUS_SUCCESS;
        break;

    default:
        break;
    }

done:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = info;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static VOID ApcdUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT   device = DriverObject->DeviceObject;
    PAPCD_EXTENSION  ext = (PAPCD_EXTENSION)device->DeviceExtension;
    UNICODE_STRING   link;

    RtlInitUnicodeString(&link, APCD_LINK);
    IoDeleteSymbolicLink(&link);

    InterlockedExchange(&ext->Stop, 1);
    KeSetEvent(&ext->KickEvent, IO_NO_INCREMENT, FALSE);

    /*
     * Waiting on the thread object is the unload barrier. When it is satisfied,
     * the worker has either run the APC or executed its rundown during exit, so
     * nothing can still reference the embedded KAPC or this driver's code.
     */
    KeWaitForSingleObject(ext->WorkerThread, Executive, KernelMode, FALSE, NULL);

    ZwClose(ext->WorkerHandle);
    ObDereferenceObject(ext->WorkerThread);
    IoDeleteDevice(device);
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT   device = NULL;
    PAPCD_EXTENSION  ext;
    UNICODE_STRING   devName;
    UNICODE_STRING   link;
    NTSTATUS         status;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, APCD_DEVNAME);
    status = IoCreateDevice(DriverObject, sizeof(APCD_EXTENSION), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PAPCD_EXTENSION)device->DeviceExtension;
    ext->WorkerThread = NULL;
    ext->WorkerHandle = NULL;
    ext->Stop = 0;
    ext->Busy = 0;
    ext->LastValue = 0;
    ext->Completed = 0;
    ext->KernelRuns = 0;

    KeInitializeEvent(&ext->KickEvent, SynchronizationEvent, FALSE);
    KeInitializeEvent(&ext->DoneEvent, NotificationEvent, FALSE);
    KeInitializeEvent(&ext->ExitedEvent, NotificationEvent, FALSE);

    RtlInitUnicodeString(&link, APCD_LINK);
    status = IoCreateSymbolicLink(&link, &devName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = ApcdCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = ApcdCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = ApcdControl;
    DriverObject->DriverUnload = ApcdUnload;

    /*
     * The thread is created last, so every earlier failure unwinds without one.
     * From here on, every failure path must stop the thread and wait for it.
     */
    status = PsCreateSystemThread(&ext->WorkerHandle, THREAD_ALL_ACCESS, NULL,
                                  NULL, NULL, ApcWorkerThread, ext);
    if (!NT_SUCCESS(status)) {
        IoDeleteSymbolicLink(&link);
        IoDeleteDevice(device);
        return status;
    }

    status = ObReferenceObjectByHandle(ext->WorkerHandle, THREAD_ALL_ACCESS,
                                       *PsThreadType, KernelMode,
                                       (PVOID *)&ext->WorkerThread, NULL);
    if (!NT_SUCCESS(status)) {
        /* No referenced object yet: ExitedEvent is the only usable barrier. */
        InterlockedExchange(&ext->Stop, 1);
        KeSetEvent(&ext->KickEvent, IO_NO_INCREMENT, FALSE);
        KeWaitForSingleObject(&ext->ExitedEvent, Executive, KernelMode,
                              FALSE, NULL);
        ZwClose(ext->WorkerHandle);
        IoDeleteSymbolicLink(&link);
        IoDeleteDevice(device);
        return status;
    }

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

Submit a value with `IOCTL_APCD_SUBMIT`, then read `IOCTL_APCD_RESULT`; the returned value is the one submitted and the counter increments once per submission. Submitting twice without reading returns `STATUS_DEVICE_BUSY` because one embedded `KAPC` cannot hold two jobs.

## Variations and advanced use

- **Critical and guarded regions as scopes.** Both are counters, so a scope guard is the only reliable way to keep them balanced across error paths. The wrappers below are C++ with no exceptions and no STL, and they must live in functions that do not also use `__try`, because mixing destructors with SEH requires `/EHa` and otherwise fails with C2712.

```cpp
// apcguard.h -- balance the APC disable counters without exceptions.
#pragma once
#include <ntddk.h>

class CriticalRegion {
public:
    CriticalRegion()  { KeEnterCriticalRegion(); }
    ~CriticalRegion() { KeLeaveCriticalRegion(); }
    CriticalRegion(const CriticalRegion&) = delete;
    CriticalRegion& operator=(const CriticalRegion&) = delete;
};

class GuardedRegion {
public:
    GuardedRegion()  { KeEnterGuardedRegion(); }
    ~GuardedRegion() { KeLeaveGuardedRegion(); }
    GuardedRegion(const GuardedRegion&) = delete;
    GuardedRegion& operator=(const GuardedRegion&) = delete;
};

// Usage, inside a PASSIVE routine that must not be interrupted by a normal APC:
//   { CriticalRegion guard; ExAcquireResourceExclusiveLite(&res, TRUE);
//     ExReleaseResourceLite(&res); }
```

- **Special kernel APC.** Pass `ApcMode = KernelMode` and leave `NormalRoutine` as `NULL`; the kernel routine then runs at `APC_LEVEL` and nothing runs at `PASSIVE`. Special APCs pierce critical regions, so this is the variant to use when the work must happen even while a file system holds a critical region. It is the same mechanism the I/O manager uses to complete I/O, and it is correspondingly easy to deadlock with.
- **User APC.** Pass `ApcMode = UserMode` and put a *user-mode* address in `NormalRoutine`. Delivery requires an alertable wait in the target thread, the address must be valid in the target process, and the whole technique is unsupported: it is the classic injection primitive, it breaks under protected-process and mitigation policies, and security products treat it as hostile. Use `QueueUserAPC` from user mode or a device event instead. If you must read the mechanism, note that the kernel routine still runs in kernel mode at `APC_LEVEL` and still owns the `KAPC` lifetime.
- **Alertable waits.** `KeWaitForSingleObject` with `Alertable = TRUE` at `PASSIVE_LEVEL` returns `STATUS_USER_APC` (`0xC0`) when a user APC was delivered. That is success, not failure: handle it, and be prepared for the wait to be restarted. An alertable wait at `APC_LEVEL` cannot deliver anything, so the flag is meaningless there.
- **APC disabling.** `KeRaiseIrql(APC_LEVEL, &old)` disables delivery on the current processor for the current thread; `KeEnterCriticalRegion` disables normal and user APCs but lets special APCs through; `KeEnterGuardedRegion` disables all three. File systems use `FsRtlEnterFileSystem`/`FsRtlExitFileSystem`, which add their own bookkeeping on top of the critical region. Never hold any of these across a long blocking call, and never wait for something that an APC must signal while delivery is disabled.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| Thread hangs forever in `KeWaitForSingleObject` | Waiting on an event that only a normal APC can signal, while inside a critical region | `!analyze -hang`, `!stacks 2`; move the wait outside the region |
| APC never runs, no crash | Target thread is not alertable (user APC), or is blocked in a critical region | `dt nt!_KAPC <addr>` shows `Inserted = 1`; check the thread's APC state |
| `0x9F` at unload | Target thread still running code from the unloading driver | Wait for the thread object before returning from `DriverUnload` |
| `0x50` with a freed `KAPC` address | Driver freed a pool `KAPC` before the kernel routine or rundown ran | `!pool`, special pool; tie the free to the routine, not to the submitter |
| `0x139` list corruption | Same `KAPC` inserted twice, or re-initialized while queued | Check `KeInsertQueueApc` return value and the `Busy` guard |
| `0x1E` or `0x8E` with a frame in your APC | Null or stale context pointer in the kernel or normal routine | `!analyze -v`, `kv`; validate `NormalContext` |
| `0x7E` in a system thread | A user APC was delivered to a thread with no user address space | Never queue `UserMode` APCs to system threads |
| `0xC0000005` in the target process | User APC routine address is not valid in that process | Map the stub first, or abandon the technique |
| `STATUS_USER_APC` (`0xC0`) from a wait | A user APC interrupted an alertable wait | Expected; loop or report, do not treat as an error |
| `KeInsertQueueApc` returns `FALSE` | APC already queued, or the thread is terminating | Reset the pending state, return failure, never spin |
| `0xCE` at unload | Pending IRPs still queued when the driver unloaded | Drain and complete the queue first (`wk-irp-queues-cancel`) |
| APC delivery stalls the system | IRQL held at `APC_LEVEL` across a blocking call, blocking special APCs | `!irql`, `!thread`; shorten the raised region |

## Safety, IRQL, and cleanup checklist

- [ ] The `KAPC` is in non-paged memory and cannot be freed while it may still be queued.
- [ ] `KeInitializeApc` runs immediately before each `KeInsertQueueApc`, never on a queued object.
- [ ] The target thread object is referenced for the entire life of the APC, and the reference is dropped only after the routine ran or the thread terminated.
- [ ] Every wait for APC completion has a bounded timeout and a defined behavior on timeout.
- [ ] `KeEnterCriticalRegion`/`KeLeaveCriticalRegion` and the guarded-region pair are balanced on every path, including error returns.
- [ ] No blocking call, file I/O, registry call, or alertable wait inside a critical or guarded region.
- [ ] The kernel routine never blocks; it frees only non-paged allocations, and any work that benefits from paging is left to the normal routine at `PASSIVE_LEVEL`.
- [ ] The rundown routine is present whenever a pool-allocated `KAPC` is used; without it, thread exit leaks the allocation.
- [ ] Unload stops new submissions, wakes the thread, waits for the thread object, then closes the handle and dereferences the object.
- [ ] User-mode APCs are not used for anything except a documented, supported user-mode feature.

## Verification

1. Build and install with your test certificate, start the driver, and issue `IOCTL_APCD_SUBMIT` then `IOCTL_APCD_RESULT`. Expected: the value echoes back and `Completed` increments once per submission.
2. Confirm the two phases: in the debugger, `bp apcdemo!ApcKernelRoutine` and `bp apcdemo!ApcNormalRoutine`. At each break, `!irql` must print `APC_LEVEL` for the kernel routine and `PASSIVE_LEVEL` for the normal routine, and `!thread` must show the worker thread, not the submitting thread.
3. Confirm the queue drains: with no submission outstanding, `dt nt!_KAPC <address of ext->Apc>` must show `Inserted = 0`.
4. Timeout path: submit a value, then stop the worker thread from the debugger (`.suspend` on its thread) and issue `IOCTL_APCD_RESULT`. Expected: `STATUS_TIMEOUT` (`0x00000102`) after five seconds, no bugcheck, and a later resume still completes the job.
5. Unload under load: submit a value and immediately stop the service. Unload must finish, and `!irpfind` must show no IRPs belonging to the device.
6. Stress: loop submit/result one hundred thousand times from four client threads and confirm `Completed` matches the submissions, `KernelRuns` matches, and pool usage is flat (`!poolused 2` before and after).
7. Verifier: `verifier /standard /driver apcdemo.sys`, reboot, and repeat the stress loop; any APC lifetime error surfaces as `0xC4`, `0xC1`, or `0x50` with a specific address rather than a silent corruption.
