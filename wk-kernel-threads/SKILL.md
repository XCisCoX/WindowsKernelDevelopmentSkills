---
name: wk-kernel-threads
description: Use when a driver must run work that outlives the current IRP, poll hardware, block on an event, or run in another process's address space. Covers PsCreateSystemThread, PsTerminateSystemThread, the KSTART_ROUTINE contract, CLIENT_ID, ObReferenceObjectByHandle to PETHREAD, KeStackAttachProcess, stop-event shutdown at unload, priority and affinity, and 0x9F/0xCE diagnosis.
---

# System threads: owning an execution context that outlives the request

A request-driven driver executes on threads it does not own, in whatever process context the caller happened to be, for only as long as the IRP takes. When work must continue after the IRP completes, must block for a long time, or must run in a different process's address space, the driver has to create its own thread. `PsCreateSystemThread` builds an `ETHREAD` with a kernel stack and starts a routine that runs until it calls `PsTerminateSystemThread`. The rule that prevents nearly every disaster in this area: **whoever creates the thread owns its shutdown; `DriverUnload` must signal the thread, wait for the thread object, and only then release the device and the memory that the routine touches**. A thread that is still running when unload returns executes code from an image that is about to be unmapped, and the resulting bugcheck is blamed on whatever ran next.

## When to use

- A driver must poll device state, a software queue, or another driver on a timer that is longer than a DPC allows and must keep running between requests.
- Work must survive the request that started it: post-processing after the client closed its handle, deferred retries, or cleanup of network/hardware state.
- The work must block: wait on an event with no timeout, call `Zw*` file or registry APIs, take a mutex, or sleep for hundreds of milliseconds.
- The driver must read or write another process's user-mode memory, which requires attaching to that process.
- A watchdog, heartbeat, or timeout reaper must run independently of the IRP flow.
- `0x9F` or `0xCE` is suspected to come from work that outlived its device or its driver's unload.
- Do not use for: short deferred work at `PASSIVE_LEVEL` (`wk-work-items` costs one worker thread and no lifetime management); deferral out of an ISR or a high-IRQL path (`wk-dpcs`); running code in a specific existing thread's context (`wk-apcs`); locking primitives for the thread's own data (`wk-synchronization`).

## Prerequisites and related skills

- Kernel C with `<ntddk.h>` or `<wdm.h>`. `PsCreateSystemThread`, `PsTerminateSystemThread`, `ObReferenceObjectByHandle`, and `KeStackAttachProcess` are available on every supported target; `IoCreateSystemThread` requires Windows 8 or later, and `KeQueryActiveProcessorCount` requires Windows Vista or later.
- `PsCreateSystemThread` and the thread start routine both require `PASSIVE_LEVEL`. `KeWaitForSingleObject` on a thread object is legal up to `DISPATCH_LEVEL` but must be called at `PASSIVE_LEVEL` in an unload path, where it may block for an unbounded time.
- The driver must already own the device object whose extension holds the thread's state, because the thread's memory must outlive every iteration of its loop.
- Every pointer the routine dereferences must be non-paged, or in a pageable section that the image keeps resident while the thread runs. The routine must never be placed in an `INIT` section.
- Sibling skills: `wk-work-items` (deferring short work without a dedicated thread), `wk-dpcs` (what may run at `DISPATCH_LEVEL`), `wk-apcs` (queuing work into an existing thread), `wk-synchronization` (events and wait semantics), `wk-object-manager` (handle semantics and `OBJ_KERNEL_HANDLE`), `wk-irql` (why the thread can block and a DPC cannot), `wk-crash-dump-analysis` (reading `0x9F`/`0xCE`).

## Mental model

Every thread is an `ETHREAD` whose first member is a `KTHREAD`, plus a kernel stack and membership in exactly one process. A **system thread** is an ordinary thread whose process is the System process (`PsInitialSystemProcess`, PID 4) unless the caller names a different process at creation. That single fact explains most of the surprises: the System process has no user-mode address space and no window station, so a system thread can run for the life of the machine but cannot touch a pointer that belongs to a client, and cannot show anything on a desktop.

The thread object is a dispatcher object: non-signaled while the thread runs, signaled once the thread has terminated. Waiting on it is a **join**, and that is the only correct way to know that the routine will never touch your memory again. The handle returned by `PsCreateSystemThread` is a handle-table entry, not the object itself. It can be closed as soon as a referenced object pointer is held, and closing it early removes an entire class of leak from the unload path.

The start routine is a plain C callback: no `this` pointer, no return value, no exceptions. It runs at `PASSIVE_LEVEL` in a thread that has no user context unless one is attached, and for `PsCreateSystemThread` it does not return — the routine must end in `PsTerminateSystemThread`. Everything the routine touches must stay valid for the whole interval between create and terminate, which is why the extension holding the stop event belongs to the device object that is deleted last.

