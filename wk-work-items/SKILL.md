---
name: wk-work-items
description: Use when a DPC, ISR, or completion routine must run blocking work at PASSIVE_LEVEL: IoAllocateWorkItem, IoQueueWorkItem, IoQueueWorkItemEx, IoFreeWorkItem, the deprecated ExQueueWorkItem, work item lifetime and drain-on-unload with an event and counter, and diagnosing 0x9F, 0xCE, 0xC2 and use-after-free crashes with !exqueue and !stacks.
---

# Work items: getting back to PASSIVE_LEVEL without losing the driver

A DPC runs at `DISPATCH_LEVEL` where the registry, the file system, and every wait are
forbidden, yet most real work a device needs — reading configuration, writing a log, calling
a helper driver, blocking on hardware — must happen at `PASSIVE_LEVEL`. A work item is the
kernel's answer: you hand a routine and a context to the system thread pool and a worker
thread runs it at `PASSIVE_LEVEL` with a normal thread context. The mechanism is trivial to
start and easy to get catastrophically wrong, because the work item can execute long after
the code that queued it has returned, and possibly after the device is gone. The rule that
prevents most disasters: the work item, its context, and the device object must all be
guaranteed to outlive the queued execution, and the only proof of that guarantee is an
explicit drain in the unload path.

## When to use

- A DPC or ISR has data ready and something must read the registry, a file, or another
  device to finish the request.
- An IRP must be completed after blocking work, and completing it from a DPC would require
  calls that are illegal at `DISPATCH_LEVEL`.
- A dispatch routine wants to return quickly and do slow work on a worker thread.
- A driver needs bounded background work without owning a thread (contrast
  `wk-kernel-threads` for long-lived polling).
- You are reviewing an unload path that can race with queued work.
- A crash shows a driver's routine running after unload, or a worker thread faulting on
  freed memory.

Do not use for: work that fits in a few microseconds at `DISPATCH_LEVEL` (`wk-dpcs`), a
long-lived polling thread that must own a stack and a stop event (`wk-kernel-threads`), or
anything that must run in the caller's process context, which a system worker never has.

## Prerequisites and related skills

- `<wdm.h>` or `<ntddk.h>` provides `IO_WORKITEM`, `IoAllocateWorkItem`, `IoQueueWorkItem`,
  `IoQueueWorkItemEx`, `IoFreeWorkItem`, `IoInitializeWorkItem`, `IoUninitializeWorkItem`,
  `IoSizeofWorkItem`, and `WORK_QUEUE_TYPE`.
- `IoQueueWorkItem` may be called at any IRQL up to `DISPATCH_LEVEL`; the routine always
  runs at `PASSIVE_LEVEL` in a system worker thread.
- The context structure must be non-paged if it can be touched before the worker runs, and
  it must not be freed until the routine has returned.
- Related: `wk-dpcs` (the usual producer of work items), `wk-kernel-threads` (when a worker
  thread pool item is the wrong tool), `wk-synchronization` (events and counters used for
  draining), `wk-registry-programming` (what the worker usually does),
  `wk-irp-queues-cancel` (pending IRP lifetime).

## Mental model

A work item has three states and one owner:

```
  allocated ──IoQueueWorkItem──► queued ──worker picks it up──► running
      ▲                            │                              │
      │                            └──── drain waits here ────────┘
      └──────────────── IoFreeWorkItem (only when neither queued nor running)
```

The queue is the system thread pool, shared with every other driver on the machine. A
worker thread has a normal kernel stack, runs at `PASSIVE_LEVEL`, and has **system process
context** — never the context of the client that sent the IRP. That single fact rules out
touching user-mode addresses, assuming `PsGetCurrentProcess()` is the caller, and relying on
per-thread state.

`IoQueueWorkItem` differs from its relatives in one decisive way: the work item was created
with `IoAllocateWorkItem(DeviceObject)`, so the I/O manager holds a reference on that device
object for the life of the item. The device object cannot be deleted underneath the routine,
and for a PnP driver the device cannot be removed while a work item still references it, so
the driver image stays loaded. That reference protects the *device object*, not your
bookkeeping: `DriverUnload` for a legacy driver still runs when the service is stopped, and
the code pages are unmapped when it returns. You must drain.

```c
// the contract in one line: allocate once, queue many times, free after the last run
dx->WorkItem = IoAllocateWorkItem(dev);          // holds a reference on dev
IoQueueWorkItem(dx->WorkItem, Routine, DelayedWorkQueue, context);
IoFreeWorkItem(dx->WorkItem);                    // only after Routine has returned
```

