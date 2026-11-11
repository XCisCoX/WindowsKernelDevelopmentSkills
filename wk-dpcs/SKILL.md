---
name: wk-dpcs
description: Use when an ISR or high-IRQL path must defer work, when a periodic timer must complete I/O, or when a DPC_WATCHDOG_VIOLATION (0x133) appears. Covers KDPC lifecycle, KeInitializeDpc, KeSetImportanceDpc, KeSetTargetProcessorDpc, KeInsertQueueDpc, KeRemoveQueueDpc, KeFlushQueuedDpcs, KTIMER with KeSetTimerEx, threaded DPCs, and safe unload.
---

# Deferred procedure calls: the only place high-IRQL work may legally finish

A DPC is a callback the kernel runs at `DISPATCH_LEVEL` on a processor you choose. It exists because interrupt service routines run at device IRQL, where `IoCompleteRequest` and most of the executive are illegal, and because that work must still happen before any thread resumes. The one rule that prevents most DPC disasters: a DPC may touch only non-paged memory, may never wait, and must finish in microseconds, because everything on that processor, including the scheduler and every lower-priority interrupt, is blocked until it returns.

## When to use

- An ISR has captured device status and must complete an IRP, free a resource, or signal an event.
- Periodic work is needed while a device is open, and a dedicated system thread is too heavy.
- A high-resolution or kernel timer needs a callback that runs ahead of ordinary threads.
- A driver must poll a device register because the hardware has no interrupt.
- `0x133 DPC_WATCHDOG_VIOLATION` needs diagnosis and a redesign of the offending routine.
- A driver is being unloaded and queued DPCs or timers must be drained deterministically.
- Do not use for: work that must block, take a mutex, read the registry, or call `Zw*` (`wk-work-items`, `wk-kernel-threads`); lock selection (`wk-spinlocks`); general timing and high-resolution timers (`wk-kernel-timers`).

## Prerequisites and related skills

- Kernel C with `<ntddk.h>`; WDK 10, Windows 7 or later. Threaded DPCs require Windows Vista or later; the DPC watchdog exists from Windows 8.
- The driver must already own a device object, because the DPC's lifetime is tied to it.
- Every structure the DPC touches must live in non-paged pool or in the device extension of a non-pageable device.
- Sibling skills: `wk-irql` (what `DISPATCH_LEVEL` permits), `wk-spinlocks` (locks legal at `DISPATCH_LEVEL`), `wk-kernel-timers` (timer resolution, coalescing, `EX_TIMER`), `wk-irp-queues-cancel` (cancel-safe queues in depth), `wk-work-items` (moving work down to `PASSIVE_LEVEL`), `wk-crash-dump-analysis` (reading `0x133`).

## Mental model

Every processor owns a DPC queue. When an ISR is about to return, or when a thread lowers IRQL toward `PASSIVE_LEVEL`, the kernel checks that queue; if it is non-empty the processor drops to `DISPATCH_LEVEL` and runs the queued routines in order, then continues downward. A DPC therefore never "starts a thread" and never has a thread of its own: it borrows whichever thread was interrupted, which is why it must not assume a process context or a user address space.

```text
  hardware IRQ
      |
      v
  ISR at DIRQL          acknowledge device, capture status, queue DPC     (microseconds)
      |
      v
  IRQL drops to 2       per-processor DPC queue drained in FIFO order
      |
      v
  DPC at DISPATCH       complete IRP, signal event, re-arm timer          (microseconds)
      |
      v
  IRQL drops to 0       interrupted thread resumes (or a new one is scheduled)
```

A timer DPC is the same object with a different producer: the kernel's timer queue inserts the DPC when the `KTIMER` expires, so the driver never calls `KeInsertQueueDpc` for it. A threaded DPC is the same object again, but the kernel may run it in a real system thread at `PASSIVE_LEVEL`, where paging and waiting are legal.