```text
     DriverEntry                                         DriverUnload
         |                                                   |
         | 1 KeInitializeEvent(StopEvent)                    | 4 KeSetEvent(StopEvent)
         | 2 PsCreateSystemThread ------------------+        | 5 ObReferenceObjectByHandle
         | 3 ObReferenceObjectByHandle              |        |   (already done at start)
         |                                          v        | 6 KeWaitForSingleObject(thread)
         |                                   +--------------+| 7 ObDereferenceObject
         |                                   |  PASSIVE     || 8 ZwClose(handle)
         |                                   |  loop:       || 9 IoDeleteDevice
         |                                   |   wait stop  ||
         |                                   |   or timeout ||
         |                                   |   poll once  ||
         |                                   +--------------+|
         |                                          |        |
         |                                PsTerminateSystemThread
         |                                          |
         +------------------------------------------+---> thread object signaled
```

Three consequences follow from this picture and drive every rule in the rest of the skill.

1. **Context is not inherited from the request.** A system thread created with `ProcessHandle = NULL` runs in the System process forever. Any user-mode virtual address captured from an IRP is meaningless there, and dereferencing it faults in non-paged memory (`0x50`) or at the wrong IRQL (`0xA`). Either copy the data before the thread reads it, or attach.
2. **The join is the lifetime boundary.** Between create and join, the routine may run at any moment. After a successful join it cannot. Therefore every teardown action — deleting the device, freeing the extension, closing a handle the routine uses, deleting a symbolic link — goes after the join.
3. **The thread runs the driver's code.** Unload cannot complete while the routine may still execute; that is a correctness requirement, not a politeness. `DriverUnload` must not return until the join succeeds.

## Key APIs and structures

| API / structure | Purpose | IRQL / context | Notes |
|---|---|---|---|
| `PsCreateSystemThread(&Handle, Access, &OA, ProcessHandle, &Cid, Routine, Context)` | Create the thread | `PASSIVE_LEVEL` | Returns a handle; the object may land in the System process |
| `IoCreateSystemThread(IoObject, &Handle, ...)` | Create with an object reference and automatic termination | `PASSIVE_LEVEL` | Windows 8+; the routine may simply return |
| `PsTerminateSystemThread(Status)` | End the calling thread | `PASSIVE_LEVEL` | Does not return; `Status` becomes the exit status |
| `KeGetCurrentThread()` / `PsGetCurrentThread()` | `PKTHREAD` / `PETHREAD` of the caller | any | Cheap; no reference added |
| `PsGetCurrentProcess()` / `IoGetCurrentProcess()` | `PEPROCESS` of the caller | any | Returns the attached process while attached |
| `PsGetCurrentThreadId()` / `PsGetCurrentProcessId()` | `HANDLE` IDs of the caller | any | For traces and debugger output |
| `PsGetThreadProcess(Thread)` | Owning `PEPROCESS` | any | No reference added |
| `PsLookupThreadByThreadId(Tid, &Thread)` | Thread ID to referenced `PETHREAD` | `PASSIVE_LEVEL` | Caller must `ObDereferenceObject` |
| `PsLookupProcessByProcessId(Pid, &Process)` | Process ID to referenced `PEPROCESS` | `PASSIVE_LEVEL` | Caller must `ObDereferenceObject` |
| `ObReferenceObjectByHandle(h, Access, *PsThreadType, KernelMode, &Obj, NULL)` | Handle to referenced pointer | `PASSIVE_LEVEL` | The canonical handle-to-object conversion |
| `ObDereferenceObject(Obj)` | Drop a reference | `<= DISPATCH_LEVEL` | Never dereference what you did not reference |
| `KeWaitForSingleObject(Thread, Executive, KernelMode, FALSE, Timeout)` | Join the thread | `PASSIVE_LEVEL` in practice | Signaled by termination |
| `KeStackAttachProcess((PRKPROCESS)P, &ApcState)` | Run in `P`'s address space | `PASSIVE_LEVEL` | Must be balanced, even on error paths |
| `KeUnstackDetachProcess(&ApcState)` | Restore the previous address space | `PASSIVE_LEVEL` | Same thread, same stack frame |
| `KeSetPriorityThread(KeGetCurrentThread(), P)` | Change a thread's base priority | `<= DISPATCH_LEVEL` | `0..31`; 8 is normal, 16+ is realtime |
| `KeQueryPriorityThread(Thread)` | Read base priority | any | Returns `KPRIORITY` |
| `ZwSetInformationThread(h, ThreadAffinityMask, &Mask, sizeof(Mask))` | Hard affinity for a thread | `PASSIVE_LEVEL` | Mask must be a non-empty subset of the process mask |
| `ZwSetInformationThread(h, ThreadIdealProcessor, &Cpu, sizeof(Cpu))` | Soft preference | `PASSIVE_LEVEL` | `MAXIMUM_PROCESSORS` means "no preference" |
| `PsIsSystemThread(Thread)` | Is this a system thread | any | Useful when validating a thread object you were handed |
| `KeQueryActiveProcessorCount(NULL)` | Number of active processors | any | Vista+; use for affinity arithmetic |