The older `ExQueueWorkItem` path takes a raw `WORK_QUEUE_ITEM` and associates it with
nothing. Nothing keeps the driver loaded, nothing tells you when the routine finished, and
the queue type is a hint rather than a priority contract. It is documented as deprecated;
every modern use has an `Io` equivalent.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `IoAllocateWorkItem(DeviceObject)` | allocate a work item bound to a device | ≤ `DISPATCH_LEVEL` | non-paged; `NULL` on failure; allocate once at init |
| `IoFreeWorkItem(WorkItem)` | release it | `PASSIVE_LEVEL` | only when neither queued nor running |
| `IoQueueWorkItem(WorkItem, Routine, QueueType, Context)` | queue for `PASSIVE_LEVEL` execution | ≤ `DISPATCH_LEVEL` | routine gets the device object and the context |
| `IoQueueWorkItemEx(WorkItem, RoutineEx, QueueType, Context)` | same, routine also receives the work item | ≤ `DISPATCH_LEVEL` | lets the routine free itself |
| `IoSizeofWorkItem()` | size for a driver-allocated work item | any | pair with `IoInitializeWorkItem` |
| `IoInitializeWorkItem(DeviceObject, WorkItem)` | initialize storage you allocated | `PASSIVE_LEVEL` | no allocation, no failure |
| `IoUninitializeWorkItem(WorkItem)` | release driver-allocated storage | `PASSIVE_LEVEL` | after the routine has finished |
| `IO_WORKITEM_ROUTINE` | `void (*)(PDEVICE_OBJECT, PVOID)` | runs at `PASSIVE_LEVEL` | the classic callback shape |
| `IO_WORKITEM_ROUTINE_EX` | `void (*)(PDEVICE_OBJECT, PVOID, PIO_WORKITEM)` | runs at `PASSIVE_LEVEL` | third argument is the item itself |
| `ExInitializeWorkItem(Item, Routine, Context)` | legacy initialization | `PASSIVE_LEVEL` | **deprecated** |
| `ExQueueWorkItem(Item, QueueType)` | legacy queueing | ≤ `DISPATCH_LEVEL` | **deprecated**: no device association, no lifetime |

Queue types accepted by both queueing calls:

| `WORK_QUEUE_TYPE` | Worker priority | Use when |
|---|---|---|
| `CriticalWorkQueue` | 13 | the item unblocks I/O that a client is waiting on |
| `DelayedWorkQueue` | 12 | ordinary background work; the usual choice |
| `HyperCriticalWorkQueue` | 15 | the item must preempt other work to keep the device alive |

Other values exist in the enumeration (`NormalWorkQueue`, `BackgroundWorkQueue`,
`RealTimeWorkQueue`, and `CustomPriorityWorkQueue`), but the documented contract for these
calls is one of the three above; passing anything else is unsupported and fails verifier
checks on some builds.

What the routine may and may not assume:

| Assumption | True? | Consequence if wrong |
|---|---|---|
| runs at `PASSIVE_LEVEL` | yes | registry, files, waits, and `Zw*` calls are legal |
| runs in the requesting process context | **no** | user pointers fault; `PsGetCurrentProcess` is the system process |
| runs on the processor that queued it | **no** | per-processor data must be indexed, not assumed |
| runs exactly once per queue call | yes | but it may run *after* the IRP was cancelled |
| a worker thread is always available | **no** | blocked workers starve the whole machine's pool |

## Step-by-step

1. **Decide the work really needs `PASSIVE_LEVEL`.** Check: if the routine would only touch
   non-paged memory and registers, a DPC is cheaper and cannot be starved.
2. **Allocate the work item at device add**, not per request: `IoAllocateWorkItem(dev)`,
   check for `NULL`, and store it in the extension. Check: the allocation happens before any
   path can queue it.
3. **Define a context that outlives the queue call** — usually the `PIRP` itself, or a
   non-paged structure the driver owns. Never pass a pointer to a stack frame of the routine
   that queues the item.
4. **Increment a counter before queueing**, under the same lock that the unload path uses,
   so the drain cannot observe zero while an item is about to be queued.
5. **Queue from the DPC** with `IoQueueWorkItem(..., DelayedWorkQueue, context)`. Check: the
   DPC does no blocking work itself and returns immediately.