Two properties explain the whole API surface. First, the queue is **per processor**: `KeInsertQueueDpc` targets the current processor unless `KeSetTargetProcessorDpc` moved the DPC, so a DPC can be used to run code on a specific CPU. Second, a `KDPC` is **one queue slot**: it can be queued at most once at a time, and a second insert before the first run returns `FALSE` rather than queuing a second execution.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `KeInitializeDpc(Dpc, Routine, Context)` | Prepare a `KDPC` once | any | Must complete before any insert; never re-init while queued |
| `KeInitializeThreadedDpc(Dpc, Routine, Context)` | Prepare a threaded DPC | any | Vista or later |
| `KeSetImportanceDpc(Dpc, Importance)` | Queue position | any | `LowImportance`, `MediumImportance`, `HighImportance` |
| `KeSetTargetProcessorDpc(Dpc, Cpu)` | Pin to one processor | any | Ignoring it means "current processor" at insert time |
| `KeInsertQueueDpc(Dpc, Arg1, Arg2)` | Queue for execution | any | `FALSE` if already queued; args surface in the routine |
| `KeRemoveQueueDpc(Dpc)` | Dequeue if not started | `<= DISPATCH` | `FALSE` if it already ran or is running |
| `KeFlushQueuedDpcs()` | Wait for queued DPCs | `PASSIVE` | System-wide barrier; can be slow |
| `KeInitializeTimer(Timer)` | One-shot or periodic timer | any | Notification timer |
| `KeInitializeTimerEx(Timer, Type)` | Choose timer type | any | `NotificationTimer` or `SynchronizationTimer` |
| `KeSetTimer(Timer, DueTime, Dpc)` | Arm one-shot | `<= DISPATCH` | Negative due time is relative |
| `KeSetTimerEx(Timer, DueTime, Period, Dpc)` | Arm, optionally periodic | `<= DISPATCH` | `Period` in milliseconds, `0` = one-shot |
| `KeCancelTimer(Timer)` | Dequeue the timer | `<= DISPATCH` | `TRUE` only if it was still pending |
| `KeReadStateTimer(Timer)` | Signaled state | `<= DISPATCH` | Nonzero while armed |

`KDPC` fields that callers may rely on: `DeferredRoutine`, `DeferredContext`, `Importance`, and the union that carries the target processor. Treat the rest as opaque, allocate the structure from non-paged memory, and keep it alive for as long as it can be queued.

`KTIMER` is a dispatcher object: it can also be waited on with `KeWaitForSingleObject`, but a DPC callback is the cheaper and more common pattern.

## Step-by-step

1. **Decide the producer.** Interrupt-driven work is queued by the ISR with `KeInsertQueueDpc`. Clock-driven work uses `KeSetTimer`/`KeSetTimerEx`. If the work can block, stop here and use a work item or a system thread instead.
2. **Allocate and initialize once.** Put the `KDPC` and `KTIMER` in the device extension (which is non-paged for a normal device) or allocate them with `ExAllocatePool2(POOL_FLAG_NON_PAGED, Size, Tag)` and keep a pointer. Call `KeInitializeDpc`, then `KeSetImportanceDpc` and, if needed, `KeSetTargetProcessorDpc`; initialize the timer with `KeInitializeTimer`.
3. **Queue it.** `KeInsertQueueDpc(&Dpc, Arg1, Arg2)` returns `FALSE` when the DPC is already queued. Treat `FALSE` as "someone else already scheduled this work", not as an error to retry in a loop.
4. **Keep the routine short.** Bound every loop, complete a fixed batch, and re-queue or hand off for anything longer. No waits, no paged memory, no `Zw*`, no user pointers, no exceptions that could escape.
5. **Guard shared state with a spinlock.** The DPC runs concurrently with dispatch code on other processors and with any ISR that shares the device lock. `KeAcquireSpinLock` is legal and does not change the level when already at `DISPATCH`.
6. **Handle teardown explicitly.** Cancel the timer, flush queued DPCs at `PASSIVE_LEVEL`, then drain and complete every pending IRP with `STATUS_CANCELLED` before deleting the device.
7. **Verify with the watchdog in mind.** Any DPC that takes longer than roughly a hundred microseconds is a design smell; one that takes tens of milliseconds risks `0x133` on a busy machine.

## Complete example

A legacy WDM driver with a periodic timer DPC that completes queued `IOCTL_DPCT_READ` requests, plus a threaded-DPC variant selected by a second IOCTL. The queue is a cancel-safe queue (`IO_CSQ`), so cancellation and teardown are handled by the kernel's CSQ machinery.