The types the calls are built from:

```c
typedef struct _CLIENT_ID {
    HANDLE UniqueProcess;      /* the process ID */
    HANDLE UniqueThread;       /* the thread ID  */
} CLIENT_ID, *PCLIENT_ID;

typedef VOID (NTAPI *PKSTART_ROUTINE)(_In_ PVOID StartContext);

typedef struct _KAPC_STATE *PKAPC_STATE;   /* opaque; allocate on the stack */

typedef struct _OBJECT_ATTRIBUTES {
    ULONG           Length;
    HANDLE          RootDirectory;
    PUNICODE_STRING ObjectName;
    ULONG           Attributes;
    PVOID           SecurityDescriptor;
    PVOID           SecurityQualityOfService;
} OBJECT_ATTRIBUTES, *POBJECT_ATTRIBUTES;
```

Parameter contract for `PsCreateSystemThread`:

- `ThreadHandle` — receives the handle. Always pair it with an `OBJECT_ATTRIBUTES` carrying `OBJ_KERNEL_HANDLE`. Without that flag the handle is created in whatever process context the caller is in: during `DriverEntry` that is the service control manager's thread, so the handle dies with that process and every later use returns `STATUS_INVALID_HANDLE`. With the flag the handle lives in the system handle table and cannot leak into a user process.
- `DesiredAccess` — `THREAD_ALL_ACCESS` is the practical value; anything narrower only limits what the handle can be used for later.
- `ProcessHandle` — `NULL` creates the thread in the System process, which is what a poller or watchdog wants. `NtCurrentProcess()` creates it in the calling process; the thread then keeps that process alive and dies with it. Any other value must be a resolvable handle to a process object.
- `ClientId` — optional `CLIENT_ID` output; capture it for the debugger even if the code never uses the IDs.
- `StartContext` — the single pointer the routine receives. Pass the extension, or a small non-paged structure that contains it.

## Step-by-step

1. **Decide the stop condition before writing the loop.** The thread must have exactly one reason to exit and one event that produces it. If the answer is "when there is nothing to do", the design is a work item, not a thread. Check: the routine contains a single `for (;;)` whose only exits are the stop event and a fatal error.
2. **Put the state where it lives longest.** Use the device extension, or `ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, Tag)` freed after the join. Never use a pool block owned by an IRP, a `FILE_OBJECT` context, or a stack frame of the creating routine. Check: the state outlives every possible iteration of the loop.
3. **Initialize the stop event once.** `KeInitializeEvent(&Ext->StopEvent, NotificationEvent, FALSE)`. A notification event stays signaled, so a stop request that arrives before the thread's first wait is not lost. Check: the event is initialized before `PsCreateSystemThread`.
4. **Build the object attributes with `OBJ_KERNEL_HANDLE`.** `InitializeObjectAttributes(&oa, NULL, OBJ_KERNEL_HANDLE, NULL, NULL)`. Check: the returned handle still resolves after the creating process exits.
5. **Create the thread and check the status.** `STATUS_INSUFFICIENT_RESOURCES` (`0xC000009A`) and `STATUS_PROCESS_IS_TERMINATING` (`0xC000010A`, when the target process is exiting) both arrive here. On failure, unwind everything already built and return the status. Check: no handle is left open on any failure path.
6. **Convert the handle to a referenced object.** `ObReferenceObjectByHandle(handle, THREAD_ALL_ACCESS, *PsThreadType, KernelMode, (PVOID *)&thread, NULL)`. Store the pointer; this is what the join waits on. Check: every successful reference is matched by exactly one `ObDereferenceObject`.
7. **Either keep or close the handle.** Close it immediately if nothing else needs it; keep it only when a later `ZwSetInformationThread` call (affinity, ideal processor, priority) needs a handle. Check: `ZwClose` appears exactly once per handle on every path.
8. **Log the `CLIENT_ID`.** `DbgPrintEx(..., "... tid %lu pid %lu", (ULONG)(ULONG_PTR)cid.UniqueThread, ...)` gives the debugger the IDs to find the thread with `!thread`.
9. **Write the routine to the contract.** Set priority first, then loop: compute the relative timeout in 100 ns units as a negative `LARGE_INTEGER`, wait on the stop event, treat `STATUS_TIMEOUT` as "do one iteration", treat any other return as "exit", do a bounded amount of work, and end in `PsTerminateSystemThread(STATUS_SUCCESS)`. Check: no path returns from the routine without terminating.
10. **Make the unload path follow the order** in the checklist: set the stopping flag, signal the event, join, dereference, close the handle, delete the link, delete the device. Check: `IoDeleteDevice` is the last statement, and the join precedes it.
11. **Prove it under the debugger.** Break in the routine, run `!thread` and `!irql`; both should show `PASSIVE_LEVEL` for a well-behaved poller, and `!stacks 2` should show the thread in `Wait:WrExecutive` while it is idle.