6. **Do the blocking work in the routine**, then complete the IRP or signal the event, then
   decrement the counter, then set the drain event if it reached zero. Check: nothing touches
   the context after the decrement.
7. **On unload, in this order**: set a stopping flag, cancel the timer, `KeFlushQueuedDpcs()`,
   cancel any IRP still pending, wait for the counter to reach zero, `IoFreeWorkItem`, then
   `IoDeleteDevice`. Check: every one of those steps is present and in that order.
8. **Prove it with verifier and a stress client** that opens and closes the device while
   requests are in flight.

## Complete example: a DPC queues a work item that reads the registry and completes an IRP

The client sends an IOCTL asking for a timeout value. The dispatch routine parks the IRP and
arms a one-shot timer that stands in for the device asserting an interrupt; the resulting
DPC cannot read the registry, so it queues a work item. The worker opens the service's
`Parameters` key, reads `TimeoutMs`, completes the IRP, and signals the drain event when the
last item finishes.

```c
// WorkItemDemo.c  —  DPC to work item to registry to IRP completion
#include <ntddk.h>

#define WID_TAG        'diWW'
#define WID_TIMEOUT_MS 50

typedef struct _DEVICE_EXTENSION {
    KSPIN_LOCK      Lock;           // protects PendingIrp
    PIRP            PendingIrp;
    KDPC            SignalDpc;
    KTIMER          SignalTimer;
    PIO_WORKITEM    WorkItem;
    KEVENT          Drained;        // notification, set when ActiveWork hits zero
    LONG            ActiveWork;
    volatile LONG   Stopping;
    PDEVICE_OBJECT  Self;
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

static NTSTATUS ReadTimeoutFromRegistry(PULONG TimeoutMs)
{
    UNICODE_STRING path = RTL_CONSTANT_STRING(
        L"\\Registry\\Machine\\System\\CurrentControlSet\\Services\\WorkItemDemo\\Parameters");
    UNICODE_STRING valueName = RTL_CONSTANT_STRING(L"TimeoutMs");
    OBJECT_ATTRIBUTES attributes;
    UCHAR buffer[sizeof(KEY_VALUE_PARTIAL_INFORMATION) + sizeof(ULONG)];
    PKEY_VALUE_PARTIAL_INFORMATION info = (PKEY_VALUE_PARTIAL_INFORMATION)buffer;
    HANDLE key = NULL;
    ULONG length = 0;
    NTSTATUS status;

    InitializeObjectAttributes(&attributes, &path,
                               OBJ_CASE_INSENSITIVE | OBJ_KERNEL_HANDLE, NULL, NULL);
    status = ZwOpenKey(&key, KEY_READ, &attributes);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    status = ZwQueryValueKey(key, &valueName, KeyValuePartialInformation,
                             info, sizeof(buffer), &length);
    if (NT_SUCCESS(status)) {
        if (info->Type == REG_DWORD && info->DataLength == sizeof(ULONG)) {
            RtlCopyMemory(TimeoutMs, info->Data, sizeof(ULONG));
        } else {
            status = STATUS_INVALID_PARAMETER;
        }
    }
    ZwClose(key);
    return status;
}

static VOID ReadTimeoutWorker(PDEVICE_OBJECT DeviceObject, PVOID Context)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIRP irp = (PIRP)Context;
    ULONG value = 0;
    NTSTATUS status;

    status = ReadTimeoutFromRegistry(&value);         // PASSIVE_LEVEL only
    if (NT_SUCCESS(status)) {
        *(PULONG)irp->AssociatedIrp.SystemBuffer = value;
        irp->IoStatus.Information = sizeof(ULONG);
    } else {
        irp->IoStatus.Information = 0;
    }
    irp->IoStatus.Status = status;
    IoCompleteRequest(irp, IO_NO_INCREMENT);

    if (0 == InterlockedDecrement(&dx->ActiveWork)) {
        KeSetEvent(&dx->Drained, IO_NO_INCREMENT, FALSE);
    }
}
```

The DPC runs at `DISPATCH_LEVEL`: it takes the IRP out of the extension under the lock,
counts it, and hands it to the pool. Everything that can block happens after that.