```c
//
// dpctimer.c -- periodic timer DPC that completes pending IRPs.
//
#include <ntddk.h>

#define DPCT_TAG        'pTfD'
#define DPCT_DEVNAME    L"\\Device\\DpcTimer"
#define DPCT_LINK       L"\\DosDevices\\DpcTimer"
#define DPCT_PERIOD_MS  100
#define DPCT_BATCH      8

#define IOCTL_DPCT_ARM \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x900, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_DPCT_READ \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x901, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_DPCT_STATS \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x902, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _DPCT_EXTENSION {
    KSPIN_LOCK      QueueLock;
    LIST_ENTRY      PendingList;
    IO_CSQ          Csq;
    KTIMER          Timer;
    KDPC            TimerDpc;
    KDPC            ThreadDpc;
    volatile LONG   PendingCount;
    volatile LONG   Sequence;
    volatile LONG   Mode;            /* 0 = timer DPC, 1 = threaded DPC */
    volatile LONG   Stopping;
    ULONG           PeriodMs;
} DPCT_EXTENSION, *PDPCT_EXTENSION;

/* ---------- cancel-safe queue callbacks ---------- */

static VOID CsqInsertIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    PDPCT_EXTENSION ext = CONTAINING_RECORD(Csq, DPCT_EXTENSION, Csq);

    InsertTailList(&ext->PendingList, &Irp->Tail.Overlay.ListEntry);
}

static VOID CsqRemoveIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(Csq);
    RemoveEntryList(&Irp->Tail.Overlay.ListEntry);
}

static PIRP CsqPeekNextIrp(_In_ PIO_CSQ Csq, _In_opt_ PIRP Irp,
                           _In_opt_ PVOID PeekContext)
{
    PDPCT_EXTENSION ext = CONTAINING_RECORD(Csq, DPCT_EXTENSION, Csq);
    PLIST_ENTRY     next;

    UNREFERENCED_PARAMETER(PeekContext);
    next = (Irp == NULL) ? ext->PendingList.Flink
                         : Irp->Tail.Overlay.ListEntry.Flink;
    if (next == &ext->PendingList) {
        return NULL;
    }
    return CONTAINING_RECORD(next, IRP, Tail.Overlay.ListEntry);
}

static VOID CsqAcquireLock(_In_ PIO_CSQ Csq, _Out_ PKIRQL Irql)
{
    PDPCT_EXTENSION ext = CONTAINING_RECORD(Csq, DPCT_EXTENSION, Csq);

    KeAcquireSpinLock(&ext->QueueLock, Irql);
}

static VOID CsqReleaseLock(_In_ PIO_CSQ Csq, _In_ KIRQL Irql)
{
    PDPCT_EXTENSION ext = CONTAINING_RECORD(Csq, DPCT_EXTENSION, Csq);

    KeReleaseSpinLock(&ext->QueueLock, Irql);
}

static VOID CsqCompleteCanceledIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    PDPCT_EXTENSION ext = CONTAINING_RECORD(Csq, DPCT_EXTENSION, Csq);

    Irp->IoStatus.Status = STATUS_CANCELLED;
    Irp->IoStatus.Information = 0;
    InterlockedDecrement(&ext->PendingCount);
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
}

/* ---------- shared completion path, safe at DISPATCH_LEVEL ---------- */

/*
 * METHOD_BUFFERED guarantees that SystemBuffer is non-paged, which is the only
 * reason writing to it here is legal. A METHOD_NEITHER UserBuffer would not be.
 */
static VOID CompleteOne(_Inout_ PDPCT_EXTENSION Ext, _Inout_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    PULONG             out = (PULONG)Irp->AssociatedIrp.SystemBuffer;
    ULONG              length = stack->Parameters.DeviceIoControl.OutputBufferLength;

    if (out != NULL && length >= sizeof(ULONG)) {
        *out = (ULONG)InterlockedIncrement(&Ext->Sequence);
        Irp->IoStatus.Information = sizeof(ULONG);
        Irp->IoStatus.Status = STATUS_SUCCESS;
    } else {
        Irp->IoStatus.Information = 0;
        Irp->IoStatus.Status = STATUS_BUFFER_TOO_SMALL;
    }

    InterlockedDecrement(&Ext->PendingCount);
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
}

static ULONG DrainQueue(_Inout_ PDPCT_EXTENSION Ext, _In_ ULONG Budget)
{
    PIRP  irp;
    ULONG done = 0;

    while (done < Budget) {
        irp = IoCsqRemoveNextIrp(&Ext->Csq, NULL);
        if (irp == NULL) {
            break;
        }
        CompleteOne(Ext, irp);
        done++;
    }
    return done;
}

/* ---------- the two deferred routines ---------- */

static VOID TimerDpcRoutine(_In_ PKDPC Dpc, _In_opt_ PVOID Context,
                            _In_opt_ PVOID SystemArgument1,
                            _In_opt_ PVOID SystemArgument2)
{
    PDPCT_EXTENSION ext = (PDPCT_EXTENSION)Context;

    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(SystemArgument1);
    UNREFERENCED_PARAMETER(SystemArgument2);

    NT_ASSERT(KeGetCurrentIrql() == DISPATCH_LEVEL);
    (VOID)DrainQueue(ext, DPCT_BATCH);
}

/*
 * LowImportance asks the kernel to run this in a system thread at PASSIVE_LEVEL.
 * That is a request, not a guarantee: under pressure, or with a higher importance,
 * the same routine can run at DISPATCH_LEVEL. Branch on the actual level.
 */
static VOID ThreadedDpcRoutine(_In_ PKDPC Dpc, _In_opt_ PVOID Context,
                               _In_opt_ PVOID SystemArgument1,
                               _In_opt_ PVOID SystemArgument2)
{
    PDPCT_EXTENSION ext = (PDPCT_EXTENSION)Context;

    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(SystemArgument1);
    UNREFERENCED_PARAMETER(SystemArgument2);

    if (KeGetCurrentIrql() >= DISPATCH_LEVEL) {
        (VOID)DrainQueue(ext, DPCT_BATCH);
        return;
    }

    /* PASSIVE_LEVEL path: paging, fast mutexes and short waits are allowed. */
    (VOID)DrainQueue(ext, DPCT_BATCH * 4);
}

/* ---------- dispatch and lifecycle ---------- */

static NTSTATUS DpctCreateClose(_In_ PDEVICE_OBJECT DeviceObject,
                                _Inout_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS DpctControl(_In_ PDEVICE_OBJECT DeviceObject, _Inout_ PIRP Irp)
{
    PDPCT_EXTENSION    ext = (PDPCT_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    ULONG              code = stack->Parameters.DeviceIoControl.IoControlCode;
    ULONG              outLen = stack->Parameters.DeviceIoControl.OutputBufferLength;
    ULONG              inLen = stack->Parameters.DeviceIoControl.InputBufferLength;
    PULONG             buffer = (PULONG)Irp->AssociatedIrp.SystemBuffer;
    NTSTATUS           status = STATUS_INVALID_DEVICE_REQUEST;
    LARGE_INTEGER      due;
    ULONG              info = 0;

    if (InterlockedCompareExchange(&ext->Stopping, 0, 0) != 0) {
        status = STATUS_DELETE_PENDING;
        goto done;
    }

    switch (code) {
    case IOCTL_DPCT_ARM:
        if (buffer == NULL || inLen < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        if (*buffer > 1) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        InterlockedExchange(&ext->Mode, (LONG)*buffer);
        due.QuadPart = -(LONGLONG)ext->PeriodMs * 10000;   /* relative, 100 ns */
        KeSetTimerEx(&ext->Timer, due, (LONG)ext->PeriodMs,
                     (*buffer == 0) ? &ext->TimerDpc : &ext->ThreadDpc);
        status = STATUS_SUCCESS;
        break;

    case IOCTL_DPCT_READ:
        if (outLen < sizeof(ULONG)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        IoMarkIrpPending(Irp);
        status = IoCsqInsertIrp(&ext->Csq, Irp, NULL);
        if (!NT_SUCCESS(status)) {
            return status;          /* already canceled; the CSQ completed it */
        }
        InterlockedIncrement(&ext->PendingCount);
        return STATUS_PENDING;

    case IOCTL_DPCT_STATS:
        if (outLen < sizeof(ULONG) || buffer == NULL) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        *buffer = (ULONG)InterlockedCompareExchange(&ext->PendingCount, 0, 0);
        info = sizeof(ULONG);
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

static VOID DpctUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT    device = DriverObject->DeviceObject;
    PDPCT_EXTENSION   ext = (PDPCT_EXTENSION)device->DeviceExtension;
    UNICODE_STRING    link;
    PIRP              irp;

    RtlInitUnicodeString(&link, DPCT_LINK);
    IoDeleteSymbolicLink(&link);

    InterlockedExchange(&ext->Stopping, 1);

    KeCancelTimer(&ext->Timer);     /* stops new timer-driven insertions */
    KeFlushQueuedDpcs();            /* PASSIVE: wait for a DPC already running */

    while ((irp = IoCsqRemoveNextIrp(&ext->Csq, NULL)) != NULL) {
        irp->IoStatus.Status = STATUS_CANCELLED;
        irp->IoStatus.Information = 0;
        InterlockedDecrement(&ext->PendingCount);
        IoCompleteRequest(irp, IO_NO_INCREMENT);
    }

    IoDeleteDevice(device);
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT   device = NULL;
    PDPCT_EXTENSION  ext;
    UNICODE_STRING   devName;
    UNICODE_STRING   link;
    NTSTATUS         status;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, DPCT_DEVNAME);
    status = IoCreateDevice(DriverObject, sizeof(DPCT_EXTENSION), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PDPCT_EXTENSION)device->DeviceExtension;
    KeInitializeSpinLock(&ext->QueueLock);
    InitializeListHead(&ext->PendingList);
    ext->PendingCount = 0;
    ext->Sequence = 0;
    ext->Mode = 0;
    ext->Stopping = 0;
    ext->PeriodMs = DPCT_PERIOD_MS;

    status = IoCsqInitialize(&ext->Csq, CsqInsertIrp, CsqRemoveIrp,
                             CsqPeekNextIrp, CsqAcquireLock, CsqReleaseLock,
                             CsqCompleteCanceledIrp);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    KeInitializeTimer(&ext->Timer);
    KeInitializeDpc(&ext->TimerDpc, TimerDpcRoutine, ext);
    KeSetImportanceDpc(&ext->TimerDpc, MediumImportance);

    KeInitializeThreadedDpc(&ext->ThreadDpc, ThreadedDpcRoutine, ext);
    KeSetImportanceDpc(&ext->ThreadDpc, LowImportance);

    RtlInitUnicodeString(&link, DPCT_LINK);
    status = IoCreateSymbolicLink(&link, &devName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = DpctCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = DpctCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DpctControl;
    DriverObject->DriverUnload = DpctUnload;

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

Client behavior: send `IOCTL_DPCT_ARM` with a `ULONG` of `0` or `1`, then issue `IOCTL_DPCT_READ` several times. Each read returns `STATUS_PENDING` from `DeviceIoControl` as `ERROR_IO_PENDING` and completes within one period with an incrementing `ULONG`. A read issued before arming simply stays queued until something arms the timer.

## Variations and advanced use

- **ISR to DPC handoff.** The ISR stays minimal: read status, acknowledge, queue, return `TRUE`. The arguments passed to `KeInsertQueueDpc` arrive as `SystemArgument1` and `SystemArgument2` and are the cheapest way to move a captured register value to the DPC.

```c
static BOOLEAN DeviceIsr(_In_ PKINTERRUPT Interrupt, _In_opt_ PVOID Context)
{
    PISR_CONTEXT ctx = (PISR_CONTEXT)Context;
    ULONG        status;

    UNREFERENCED_PARAMETER(Interrupt);

    status = READ_REGISTER_ULONG(&ctx->Registers->Status);
    if ((status & DEVICE_IRQ_ACTIVE) == 0) {
        return FALSE;                       /* not ours; do not queue work */
    }
    WRITE_REGISTER_ULONG(&ctx->Registers->Status, status);   /* acknowledge */
    KeInsertQueueDpc(&ctx->IsrDpc, (PVOID)(ULONG_PTR)status, NULL);
    return TRUE;
}
```

  Only one `KDPC` slot exists per interrupt source: if a second interrupt arrives while the DPC is still queued, `KeInsertQueueDpc` returns `FALSE` and the queued arguments are not updated. Design the DPC to re-read device state rather than trusting the arguments to be the latest interrupt.

- **Pinning to a processor.** `KeSetTargetProcessorDpc(&ext->TimerDpc, (CCHAR)0)` runs the DPC on CPU 0. This is useful for a per-processor cache or for hardware whose registers must be touched from one CPU, but it concentrates all of that work on one processor. `KeSetTargetProcessorDpcEx` accepts an affinity mask when a set of processors is acceptable.

- **One-shot chaining instead of a periodic timer.** A periodic `KeSetTimerEx` is cheaper than re-arming from the DPC, but a DPC that re-arms its own timer with `KeSetTimer` gives finer control over the next due time and avoids drift. If you re-arm, do it before returning from the DPC so the timer cannot be cancelled in the gap.

- **Importance.** `HighImportance` places the DPC at the head of the queue, so it runs before other work on that processor. It is legitimate for a device that is about to overflow a buffer and abusive elsewhere; `MediumImportance` is the default and should be the default in your code too.

- **Threaded DPC trade-offs.** A threaded DPC at `LowImportance` can page, allocate, and take a fast mutex, which makes it a middle ground between a DPC and a work item with lower latency than queueing to a worker thread. Its routine must still be safe if the kernel decides to run it at `DISPATCH_LEVEL`, and it must not assume a user context. For anything that blocks for long, use `wk-work-items` instead.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `0x133` with parameter 1 = 0 | One DPC or ISR exceeded its time budget | `!analyze -v`, then `!dpcs`; shorten the routine and hand off the rest |
| `0x133` with parameter 1 = 1 | Cumulative time at `DISPATCH_LEVEL` too high across the system | Look for a busy-polling DPC or a spinlock held too long |
| `0xA` inside a DPC, arg2 = 2 | Paged memory or a bad pointer touched at `DISPATCH_LEVEL` | `!analyze -v`; move the data into non-paged memory |
| `0xD1` with a driver frame in the DPC | Same fault attributed to your module | Check the pointer's allocation tag with `!pool` |
| Timer never fires | Positive `DueTime` treated as absolute time in 1601, or `Period` left at 0 | Use a negative `DueTime` for a relative delay |
| Timer fires once then stops | `KeSetTimer` used where `KeSetTimerEx` with a period was needed | Re-arm explicitly or pass `Period` |
| `KeCancelTimer` returns `FALSE` and the DPC still runs | The timer expired and the DPC was already queued | Cancel, then `KeFlushQueuedDpcs`, then drain |
| `0xCE` after unload | Pending IRPs were still queued when the driver unloaded | Drain and complete the queue before `IoDeleteDevice` |
| `0x50` or `0x1E` after unload, module missing in `lm` | A queued DPC executed code from the unloaded image | `KeCancelTimer` plus `KeFlushQueuedDpcs` before returning from unload |
| Client hangs on a read | The dispatch returned `STATUS_PENDING` but the queue was drained without completing that IRP | `!irpfind` for outstanding IRPs; check every drain path |
| `KeFlushQueuedDpcs` never returns | A DPC is blocked, usually on a lock held by a thread that is waiting for the flush | `!stacks 2`, `!dpcs`; never block inside a DPC |

## Safety, IRQL, and cleanup checklist

- [ ] `KDPC` and `KTIMER` live in non-paged memory and outlive every possible queue operation.
- [ ] `KeInitializeDpc` is called once, before the first insert, and never while the object is queued.
- [ ] The DPC routine touches no paged pool, no pageable code, no user buffer, no registry, and no `Zw*` call.
- [ ] No wait, mutex, `ERESOURCE`, or file I/O inside a DPC; spinlocks only.
- [ ] Every loop in the DPC is bounded; the routine completes a batch and returns.
- [ ] Shared state is protected by a spinlock that the ISR, the DPC, and the dispatch paths all honor.
- [ ] `KeInsertQueueDpc` return value is either handled or deliberately ignored with a comment explaining why.
- [ ] Unload order: stop accepting I/O, `KeCancelTimer`, `KeFlushQueuedDpcs`, drain the queue with `STATUS_CANCELLED`, delete the symbolic link, delete the device.
- [ ] No DPC can re-queue itself after the flush; if one can, add a `Stopping` flag checked before every insert.
- [ ] Threaded DPCs branch on `KeGetCurrentIrql()` rather than assuming `PASSIVE_LEVEL`.

## Verification

1. Build for x64 in Debug and install with your test certificate; start the service.
2. Arm the timer and confirm completions: a client loop of eight overlapped reads must complete within about `8 * 100 ms` and return increasing sequence numbers.
3. Switch to threaded mode with `IOCTL_DPCT_ARM` and a value of `1`, and confirm reads still complete. In the debugger, break in `ThreadedDpcRoutine` and run `!irql`; with `LowImportance` on an idle system it should print `PASSIVE_LEVEL`, and the routine must still behave if it prints `DISPATCH_LEVEL`.
4. Check the queue is empty at rest: `IOCTL_DPCT_STATS` must return `0` when no read is outstanding, and exactly the number of outstanding reads otherwise.
5. Cancel storm: issue 64 overlapped reads, cancel half with `CancelIoEx`, and confirm the canceling reads return `ERROR_OPERATION_ABORTED` while the rest succeed. No IRP may remain: `!irpfind` filtered to your device must show none.
6. Unload under load: with reads pending, stop the service. The unload must complete without a bugcheck and the pending reads must fail with `ERROR_OPERATION_ABORTED` or `STATUS_CANCELLED`.
7. Watchdog check: enable verifier with I/O verification (`verifier /flags 0x10 /driver dpctimer.sys`), soak for ten minutes on a four-processor VM, and confirm no `0x133` appears in the event log.
8. In the debugger, `!dpcs` on each processor must show an empty queue when the driver is idle; a DPC still listed while idle means it is being re-queued in a loop.