## Complete example

A control device whose poller starts in `DriverEntry` and stops during unload. The public header is shared with the client.

```c
/* KwThreadPublic.h -- shared between the driver and its client */
#pragma once

#define KWT_DEVICE_TYPE 0x8333

typedef struct _KWT_STATS {
    ULONG PollCount;      /* poll iterations completed            */
    ULONG IntervalMs;     /* current wait interval                */
    ULONG LastLateUs;     /* how late the last wait returned      */
    ULONG WorstLateUs;    /* worst lateness seen since load       */
    ULONG ThreadId;       /* thread ID of the poller              */
    ULONG Running;        /* 1 while the thread is alive          */
} KWT_STATS, *PKWT_STATS;

#define IOCTL_KWT_GET_STATS \
    CTL_CODE(KWT_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_KWT_SET_INTERVAL \
    CTL_CODE(KWT_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_KWT_SET_AFFINITY \
    CTL_CODE(KWT_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_ANY_ACCESS)
```

```c
/* kwthread.c -- a polling system thread with a clean unload path */
#include <ntddk.h>
#include "KwThreadPublic.h"

#define KWT_DEVNAME   L"\\Device\\KWThread"
#define KWT_SYMLINK   L"\\??\\KWThread"
#define KWT_PREFIX    "KWT: "

#define KWT_POLL_MS   500
#define KWT_MIN_MS    10
#define KWT_MAX_MS    60000
#define KWT_PRIORITY  9          /* 0..31; 8 is normal, 16+ is realtime */

typedef struct _KWT_DEVICE_EXT {
    KEVENT            StopEvent;
    PETHREAD          ThreadObject;      /* referenced; the join target */
    HANDLE            ThreadHandle;      /* kernel handle; kept for Zw* calls */
    volatile LONG     IntervalMs;
    volatile LONG     StopRequested;
    volatile LONG     PollCount;
    volatile LONG     LastLateUs;
    volatile LONG     WorstLateUs;
    ULONG             ThreadId;
    volatile ULONGLONG LastPollTime;
} KWT_DEVICE_EXT, *PKWT_DEVICE_EXT;

static VOID KwtPollRoutine(_In_ PVOID Context);

/* Keep the largest observed value without a lock: compare-and-swap retry loop. */
static VOID KwtUpdateWorst(_Inout_ volatile LONG *Worst, _In_ LONG Value)
{
    LONG current = *Worst;

    while (Value > current) {
        LONG previous = InterlockedCompareExchange(Worst, Value, current);
        if (previous == current) {
            break;
        }
        current = previous;
    }
}

/* One iteration of real work. Replace with the device or queue poll. */
static VOID KwtPollOnce(_In_ PKWT_DEVICE_EXT Ext)
{
    ULONG count = (ULONG)InterlockedIncrement(&Ext->PollCount);

    Ext->LastPollTime = KeQueryInterruptTime();
    if ((count % 20) == 0) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   KWT_PREFIX "heartbeat %lu\n", count);
    }
}

/*
 * Runs at PASSIVE_LEVEL in a system thread. It never returns: every exit
 * path ends in PsTerminateSystemThread.
 */
static VOID KwtPollRoutine(_In_ PVOID Context)
{
    PKWT_DEVICE_EXT ext = (PKWT_DEVICE_EXT)Context;
    LARGE_INTEGER   start;
    LARGE_INTEGER   end;
    LARGE_INTEGER   timeout;
    LONGLONG        late;
    ULONG           interval;
    NTSTATUS        status;

    (VOID)KeSetPriorityThread(KeGetCurrentThread(), KWT_PRIORITY);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               KWT_PREFIX "thread %lu running\n", ext->ThreadId);

    for (;;) {
        interval = (ULONG)InterlockedCompareExchange(&ext->IntervalMs, 0, 0);
        if (interval < KWT_MIN_MS || interval > KWT_MAX_MS) {
            interval = KWT_POLL_MS;
        }
        timeout.QuadPart = -(LONGLONG)interval * 10000;   /* relative, 100 ns */

        start.QuadPart = (LONGLONG)KeQueryInterruptTime();
        status = KeWaitForSingleObject(&ext->StopEvent, Executive, KernelMode,
                                       FALSE, &timeout);
        if (status != STATUS_TIMEOUT) {
            break;                       /* stop requested, or a wait failure */
        }
        end.QuadPart = (LONGLONG)KeQueryInterruptTime();

        late = (end.QuadPart - start.QuadPart) - (LONGLONG)interval * 10000;
        if (late > 0) {
            ULONG lateUs = (ULONG)(late / 10);       /* 100 ns ticks to us */
            InterlockedExchange(&ext->LastLateUs, (LONG)lateUs);
            KwtUpdateWorst(&ext->WorstLateUs, (LONG)lateUs);
        }

        KwtPollOnce(ext);
    }

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               KWT_PREFIX "thread %lu stopping\n", ext->ThreadId);

    PsTerminateSystemThread(STATUS_SUCCESS);
}

/* Create in the System process, convert the handle, keep both forms. */
static NTSTATUS KwtStartThread(_Inout_ PKWT_DEVICE_EXT Ext)
{
    OBJECT_ATTRIBUTES oa;
    CLIENT_ID         cid;
    PETHREAD          thread = NULL;
    HANDLE            handle = NULL;
    NTSTATUS          status;

    InitializeObjectAttributes(&oa, NULL, OBJ_KERNEL_HANDLE, NULL, NULL);

    status = PsCreateSystemThread(&handle, THREAD_ALL_ACCESS, &oa, NULL, &cid,
                                  KwtPollRoutine, Ext);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   KWT_PREFIX "PsCreateSystemThread failed 0x%08X\n", status);
        return status;
    }

    status = ObReferenceObjectByHandle(handle, THREAD_ALL_ACCESS, *PsThreadType,
                                       KernelMode, (PVOID *)&thread, NULL);
    if (!NT_SUCCESS(status)) {
        ZwClose(handle);
        return status;
    }

    Ext->ThreadObject = thread;
    Ext->ThreadHandle = handle;
    Ext->ThreadId = (ULONG)(ULONG_PTR)cid.UniqueThread;

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               KWT_PREFIX "created tid %lu in pid %lu\n",
               Ext->ThreadId, (ULONG)(ULONG_PTR)cid.UniqueProcess);
    return STATUS_SUCCESS;
}

static NTSTATUS KwtComplete(_In_ PIRP Irp, _In_ NTSTATUS Status,
                            _In_ ULONG_PTR Information)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = Information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return Status;
}

static NTSTATUS KwtCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return KwtComplete(Irp, STATUS_SUCCESS, 0);
}

static NTSTATUS KwtControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PKWT_DEVICE_EXT   ext = (PKWT_DEVICE_EXT)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PVOID             buffer = Irp->AssociatedIrp.SystemBuffer;
    ULONG             inLen = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG             outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    NTSTATUS          status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR         info = 0;

    if (ext->ThreadHandle == NULL) {
        return KwtComplete(Irp, STATUS_DEVICE_NOT_READY, 0);
    }

    switch (sp->Parameters.DeviceIoControl.IoControlCode) {
    case IOCTL_KWT_GET_STATS:
        if (buffer == NULL || outLen < sizeof(KWT_STATS)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        {
            PKWT_STATS stats = (PKWT_STATS)buffer;

            stats->PollCount = (ULONG)InterlockedCompareExchange(&ext->PollCount, 0, 0);
            stats->IntervalMs = (ULONG)InterlockedCompareExchange(&ext->IntervalMs, 0, 0);
            stats->LastLateUs = (ULONG)InterlockedCompareExchange(&ext->LastLateUs, 0, 0);
            stats->WorstLateUs = (ULONG)InterlockedCompareExchange(&ext->WorstLateUs, 0, 0);
            stats->ThreadId = ext->ThreadId;
            stats->Running =
                (InterlockedCompareExchange(&ext->StopRequested, 0, 0) == 0);
            info = sizeof(KWT_STATS);
            status = STATUS_SUCCESS;
        }
        break;

    case IOCTL_KWT_SET_INTERVAL:
        if (buffer == NULL || inLen < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        if (*((PULONG)buffer) < KWT_MIN_MS || *((PULONG)buffer) > KWT_MAX_MS) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        InterlockedExchange(&ext->IntervalMs, (LONG)*((PULONG)buffer));
        info = sizeof(ULONG);          /* takes effect on the next wait */
        status = STATUS_SUCCESS;
        break;

    case IOCTL_KWT_SET_AFFINITY:
        if (buffer == NULL || inLen < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        {
            ULONG     cpu = *((PULONG)buffer);
            KAFFINITY mask;

            if (cpu >= KeQueryActiveProcessorCount(NULL)) {
                status = STATUS_INVALID_PARAMETER;
                break;
            }
            mask = (KAFFINITY)1 << cpu;
            status = ZwSetInformationThread(ext->ThreadHandle, ThreadAffinityMask,
                                            &mask, sizeof(mask));
            if (NT_SUCCESS(status)) {
                info = sizeof(ULONG);
            }
        }
        break;

    default:
        break;
    }

    return KwtComplete(Irp, status, info);
}

/*
 * Order matters: stop the producer, join it, release the handle, and only
 * then destroy the memory the thread was using.
 */
static VOID KwtUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT   device = DriverObject->DeviceObject;
    PKWT_DEVICE_EXT  ext = (PKWT_DEVICE_EXT)device->DeviceExtension;
    UNICODE_STRING   link;

    RtlInitUnicodeString(&link, KWT_SYMLINK);
    IoDeleteSymbolicLink(&link);

    InterlockedExchange(&ext->StopRequested, 1);
    KeSetEvent(&ext->StopEvent, IO_NO_INCREMENT, FALSE);

    if (ext->ThreadObject != NULL) {
#if DBG
        LARGE_INTEGER deadline;
        NTSTATUS      status;

        deadline.QuadPart = -(LONGLONG)10 * 1000 * 10000;   /* 10 seconds */
        status = KeWaitForSingleObject(ext->ThreadObject, Executive, KernelMode,
                                       FALSE, &deadline);
        if (status == STATUS_TIMEOUT) {
            DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                       KWT_PREFIX "poll thread stuck; breaking in\n");
            DbgBreakPoint();
            KeWaitForSingleObject(ext->ThreadObject, Executive, KernelMode,
                                  FALSE, NULL);
        }
#else
        KeWaitForSingleObject(ext->ThreadObject, Executive, KernelMode,
                              FALSE, NULL);
#endif
        ObDereferenceObject(ext->ThreadObject);
        ext->ThreadObject = NULL;
    }

    if (ext->ThreadHandle != NULL) {
        ZwClose(ext->ThreadHandle);
        ext->ThreadHandle = NULL;
    }

    IoDeleteDevice(device);
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT   device = NULL;
    PKWT_DEVICE_EXT  ext;
    UNICODE_STRING   devName;
    UNICODE_STRING   link;
    NTSTATUS         status;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, KWT_DEVNAME);
    status = IoCreateDevice(DriverObject, sizeof(KWT_DEVICE_EXT), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PKWT_DEVICE_EXT)device->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));
    KeInitializeEvent(&ext->StopEvent, NotificationEvent, FALSE);
    InterlockedExchange(&ext->IntervalMs, KWT_POLL_MS);

    DriverObject->MajorFunction[IRP_MJ_CREATE] = KwtCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = KwtCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = KwtControl;
    DriverObject->DriverUnload = KwtUnload;

    RtlInitUnicodeString(&link, KWT_SYMLINK);
    status = IoCreateSymbolicLink(&link, &devName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    status = KwtStartThread(ext);
    if (!NT_SUCCESS(status)) {
        IoDeleteSymbolicLink(&link);
        IoDeleteDevice(device);
        return status;
    }

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

```c
/* kwthreadclient.c -- read the poller's counters and change its schedule */
#include <windows.h>
#include <stdio.h>
#include "KwThreadPublic.h"