```c
static VOID SignalDpcRoutine(PKDPC Dpc, PVOID Context, PVOID Arg1, PVOID Arg2)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)Context;
    KLOCK_QUEUE_HANDLE handle;
    PIRP irp;

    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(Arg1);
    UNREFERENCED_PARAMETER(Arg2);

    KeAcquireInStackQueuedSpinLockAtDpcLevel(&dx->Lock, &handle);
    irp = dx->PendingIrp;
    dx->PendingIrp = NULL;
    if (irp != NULL) {
        InterlockedIncrement(&dx->ActiveWork);        // counted before it can run
    }
    KeReleaseInStackQueuedSpinLockFromDpcLevel(&handle);

    if (irp != NULL) {
        IoQueueWorkItem(dx->WorkItem, ReadTimeoutWorker, DelayedWorkQueue, irp);
    }
}

#define IOCTL_WID_GET_TIMEOUT \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0xA00, METHOD_BUFFERED, FILE_READ_DATA)

static NTSTATUS DispatchDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    LARGE_INTEGER due;
    KLOCK_QUEUE_HANDLE handle;
    NTSTATUS status;

    if (sp->Parameters.DeviceIoControl.IoControlCode != IOCTL_WID_GET_TIMEOUT) {
        status = STATUS_INVALID_DEVICE_REQUEST;
        goto Complete;
    }
    if (InterlockedCompareExchange(&dx->Stopping, 0, 0) != 0) {
        status = STATUS_DELETE_PENDING;      // unload has begun: take no new work
        goto Complete;
    }
    if (sp->Parameters.DeviceIoControl.OutputBufferLength < sizeof(ULONG)) {
        status = STATUS_BUFFER_TOO_SMALL;
        goto Complete;
    }

    KeAcquireInStackQueuedSpinLock(&dx->Lock, &handle);
    if (dx->PendingIrp != NULL) {
        KeReleaseInStackQueuedSpinLock(&handle);
        status = STATUS_DEVICE_BUSY;
        goto Complete;
    }
    dx->PendingIrp = Irp;
    KeReleaseInStackQueuedSpinLock(&handle);

    IoMarkIrpPending(Irp);
    due.QuadPart = -(LONGLONG)WID_TIMEOUT_MS * 10000;   // 100 ns units, relative
    KeSetTimer(&dx->SignalTimer, due, &dx->SignalDpc);
    return STATUS_PENDING;

Complete:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

Unload is the interesting part. It stops new work first, flushes DPCs so nothing else can
queue, cancels a still-parked IRP, and only then waits on the counter.

```c
static VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\WorkItemDemo");
    PDEVICE_OBJECT dev = DriverObject->DeviceObject;

    IoDeleteSymbolicLink(&linkName);
    while (dev != NULL) {
        PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)dev->DeviceExtension;
        PDEVICE_OBJECT next = dev->NextDevice;
        KLOCK_QUEUE_HANDLE handle;
        PIRP parked;
        LARGE_INTEGER oneSecond;

        InterlockedExchange(&dx->Stopping, 1);
        KeCancelTimer(&dx->SignalTimer);
        KeFlushQueuedDpcs();                 // no DPC can queue work after this line

        KeAcquireInStackQueuedSpinLock(&dx->Lock, &handle);
        parked = dx->PendingIrp;
        dx->PendingIrp = NULL;
        KeReleaseInStackQueuedSpinLock(&handle);
        if (parked != NULL) {
            parked->IoStatus.Status = STATUS_CANCELLED;
            parked->IoStatus.Information = 0;
            IoCompleteRequest(parked, IO_NO_INCREMENT);
        }

        oneSecond.QuadPart = -10 * 1000 * 1000;
        while (InterlockedCompareExchange(&dx->ActiveWork, 0, 0) != 0) {
            KeWaitForSingleObject(&dx->Drained, Executive, KernelMode, FALSE, &oneSecond);
            KeClearEvent(&dx->Drained);
            DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_WARNING_LEVEL,
                       "WorkItemDemo: still draining %ld item(s)\n",
                       InterlockedCompareExchange(&dx->ActiveWork, 0, 0));
        }

        if (dx->WorkItem != NULL) {
            IoFreeWorkItem(dx->WorkItem);    // routine has returned, item is idle
            dx->WorkItem = NULL;
        }
        IoDeleteDevice(dev);
        dev = next;
    }
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING devName  = RTL_CONSTANT_STRING(L"\\Device\\WorkItemDemo");
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\WorkItemDemo");
    PDEVICE_OBJECT dev = NULL;
    PDEVICE_EXTENSION dx;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);
    DriverObject->DriverUnload = DriverUnload;

    status = IoCreateDevice(DriverObject, sizeof(DEVICE_EXTENSION), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE, &dev);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    dev->Flags |= DO_BUFFERED_IO;

    dx = (PDEVICE_EXTENSION)dev->DeviceExtension;
    RtlZeroMemory(dx, sizeof(*dx));
    dx->Self = dev;
    KeInitializeSpinLock(&dx->Lock);
    KeInitializeEvent(&dx->Drained, NotificationEvent, FALSE);
    KeInitializeTimer(&dx->SignalTimer);
    KeInitializeDpc(&dx->SignalDpc, SignalDpcRoutine, dx);

    dx->WorkItem = IoAllocateWorkItem(dev);  // one item, reused by every request
    if (dx->WorkItem == NULL) {
        IoDeleteDevice(dev);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    status = IoCreateSymbolicLink(&linkName, &devName);
    if (!NT_SUCCESS(status)) {
        IoFreeWorkItem(dx->WorkItem);
        IoDeleteDevice(dev);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchDeviceControl;
    dev->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

The drain is provably safe because three facts hold together: the counter is incremented
before the item is queued, `KeFlushQueuedDpcs` guarantees no DPC is still between that
increment and the queue call, and the worker always reaches the decrement because every
operation it performs is bounded. If any of those three were relaxed, the wait would become
an unbounded hang instead of an ordering proof.

## Variations and advanced use

- **Self-freeing items with `IoQueueWorkItemEx`.** When one work item is allocated per
  request, use the `Ex` routine form, whose third parameter is the item itself, so the
  routine can call `IoFreeWorkItem(WorkItem)` as its last action. The item cannot be used
  again; the device object reference is released at that moment.
- **Driver-allocated storage instead of `IoAllocateWorkItem`.** `IoSizeofWorkItem` plus
  `IoInitializeWorkItem(DeviceObject, WorkItem)` embeds the item in your own allocation,
  saving a pool block per request. Pair with `IoUninitializeWorkItem`, and remember the
  device reference still exists while the item is queued.
- **Many items, one per pending IRP.** This is the throughput design: allocate in the
  dispatch routine (or take one from a lookaside list), queue, and free in the worker. The
  drain counter then matters far more than the single reusable item shown above.
- **A dedicated system thread instead.** If the work is not request-bound — polling a
  device, draining a queue forever — a `PsCreateSystemThread` worker with a stop event and a
  `KeWaitForSingleObject` on the thread object is clearer than chaining work items, and it
  gives the code its own stack and priority. That is `wk-kernel-threads` territory.
- **Threaded DPCs.** A threaded DPC also runs at `PASSIVE_LEVEL`, but it still runs on the
  DPC path with no user context and less scheduling flexibility than a work item; prefer
  work items when you need the registry or file I/O, which the registry path may block on.
- **Never block forever in a worker.** Each blocked worker removes a thread from a pool
  shared with the whole system. A worker that waits on an event only your unload path will
  set is a machine-wide deadlock, not a private one.

## Failure modes and diagnosis

| Symptom | Likely cause | First action |
|---|---|---|
| Bugcheck 0x9F `DRIVER_POWER_STATE_FAILURE` | a queued or running work item prevents the device from completing a power transition | `!analyze -v`; `!stacks 2`; find the worker blocked in a wait |
| Bugcheck 0xCE `DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | unload returned with an item still queued | `!analyze -v`; audit the drain; `!exqueue` |
| Bugcheck 0xC2 `BAD_POOL_CALLER` | `IoFreeWorkItem` on a queued item, a double free, or a `WORK_QUEUE_ITEM` in paged pool | `!analyze -v`; `!pool <addr>`; enable pool tracking |
| Bugcheck 0x1E or 0x8E in a worker | the context, the extension, or the driver's code pages were freed or unloaded | `!analyze -v`; `!pool` the context; check unload ordering |
| Bugcheck 0x133 `DPC_WATCHDOG_VIOLATION` | a DPC queued a work item and then spun or waited for it to finish | `!analyze -v`; `!dpcs`; the DPC must return immediately |
| Bugcheck 0xC1 `SPECIAL_POOL_DETECTED_MEMORY_CORRUPTION` | an overrun of the context block passed to the routine | `!analyze -v`; `!pool` both addresses; special pool for the tag |
| Request never completes, client hangs | the item was queued with a context that no path completes, or the queue type starved | `!stacks 2` for worker threads; `!exqueue` |
| Work items stop running for every driver | the pool is exhausted by workers blocked on events that only unload sets | `!stacks 2`; `!exqueue`; `!locks` if they block on a resource |
| IRP completes with `STATUS_CANCELLED` and the client sees a stale buffer | the item ran after cancellation; it must check the IRP state or be drained before cancel is allowed | `!irp <addr>`; decide ownership of the IRP before queueing |
| Crash only under load, never at idle | a race between the drain counter and the queue call | `!analyze -v`; `verifier /standard /driver WorkItemDemo.sys` |

Diagnosis sequence for work that runs after unload. Representative output, abbreviated:

```
kd> !analyze -hang
*** STACK VERIFICATION FAILED: a worker thread is executing code from an unloaded image
kd> !stacks 2
 4.1a2c  ffff8001a4c3b080 0x332  nt!KiSwapContext
                                 nt!ZwOpenKey
                                 WorkItemDemo!ReadTimeoutFromRegistry+0x31
kd> !exqueue
*** Work queue for CPU 0
    ...
```

The stack is the proof: a system worker thread with the unloaded driver's frame on top means
`IoFreeWorkItem` ran before the routine returned, or the routine was never counted at all.
`!exqueue` shows what the pool still has queued; `!stacks 2` shows what is running. Pair them
with `!pool` on the context address to see whether the memory was freed before the fault.

## Safety, IRQL, and cleanup checklist

- [ ] The work item was allocated with the device object and stored in the extension, not on
      a stack and not per call, unless the per-call design is deliberate.
- [ ] The context passed to the routine outlives the queue call and is non-paged.
- [ ] The active-work counter is incremented **before** the queue call, under the lock the
      unload path takes.
- [ ] Unload order: stopping flag, `KeCancelTimer`, `KeFlushQueuedDpcs`, cancel parked IRPs,
      wait for the counter, `IoFreeWorkItem`, `IoDeleteDevice`.
- [ ] The worker never assumes client process context and never touches user addresses.
- [ ] The worker never waits indefinitely on an event that unload must set.
- [ ] `IoFreeWorkItem` and `IoUninitializeWorkItem` are called at `PASSIVE_LEVEL` only, after
      the routine finished.
- [ ] Every IRP handed to a work item has exactly one owner and exactly one completion.
- [ ] `ExQueueWorkItem` is not used in new code; legacy uses get a drain counter before any
      other change.

## Verification

```
msbuild WorkItemDemo.vcxproj /p:Configuration=Debug /p:Platform=x64
reg add "HKLM\SYSTEM\CurrentControlSet\Services\WorkItemDemo\Parameters" ^
        /v TimeoutMs /t REG_DWORD /d 2500
sc create WorkItemDemo type= kernel binPath= C:\test\WorkItemDemo.sys
sc start WorkItemDemo
```

Set `TimeoutMs` first: if the key is missing the worker still runs and completes the IRP with
`STATUS_OBJECT_NAME_NOT_FOUND`, which is a useful negative test but not the happy path. Run
the client below while a second window cycles `sc stop` and `sc start`; a correct drain never
lets the stop hang and never crashes the machine.

```c
// widtest.c  —  cl /W4 widtest.c
#include <windows.h>
#include <stdio.h>

#define GET_TIMEOUT CTL_CODE(FILE_DEVICE_UNKNOWN, 0xA00, METHOD_BUFFERED, FILE_READ_DATA)

int main(void)
{
    ULONG value = 0;
    DWORD bytes = 0, i;
    HANDLE dev = CreateFileW(L"\\\\.\\WorkItemDemo", GENERIC_READ | GENERIC_WRITE, 0,
                             NULL, OPEN_EXISTING, 0, NULL);

    if (dev == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }
    for (i = 0; i < 500; ++i) {
        if (!DeviceIoControl(dev, GET_TIMEOUT, NULL, 0, &value, sizeof(value),
                             &bytes, NULL)) {
            printf("ioctl failed: %lu\n", GetLastError());
            CloseHandle(dev);
            return 1;
        }
        if (bytes != sizeof(value)) {
            printf("short reply: %lu\n", bytes);
            CloseHandle(dev);
            return 1;
        }
    }
    CloseHandle(dev);
    printf("timeout=%lu ms\n", value);
    return 0;
}
```

Expected results: the client prints the value stored in the registry, the stop never times
out, and `!stacks 2` shows no worker thread left inside the driver after the stop. Under
Driver Verifier with Special pool and Pool tracking, a run of several hundred start/stop
cycles with the client attached must produce no bugcheck; any 0xCE or 0xC2 in that loop is a
lifetime bug in the work item, not a flaky test.