int main(void)
{
    HANDLE    device;
    KWT_STATS stats;
    DWORD     bytes = 0;
    ULONG     value;

    device = CreateFileW(L"\\\\.\\KWThread", GENERIC_READ | GENERIC_WRITE,
                         0, NULL, OPEN_EXISTING, 0, NULL);
    if (device == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }

    value = 100;
    if (!DeviceIoControl(device, IOCTL_KWT_SET_INTERVAL, &value, sizeof(value),
                         NULL, 0, &bytes, NULL)) {
        printf("set interval failed: %lu\n", GetLastError());
    }

    Sleep(1000);

    if (DeviceIoControl(device, IOCTL_KWT_GET_STATS, NULL, 0, &stats,
                        sizeof(stats), &bytes, NULL)) {
        printf("polls=%lu interval=%lu ms last=%lu us worst=%lu us "
               "tid=%lu running=%lu\n",
               stats.PollCount, stats.IntervalMs, stats.LastLateUs,
               stats.WorstLateUs, stats.ThreadId, stats.Running);
    } else {
        printf("get stats failed: %lu\n", GetLastError());
    }

    value = 1;                     /* pin the poller to CPU 1 */
    if (!DeviceIoControl(device, IOCTL_KWT_SET_AFFINITY, &value, sizeof(value),
                         NULL, 0, &bytes, NULL)) {
        printf("set affinity failed: %lu\n", GetLastError());
    }

    CloseHandle(device);
    return 0;
}
```

With a 100 ms interval the counters advance about ten times per second. `WorstLateUs` stays in the low hundreds on an idle machine and jumps into the thousands under load, which is exactly the measurement a poller exists to expose.

## Variations and advanced use

- **`IoCreateSystemThread` instead of the two-call sequence.** From Windows 8, `IoCreateSystemThread(&Ext->IoObject, &handle, THREAD_ALL_ACCESS, &oa, NULL, &cid, Routine, Context)` takes a reference on a device or driver object, terminates the thread for you when the routine returns, and leaves the exit status as `STATUS_SUCCESS`. The reference is a backstop for object lifetime, not a substitute for the join: `DriverUnload` still runs while the thread runs, and still must wait.

- **Create the thread in a user process.** Passing `NtCurrentProcess()` as `ProcessHandle` puts the thread in the calling process, which is required when the work is session-bound (audio, a desktop, a per-session endpoint). Two prices: the thread keeps that process alive as an orphan even after the user's last window closes, and the process can exit at any moment, after which `PsTerminateSystemThread` fires under the thread and any user address it cached is gone. Record which process you chose, and reject requests from a different session rather than creating a second thread.

- **Attach to read another process's memory.** Attaching swaps the address space only; it does not change the thread's token, so it grants no access rights and defeats no security check. The dangerous part is the error path: a fault after the attach must not skip the detach, or the thread stays in a process that can exit underneath it.

```c
static NTSTATUS KwtPeekUserUlong(_In_ HANDLE ProcessId,
                                 _In_ const VOID *UserAddress,
                                 _Out_ PULONG Value)
{
    PEPROCESS Process = NULL;
    KAPC_STATE ApcState;
    BOOLEAN   attached = FALSE;
    NTSTATUS  status;

    status = PsLookupProcessByProcessId(ProcessId, &Process);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    __try {
        KeStackAttachProcess((PRKPROCESS)Process, &ApcState);
        attached = TRUE;
        ProbeForRead(UserAddress, sizeof(ULONG), sizeof(ULONG));
        *Value = *(const volatile ULONG *)UserAddress;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
    }

    if (attached) {
        KeUnstackDetachProcess(&ApcState);      /* never inside the handler */
    }

    ObDereferenceObject(Process);
    return status;
}
```

  `KAPC_STATE` must live on the attaching thread's own stack and the pair must be perfectly nested. This helper is an arbitrary kernel read primitive; never expose it through an IOCTL without an access check on the device, or the driver becomes the vulnerability.

- **Priority.** Set it from inside the routine with `KeSetPriorityThread(KeGetCurrentThread(), 9)` so there is no window in which the `KTHREAD` pointer could already be stale. Priorities 0..31 map to the 32 Windows priority levels; 8 is the normal class, and anything at 16 or above is realtime and can starve the input and disk paths on a single-processor VM. A poller almost never needs more than 9..11. `KeQueryPriorityThread` reads it back for a trace.

- **Affinity.** `ThreadAffinityMask` is a hard pin: the scheduler will only run that thread on the chosen set, and the mask must be a non-empty subset of the process affinity mask, which is why the call returns `STATUS_INVALID_PARAMETER` on a machine with fewer processors than the bit you set. `ThreadIdealProcessor` is the soft form and lets the scheduler migrate the thread when the preferred CPU is busy. For a short critical region, `KeSetSystemAffinityThreadEx(mask)` / `KeRevertToUserAffinityThreadEx(saved)` pin the *calling* thread and must be balanced on the same thread.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `0xCE DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` at `sc stop` | `DriverUnload` returned while the thread still had work or pending IRPs; the image was unmapped under it | Join before returning; `!analyze -v`, then check the module list for the driver's image |
| `0x9F DRIVER_POWER_STATE_FAILURE` during sleep or resume | A thread is blocked in a pending IRP or holding device state across the power transition, so the power IRP never completes | Stop the poller at `IRP_MN_SET_POWER`/D0-exit, not only at unload; never wait on your own device from the poll loop |
| `0x50` in the routine with a user-mode-looking address | The system thread dereferenced a pointer captured from an IRP; the System process has no such address | Copy the data in the dispatch routine, or attach first; `!analyze -v` plus `!pte` on the faulting address |
| `0xA` inside the routine | The routine raised IRQL (for example with a spinlock) and then touched paged memory or waited | Keep IRQL at `PASSIVE_LEVEL`; `!irql` at the breakpoint |
| `0x1E`/`0xCE` immediately after the routine returns | The routine returned instead of calling `PsTerminateSystemThread` | Every exit path must terminate; there is no return address to go back to |
| `STATUS_INVALID_HANDLE` from a later `Zw*` call on the thread handle | The handle was created without `OBJ_KERNEL_HANDLE`, in a process that has since exited | Rebuild the object attributes with `OBJ_KERNEL_HANDLE` |
| Handle count in the System process grows on every start/stop cycle | A `ZwClose` is missing on a success or failure path | Compare `!process 4 0` handle counts across ten load/unload cycles |
| Unload hangs forever, `sc stop` times out | The loop is blocked in something that ignores the stop event: an infinite wait on another event, a synchronous IRP to a device that is gone, or a mutex held by a thread that is waiting for unload | `!stacks 2` to find the blocked frames; bound every wait inside the loop and re-check the stop event between chunks |
| The thread never runs after a successful create | The image section that contains the routine was discarded (an `INIT`-only section), or the routine's first instruction faults | Never put a thread routine in `INIT`; `lm` and `!thread` on the new thread |
| `0xC000010A STATUS_PROCESS_IS_TERMINATING` from `PsCreateSystemThread` | `ProcessHandle` names a process that is already exiting | Create in the System process, or fail the request cleanly |
| `0xC000009A STATUS_INSUFFICIENT_RESOURCES` | One thread per request exhausted the system | One long-lived thread per device, not per IRP |
| `STATUS_TIMEOUT` treated as an error, so the poller exits after one interval | The loop compared the wait result to `STATUS_SUCCESS` | `STATUS_TIMEOUT` is the normal case; only its absence means "stop" |

## Safety, IRQL, and cleanup checklist

- [ ] The start routine runs at `PASSIVE_LEVEL`, lives outside any `INIT` section, and is declared with the `PKSTART_ROUTINE` signature.
- [ ] Every exit path from the routine calls `PsTerminateSystemThread`; there is no `return` statement in the routine.
- [ ] All state the routine touches is non-paged or in the device extension, and is freed only after the join.
- [ ] `KeInitializeEvent` runs before the thread is created, and the stop event is a notification event so a stop request cannot be lost.
- [ ] The thread handle is created with `OBJ_KERNEL_HANDLE` and closed exactly once on every path.
- [ ] The object reference from `ObReferenceObjectByHandle` is released exactly once, after the join.
- [ ] `KeSetPriorityThread` uses a priority below 16 unless there is a measured reason to go realtime.
- [ ] Any `KeStackAttachProcess` is balanced by `KeUnstackDetachProcess` on the faulting path as well as the success path, on the same thread.
- [ ] No wait inside the loop is unbounded: every blocking call is either the stop event with a timeout or something that cannot wait on the driver's own unload.
- [ ] Unload order: flag, signal, join, dereference, close handle, delete symbolic link, delete device.
- [ ] No spinlock or mutex is held while waiting on the thread object; the unload path holds nothing the poller needs.
- [ ] The driver can be stopped and started a hundred times without growing any handle count or pool tag.

## Verification

1. Build for x64, sign with the test certificate, `sc create kwthread type= kernel binPath= ...`, then `sc start kwthread`. DebugView must show `KWT: created tid <n> in pid 4` and `KWT: thread <n> running`.
2. Run the client: the first line must show a `tid` matching the debug output and `running=1`; `polls` must grow by roughly ten per second at the 100 ms interval.
3. In the kernel debugger, `!thread <tid>` must show `Cid: 0004.0<tid>`, a `Wait` reason of `WrExecutive`, and a stack ending in `KeWaitForSingleObject` from your routine. `!irql` at a breakpoint inside the routine must print `PASSIVE_LEVEL`.
4. Exercise the affinity IOCTL with a CPU index equal to the processor count; it must fail with `ERROR_INVALID_PARAMETER` (`87`) rather than a bugcheck. With a valid index, `!thread` must report the new ideal processor or affinity.
5. `sc stop kwthread` must return within a second. DebugView must show `KWT: thread <n> stopping` before the unload output, never after.
6. Loop one hundred `sc start` / `sc stop` cycles and compare `!process 4 0` handle counts before and after: the count must return to its starting value. Any upward drift is a `ZwClose` or `ObDereferenceObject` missing on a path.
7. Repeat the cycles with Driver Verifier: `verifier /standard /driver kwthread.sys`, reboot, then run the loop. No `0xC4`, `0xCE`, or `0x9F` may appear; `!verifier 3 kwthread.sys` must report zero outstanding allocations.
8. Force the failure path: temporarily make the poll routine ignore the stop event and confirm that a debug build prints `poll thread stuck; breaking in` and breaks into the debugger instead of unmapping the image. Restore the code afterwards.
