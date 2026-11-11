---
name: wk-synchronization
description: Use when two kernel threads share data at PASSIVE_LEVEL or APC_LEVEL: choose among Interlocked*, KEVENT, KSEMAPHORE, KMUTEX, fast mutex, and ERESOURCE, wait with timeouts via KeWaitForSingleObject, publish named events to user mode, order locks, and diagnose deadlocks with !locks, !stacks, and !analyze -hang.
---

# Passive-level synchronization: pick by IRQL and blocking, not by habit

When two threads touch the same memory and at least one writes, the processor may
interleave, cache, and reorder those accesses; the damage appears elsewhere, usually as a
corrupted list or a bugcheck in unrelated code. The kernel offers three families of defense
and they are not interchangeable: atomic instructions (`Interlocked*`), dispatcher objects a
thread can block on (event, semaphore, mutant, timer), and executive primitives that never
enter the scheduler's wait path (fast mutex, `ERESOURCE`). Two facts decide the choice
before any code is written: the highest IRQL at which the data can be touched, and whether
that code may block. If any path reaches the data at `DISPATCH_LEVEL` or above, this is the
wrong tool and a spin lock is the right one. The rule that prevents most disasters: a
primitive that can block is legal only where the scheduler can still run, and a primitive
that spins is legal only where nothing it touches can take a hard fault.

## When to use

- Two or more threads at `PASSIVE_LEVEL` or `APC_LEVEL` mutate a list, table, or device
  state block.
- A caller must wait for a condition, a count of free buffers, or another thread's
  progress, with a deadline — possibly on whichever of several objects fires first.
- Reads dominate and writes are rare, so a single-writer/multiple-reader lock is worth its
  bookkeeping (`ERESOURCE`).
- A queue or buffer pool must be bounded so producers block instead of growing memory
  without limit (`KSEMAPHORE`).
- You are diagnosing a hang, a lost wakeup, or a lock-order inversion.

Do not use for: data any path touches at `DISPATCH_LEVEL` or above (`wk-spinlocks`),
per-request cancellation queues (`wk-irp-queues-cancel`), or timekeeping and APC mechanics
(`wk-kernel-timers`, `wk-apcs`).

## Prerequisites and related skills

- `<ntddk.h>` declares `KEVENT`, `KSEMAPHORE`, `KMUTEX`, `ERESOURCE`, `FAST_MUTEX`, and
  both wait functions; `KeInitializeMutant` and `KeQueryOwnerMutant` come from `<ntifs.h>`.
- Dispatcher objects and `ERESOURCE` must live in **non-paged pool**; the scheduler touches
  them at `DISPATCH_LEVEL` on your behalf.
- Waits are legal at `PASSIVE_LEVEL` and `APC_LEVEL`. At `DISPATCH_LEVEL` the only legal
  wait is a poll whose `Timeout` points at zero, and a mutant must never be waited for
  there. `ERESOURCE` acquisition is `PASSIVE_LEVEL` only; fast mutex acquisition requires
  `IRQL < DISPATCH_LEVEL`.
- Related: `wk-spinlocks` (same data reachable above `PASSIVE_LEVEL`), `wk-irql`,
  `wk-kernel-threads`, `wk-kernel-timers` (`KTIMER` as a cancelable deadline),
  `wk-kernel-linked-lists` (the structures usually protected).

## Mental model

Give every shared object a **home IRQL**: the highest IRQL at which any path touches it.
That number, not taste, selects the primitive.

```
  does any path touch this data at IRQL >= DISPATCH_LEVEL?
    (DPC, ISR, timer callback, cancel routine, completion routine)
    yes ──► spin lock; nothing inside may block or take a hard fault  (wk-spinlocks)
    no ──► must a thread sleep until state changes?
             no ──► Interlocked* on one word, or a lock held a few dozen instructions
             yes ─► is a reader/writer split needed?
                      yes ─► ERESOURCE (shared/exclusive, recursive, PASSIVE_LEVEL)
                      no  ─► exactly one owner at a time?  yes ─► KMUTEX (recursive, owned)
                                                         no  ─► FAST_MUTEX (non-recursive)
                                                         counting ─► SEMAPHORE / EVENT
```

Four axes decide behavior under stress:

- **Ownership.** A mutant, fast mutex, or `ERESOURCE` records the owning thread, so only
  that thread may release it, recursion is counted, and a dead owner is detectable (the next
  waiter receives `STATUS_ABANDONED`). Events and semaphores have no owner: the thread that
  signals need not be the thread that waited, which is why an ISR can signal an event and one
  thread can hand a semaphore unit to another.
- **Counting versus flag.** A semaphore carries a count with a ceiling; an event is one bit.
- **Blocking versus spinning.** A wait removes the thread from the processor; a spin lock
  burns cycles on another processor.
- **Fairness.** `ERESOURCE` queues waiters, so readers do not starve a writer unless you ask
  for reader preference. A spin lock promises nothing about acquisition order.

## Key APIs and structures

Atomic instructions — the cheapest correct answer when one aligned word holds the state:

| API | Purpose | Returns | Notes |
|---|---|---|---|
| `InterlockedIncrement`, `...Decrement` | add or subtract one | the **new** value | 16- and 64-bit variants |
| `InterlockedAdd`, `...Add64` | add a signed amount | the **new** value | byte and credit counters |
| `InterlockedExchange`, `...64`, `...Pointer` | swap | the **old** value | read-and-clear idioms |
| `InterlockedCompareExchange`, `...64`, `...Pointer` | conditional swap | the **initial** value | success is `initial == Comparand`, not a boolean |
| `ReadNoFence`, `WriteNoFence`, `ReadAcquire`, `WriteRelease` | explicit ordering | value read | Windows 8 / WDK 8 and later; matters on ARM64 |

The `Interlocked*` intrinsics are ordered operations: the compiler will not move accesses
across them, and on x86/x64 the lock prefix drains the store buffer, so they act as full
barriers. `volatile` guarantees none of that — it prevents some compiler reordering and
forces reloads, but it does not make a read-modify-write atomic, does not order the other
processor's view, and on ARM64 prevents no reordering at all. Every wait status below is
success severity, so `NT_SUCCESS` cannot distinguish a timeout from a wakeup.

Waits, and why `NT_SUCCESS` is the wrong test on their return value:

| Call | IRQL | Notes |
|---|---|---|
| `KeWaitForSingleObject(Object, Reason, Mode, Alertable, Timeout)` | ≤ `APC_LEVEL`; `DISPATCH_LEVEL` only with a zero timeout | `Reason` = `Executive`, or `UserRequest` when serving an IRP |
| `KeWaitForMultipleObjects(Count, Object[], Type, ...)` | same | returns `STATUS_WAIT_0 + index` for `WaitAny` |
| Timeout | — | `NULL` infinite; negative relative in 100 ns units; positive absolute since 1601-01-01; `0` poll |

| Status | Meaning | Handling |
|---|---|---|
| `STATUS_SUCCESS` (0) | object became signaled | proceed |
| `STATUS_TIMEOUT` (0x102) | deadline expired | abandon, touch nothing |
| `STATUS_ABANDONED` (0x80) | owner died holding a mutant | log; resource state unknown |
| `STATUS_USER_APC` (0xC0), `STATUS_ALERTED` (0x101) | APC interrupted an alertable wait | retry the wait |

Dispatcher objects and what "signaled" means for each:

| Object | Signaled means | Initialize | Signal and reset |
|---|---|---|---|
| `KEVENT` notification | flag set, all waiters released, stays set | `KeInitializeEvent(&e, NotificationEvent, State)` | `KeSetEvent`; `KeResetEvent` / `KeClearEvent` |
| `KEVENT` synchronization | flag set, **one** waiter released | `KeInitializeEvent(&e, SynchronizationEvent, State)` | `KeSetEvent`; clears itself on release |
| `KSEMAPHORE` | count > 0 | `KeInitializeSemaphore(&s, Count, Limit)` | `KeReleaseSemaphore(&s, Increment, Adjustment, Wait)` |
| `KMUTEX` | unowned | `KeInitializeMutex(&m, 0)` | `KeReleaseMutex(&m, FALSE)` |
| `KTIMER` | expired | `KeInitializeTimer(&t)` | `KeSetTimer(&t, Due, Dpc)`; `KeCancelTimer` |

A mutant (`KMUTEX`) is the only dispatcher object with ownership: `KeInitializeMutex(&m, 0)`
creates it unowned, a successful wait makes the caller the owner, recursion is counted, and
only that owner may call `KeReleaseMutex(&m, FALSE)`. Pass `FALSE` for the `Wait` hint —
`TRUE` leaves IRQL at `DISPATCH_LEVEL` for the following wait, which may be illegal there.

Executive locks that block without a dispatcher object:

| API | IRQL | Notes |
|---|---|---|
| `ExInitializeFastMutex`, `ExAcquireFastMutex` | `< DISPATCH_LEVEL` | exclusive, non-recursive, no timeout |
| `ExAcquireFastMutexUnsafe`, `ExReleaseFastMutexUnsafe` | exactly `APC_LEVEL` | no IRQL bookkeeping; for callers already there |
| `ExInitializeResourceLite` | `PASSIVE_LEVEL` | returns `NTSTATUS`; non-paged storage |
| `ExEnterCriticalRegionAndAcquireResourceExclusive` | `PASSIVE_LEVEL` | long form is `KeEnterCriticalRegion` + acquire |
| `ExAcquireResourceSharedLite(&r, Wait)` | `PASSIVE_LEVEL` | `BOOLEAN`; queues behind a waiting writer |
| `ExAcquireSharedStarveExclusive(&r, Wait)` | `PASSIVE_LEVEL` | reader-preferring; can starve writers |
| `ExReleaseResourceAndLeaveCriticalRegion`, `ExDeleteResourceLite` | `PASSIVE_LEVEL` | pair the first with every acquire; delete only while unowned |

## Decision matrix

| Primitive | Blocks? | Highest IRQL | Recursive | Owned | Nameable, cross-process | Choose when |
|---|---|---|---|---|---|---|
| `Interlocked*` | never | any | n/a | no | no | one aligned word holds the whole state |
| any spin lock | no, it spins | any; raises to `DISPATCH_LEVEL` | **no** | no | no | data is touched at `DISPATCH_LEVEL`+ |
| `FAST_MUTEX` | yes | `< DISPATCH_LEVEL` | **no** | yes | no | short exclusive section, no I/O inside |
| `KMUTEX` | yes | ≤ `APC_LEVEL` | yes | yes | yes | recursion or abandonment detection needed |
| `ERESOURCE` | yes | `PASSIVE_LEVEL` | yes, exclusive | yes | no | readers outnumber writers |
| `KSEMAPHORE` | yes | ≤ `APC_LEVEL` | n/a | **no** | yes | counting free slots, producer/consumer |
| `KEVENT` synchronization | yes | ≤ `APC_LEVEL` | n/a | no | yes | one-shot handoff, wake exactly one |
| `KEVENT` notification | yes | ≤ `APC_LEVEL` | n/a | no | yes | broadcast a state change to many |
| `KTIMER` wait | yes | ≤ `APC_LEVEL` | n/a | no | no | a deadline that must be cancelable |

## Step-by-step

1. **List every path that touches the data** with its IRQL: dispatch routine (usually
   `PASSIVE_LEVEL`), completion routine and DPC (`DISPATCH_LEVEL`), ISR (device IRQL),
   worker thread (`PASSIVE_LEVEL`). Check: nothing above `APC_LEVEL`, or use `wk-spinlocks`.
2. **Classify the access** — read-only (shared lock), read/write (exclusive or `ERESOURCE`),
   counting (semaphore), announcement (event), single word (interlocked) — and confirm the
   acquisition site may wait at all: code in a completion routine, a cancel routine, or a
   DPC cannot wait regardless of the IRQL it reports.
3. **Place every object in non-paged storage** in the extension, zeroed before
   initialization so a half-finished init is obvious in a dump, and initialize in dependency
   order — spin lock, events, semaphores, `ERESOURCE` last because it is the only one that
   can fail. Check every `NTSTATUS`.
4. **Acquire `ERESOURCE` inside a critical region**, always through the combined
   `ExEnter...AndAcquire` / `ExRelease...AndLeave` pair so no exit path can forget it.
5. **Compute timeouts from the request**, in relative 100 ns units, and map
   `STATUS_TIMEOUT` to `STATUS_IO_TIMEOUT`.
6. **Release in reverse order, on a single exit path, before completing the IRP.** A
   completion routine runs at `DISPATCH_LEVEL` and may re-enter your dispatch routine, so
   completing while holding a lock is a self-deadlock waiting for the right client.
7. **Tear down**: set a stop flag under the lock, wake every waiter, join helper threads,
   release and delete the lock objects, then free the extension.

## Complete example: an `ERESOURCE`-protected configuration store

A driver that keeps a fixed-schema configuration table in its device extension. Reads take
the shared lock and run concurrently; writes take the exclusive lock. Both requests use one
fixed layout, so a single structure is validated, and one access function takes the write
flag so the two lock modes cannot drift apart.

```c
// ConfigStore.c  —  WDM driver, x64
#include <ntddk.h>

#define CONFIG_MAX    8
#define CONFIG_NAMECH 24                // characters, including terminator

typedef struct _CONFIG_QUERY {          // shared with the client, byte for byte
    WCHAR Name[CONFIG_NAMECH];
    ULONG Value;
} CONFIG_QUERY, *PCONFIG_QUERY;

typedef struct _DEVICE_EXTENSION {
    ERESOURCE Lock;
    BOOLEAN   LockReady;
    ULONG     Values[CONFIG_MAX];       // slot order matches kConfigNames
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

static const PCWSTR kConfigNames[CONFIG_MAX] = {
    L"MaxRetries", L"TimeoutMs", L"DebugLevel", L"BufferSize",
    L"PoolTag", L"Verbose", L"SamplingHz", L"Reserved"
};

#define IOCTL_CONFIG_GET CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_READ_DATA)
#define IOCTL_CONFIG_SET CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_WRITE_DATA)

#pragma alloc_text(PAGE, ConfigAccess)

static NTSTATUS ConfigAccess(PDEVICE_EXTENSION dx, PCONFIG_QUERY Query, BOOLEAN Write)
{
    UNICODE_STRING key, name;
    NTSTATUS status = STATUS_OBJECT_NAME_NOT_FOUND;
    ULONG i;
    PAGED_CODE();
    if (Write) {
        ExEnterCriticalRegionAndAcquireResourceExclusive(&dx->Lock);
    } else {
        ExEnterCriticalRegionAndAcquireResourceShared(&dx->Lock);
    }
    RtlInitUnicodeString(&key, Query->Name);
    for (i = 0; i < CONFIG_MAX; ++i) {
        RtlInitUnicodeString(&name, kConfigNames[i]);
        if (!RtlEqualUnicodeString(&key, &name, TRUE)) {
            continue;
        }
        if (Write) {
            dx->Values[i] = Query->Value;
        } else {
            Query->Value = dx->Values[i];
        }
        status = STATUS_SUCCESS;
        break;
    }
    ExReleaseResourceAndLeaveCriticalRegion(&dx->Lock);
    return status;
}
```

The dispatch routines validate the code and both lengths before reading the system buffer,
and complete the IRP only after the lock is gone.

```c
static NTSTATUS DispatchCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    UNREFERENCED_PARAMETER(DeviceObject);
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = (sp->MajorFunction == IRP_MJ_CREATE) ? FILE_OPENED : 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS DispatchDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    ULONG code   = sp->Parameters.DeviceIoControl.IoControlCode;
    ULONG inLen  = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    PCONFIG_QUERY query = (PCONFIG_QUERY)Irp->AssociatedIrp.SystemBuffer;
    NTSTATUS status;
    ULONG bytes = sizeof(CONFIG_QUERY);
    if (code != IOCTL_CONFIG_GET && code != IOCTL_CONFIG_SET) {
        status = STATUS_INVALID_DEVICE_REQUEST;
    } else if (inLen < sizeof(CONFIG_QUERY) || outLen < sizeof(CONFIG_QUERY)) {
        status = STATUS_BUFFER_TOO_SMALL;
    } else {
        query->Name[CONFIG_NAMECH - 1] = L'\0';
        if (query->Name[0] == L'\0') {
            status = STATUS_INVALID_PARAMETER;
        } else {
            status = ConfigAccess(dx, query, (BOOLEAN)(code == IOCTL_CONFIG_SET));
            if (code == IOCTL_CONFIG_SET) {
                bytes = 0;
            }
        }
    }
    if (!NT_SUCCESS(status)) {
        bytes = 0;
    }
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = bytes;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

Initialization and teardown: the resource is deleted before the extension memory
disappears, and only if it was initialized.

```c
static VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\ConfigStore");
    PDEVICE_OBJECT dev = DriverObject->DeviceObject;
    IoDeleteSymbolicLink(&linkName);
    while (dev != NULL) {
        PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)dev->DeviceExtension;
        PDEVICE_OBJECT next = dev->NextDevice;
        if (dx->LockReady) {
            ExDeleteResourceLite(&dx->Lock);   // no owner, no waiter at this point
        }
        IoDeleteDevice(dev);
        dev = next;
    }
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING devName  = RTL_CONSTANT_STRING(L"\\Device\\ConfigStore");
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\ConfigStore");
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
    dx = (PDEVICE_EXTENSION)dev->DeviceExtension;
    RtlZeroMemory(dx, sizeof(*dx));
    status = ExInitializeResourceLite(&dx->Lock);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(dev);
        return status;
    }
    dx->LockReady = TRUE;
    status = IoCreateSymbolicLink(&linkName, &devName);
    if (!NT_SUCCESS(status)) {
        ExDeleteResourceLite(&dx->Lock);
        IoDeleteDevice(dev);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE]         = DispatchCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]          = DispatchCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchDeviceControl;
    dev->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

## Complete example: producer/consumer with a semaphore and an event

A bounded queue that a driver-owned thread drains. The semaphore counts free slots, so a
producer blocks when the queue is full instead of growing memory. The event announces that
work exists; the consumer drains the list in a loop because an auto-reset event coalesces —
two producers enqueueing two items and each setting the event release the consumer **once**,
and the second item would sit unnoticed if the consumer handled one item per wakeup. Paste
these functions into the driver skeleton above, call `PqInitialize` where that code calls
`ExInitializeResourceLite`, call `PqTeardown` from `DriverUnload` before `IoDeleteDevice`,
and add `PqWrite` to `MajorFunction[IRP_MJ_WRITE]`.

```c
// PipeQueue.c  —  the whole producer/consumer mechanism
#include <ntddk.h>

#define PQ_TAG      'qQpP'
#define PQ_DEPTH    8
#define PQ_MAX_DATA 64

typedef struct _PQ_ITEM {
    LIST_ENTRY Link;
    ULONG      Length;
    UCHAR      Data[PQ_MAX_DATA];
} PQ_ITEM, *PPQ_ITEM;

typedef struct _PQ_DEVEXT {
    KSPIN_LOCK QueueLock;           // protects Queue
    LIST_ENTRY Queue;
    KSEMAPHORE FreeSlots;           // count == free capacity
    KEVENT     DataReady;           // synchronization, auto-reset
    KEVENT     StopEvent;           // notification
    PETHREAD   WorkerThread;
    HANDLE     WorkerHandle;
} PQ_DEVEXT, *PPQ_DEVEXT;

VOID PqTeardown(PPQ_DEVEXT dx);

static VOID PqWorkerThread(PVOID Context)
{
    PPQ_DEVEXT dx = (PPQ_DEVEXT)Context;
    PVOID waits[2];
    waits[0] = &dx->StopEvent;
    waits[1] = &dx->DataReady;
    for (;;) {
        NTSTATUS status = KeWaitForMultipleObjects(2, waits, WaitAny, Executive,
                                                   KernelMode, FALSE, NULL, NULL);
        if (status == STATUS_WAIT_0) {
            break;                              // stop requested
        }
        if (status != STATUS_WAIT_1) {
            continue;                           // APC or spurious wakeup
        }
        for (;;) {
            KLOCK_QUEUE_HANDLE handle;
            PPQ_ITEM item = NULL;
            KeAcquireInStackQueuedSpinLock(&dx->QueueLock, &handle);
            if (!IsListEmpty(&dx->Queue)) {
                item = CONTAINING_RECORD(RemoveHeadList(&dx->Queue), PQ_ITEM, Link);
            }
            KeReleaseInStackQueuedSpinLock(&handle);
            if (item == NULL) {
                break;                          // drained
            }
            DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                       "PipeQueue: consumed %lu bytes\n", item->Length);
            ExFreePoolWithTag(item, PQ_TAG);
            KeReleaseSemaphore(&dx->FreeSlots, 1, 1, FALSE);
        }
    }
    PsTerminateSystemThread(STATUS_SUCCESS);
}

static NTSTATUS PqWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PPQ_DEVEXT dx = (PPQ_DEVEXT)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    ULONG length = sp->Parameters.Write.Length;
    PPQ_ITEM item;
    KLOCK_QUEUE_HANDLE handle;
    LARGE_INTEGER timeout;
    NTSTATUS status;
    if (length == 0 || length > PQ_MAX_DATA) {
        status = STATUS_INVALID_PARAMETER;
        goto Complete;
    }
    timeout.QuadPart = -30 * 1000 * 1000LL;     // 30 seconds, relative
    status = KeWaitForSingleObject(&dx->FreeSlots, Executive, KernelMode, FALSE, &timeout);
    if (status != STATUS_SUCCESS) {
        status = (status == STATUS_TIMEOUT) ? STATUS_IO_TIMEOUT : status;
        goto Complete;
    }
    item = (PPQ_ITEM)ExAllocatePool2(POOL_FLAG_NON_PAGED, sizeof(PQ_ITEM), PQ_TAG);
    if (item == NULL) {
        KeReleaseSemaphore(&dx->FreeSlots, 1, 1, FALSE);   // give the slot back
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Complete;
    }
    item->Length = length;
    RtlCopyMemory(item->Data, Irp->AssociatedIrp.SystemBuffer, length);
    KeAcquireInStackQueuedSpinLock(&dx->QueueLock, &handle);
    InsertTailList(&dx->Queue, &item->Link);
    KeReleaseInStackQueuedSpinLock(&handle);
    KeSetEvent(&dx->DataReady, IO_NO_INCREMENT, FALSE);
Complete:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = NT_SUCCESS(status) ? length : 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

NTSTATUS PqInitialize(PPQ_DEVEXT dx)
{
    NTSTATUS status;
    RtlZeroMemory(dx, sizeof(*dx));
    InitializeListHead(&dx->Queue);
    KeInitializeSpinLock(&dx->QueueLock);
    KeInitializeSemaphore(&dx->FreeSlots, PQ_DEPTH, PQ_DEPTH);
    KeInitializeEvent(&dx->DataReady, SynchronizationEvent, FALSE);
    KeInitializeEvent(&dx->StopEvent, NotificationEvent, FALSE);
    status = PsCreateSystemThread(&dx->WorkerHandle, THREAD_ALL_ACCESS, NULL, NULL, NULL,
                                  PqWorkerThread, dx);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    status = ObReferenceObjectByHandle(dx->WorkerHandle, THREAD_ALL_ACCESS, *PsThreadType,
                                       KernelMode, (PVOID*)&dx->WorkerThread, NULL);
    if (!NT_SUCCESS(status)) {
        ZwClose(dx->WorkerHandle);
        dx->WorkerHandle = NULL;
    }
    return status;
}

VOID PqTeardown(PPQ_DEVEXT dx)
{
    if (dx->WorkerThread != NULL) {
        KeSetEvent(&dx->StopEvent, IO_NO_INCREMENT, FALSE);
        KeWaitForSingleObject(dx->WorkerThread, Executive, KernelMode, FALSE, NULL);
        ObDereferenceObject(dx->WorkerThread);
        dx->WorkerThread = NULL;
    }
    if (dx->WorkerHandle != NULL) {
        ZwClose(dx->WorkerHandle);
        dx->WorkerHandle = NULL;
    }
    for (;;) {
        KLOCK_QUEUE_HANDLE handle;
        PPQ_ITEM item = NULL;
        KeAcquireInStackQueuedSpinLock(&dx->QueueLock, &handle);
        if (!IsListEmpty(&dx->Queue)) {
            item = CONTAINING_RECORD(RemoveHeadList(&dx->Queue), PQ_ITEM, Link);
        }
        KeReleaseInStackQueuedSpinLock(&handle);
        if (item == NULL) {
            break;
        }
        ExFreePoolWithTag(item, PQ_TAG);
    }
}
```

Teardown stops the consumer first, then frees what is still queued; the stop event stays set,
so a worker that loops back sees it, and `PqTeardown` is safe on a zeroed extension if
`PqInitialize` failed early.

## Variations and advanced use

- **Named events across the user boundary.** A name under `\BaseNamedObjects` is
  per-session, so a service in session 0 and a desktop process in session 1 meet only in the
  machine-wide directory: the `Global\` prefix selects it on the client side and the literal
  path selects it on the driver side.
```c
// driver: publish state to user mode (the helper creates the event already signaled)
UNICODE_STRING evName = RTL_CONSTANT_STRING(L"\\BaseNamedObjects\\MyDriverState");
HANDLE rawHandle;
PKEVENT stateEvent = IoCreateNotificationEvent(&evName, &rawHandle);
if (stateEvent == NULL) { return STATUS_INSUFFICIENT_RESOURCES; }
// unload: ZwClose(rawHandle)

// client: wait for the driver to publish state, then read it through an IOCTL
HANDLE ev = OpenEventW(SYNCHRONIZE, FALSE, L"Global\\MyDriverState");
if (ev == NULL) { printf("open failed: %lu\n", GetLastError()); return 1; }
WaitForSingleObject(ev, 5000);
CloseHandle(ev);
```

  Reset the event once the consumer has observed the state, and keep the reset type
  consistent: `CreateEventW` with manual reset maps to a notification event. These helpers
  take no security descriptor, so if a low-privilege process must not open the event, create
  it with `ZwCreateEvent` and explicit `OBJECT_ATTRIBUTES`.
- **`WaitAll` and arrays above three objects.** A thread has `THREAD_WAIT_OBJECTS` (3)
  built-in wait blocks. For more, allocate `Count * sizeof(KWAIT_BLOCK)` from non-paged pool,
  pass it as the last argument, and free it after the wait. `WaitAll` returns
  `STATUS_SUCCESS` only when every object is signaled at once.
- **Reader versus writer preference.** Default shared acquisition queues behind a waiting
  writer, so writers cannot starve; `ExAcquireSharedStarveExclusive` lets readers jump the
  queue and can starve a writer forever, acceptable only for short bounded reads.
- **A cancelable sleep.** Give the waiter a `KTIMER` instead of a timeout when another
  thread must reset the deadline: `KeSetTimer(&t, due, NULL)`, then `WaitAny` over
  `{ stopEvent, timer }`, with `KeCancelTimer` to withdraw it.

## Failure modes and diagnosis

| Symptom | Likely cause | First action |
|---|---|---|
| Hang, no bugcheck, two threads each in `KeWaitForSingleObject` | lock-order inversion: A holds L1 and wants L2, B holds L2 and wants L1 | `!analyze -hang`; `!stacks 2`; `!locks` |
| `!locks` shows a resource owned by a thread that is itself blocked | a wait was issued while holding the resource, and the awaited work needs it | never wait while holding a lock |
| `STATUS_TIMEOUT` treated as success and the object used anyway | `NT_SUCCESS` is true for `STATUS_TIMEOUT` (0x102) | compare `status == STATUS_TIMEOUT` |
| Bugcheck 0x1E right after an acquire | the lock memory was freed or never initialized (init failed, unchecked) | `!analyze -v`; `!pool <addr>`; audit the init path |
| Bugcheck 0x8E in a helper thread | an exception unwound past release, leaving the lock owned forever | `__try/__finally` or a scope-bound release object |
| Bugcheck 0x9F on sleep or shutdown | a thread sits in an infinite wait the power path needs to finish | `!analyze -v`; `!stacks 2`; find the `Timeout == NULL` wait |
| Bugcheck 0x133 with no visibly long DPC | a `DISPATCH_LEVEL` path busy-waits for a flag owned by a thread that cannot run | `!analyze -v`; `!dpcs`; locate the spinning frame |
| Bugcheck 0xC1 on a block that holds a `KEVENT` | an overrun past a neighbor allocation, caught at free by special pool | `!analyze -v`; `!pool` both addresses |
| Bugcheck 0xC2 `BAD_POOL_CALLER` | double free of a structure still holding a live dispatcher object | `!analyze -v`; `!pool` with the tag; pool tracking |
| `0xC0000046` `STATUS_MUTANT_NOT_OWNED` | releasing a mutant from a thread that does not own it | acquire and release on the same thread |
| Wait never satisfied although the signaler ran | auto-reset event set while already set, so only one wakeup happened | drain in a loop; broadcast with a notification event |

A hang investigation, in order. Representative output, abbreviated:

```
kd> !analyze -hang
*** STACK VERIFICATION FAILED: 2 threads are waiting on each other
kd> !stacks 2
 0.1ff4  ffff8c0a11a2c080 0x1a3  nt!KiSwapContext
                                nt!KeWaitForSingleObject
                                ConfigStore!ConfigAccess+0x5e
kd> !locks
Resource @ ffff8c0a12345680    Exclusively owned
    Contention Count = 3
    NumberOfSharedWaiters = 1
```

Read it as: the thread that owns the resource is blocked in a wait, while another waits for
shared access. The owner is the bug, and a resource outliving its device means
`ExDeleteResourceLite` was skipped.

## Safety, IRQL, and cleanup checklist

- [ ] Every object is in non-paged memory, zeroed before init, initialized exactly once.
- [ ] No wait is issued while holding any lock; no lock is acquired inside a wait loop.
- [ ] `ERESOURCE` is always acquired and released through the combined critical-region
      helpers, so normal APCs cannot run while ownership is held.
- [ ] Every acquisition has exactly one release on every exit path, including errors, and
      every wait status is compared with `==`, never `NT_SUCCESS`.
- [ ] Every user-reachable path uses a finite timeout and maps `STATUS_TIMEOUT` to
      `STATUS_IO_TIMEOUT`.
- [ ] Named events are closed with `ZwClose`, or referenced with `ObReferenceObject` and
      released with `ObDereferenceObject` — one or the other, not both.
- [ ] The lock hierarchy is documented in a comment at the top of the file and obeyed
      everywhere; ranks are never acquired in descending order; teardown runs stop flag,
      wake waiters, join helper threads, release and delete locks, free the extension.

## Verification

```
msbuild ConfigStore.vcxproj /p:Configuration=Debug /p:Platform=x64
sc create ConfigStore type= kernel binPath= C:\test\ConfigStore.sys
sc start ConfigStore
```

With the driver loaded and nothing attached, `!locks` must report no resource owned by it.
Then run two writer and two reader copies of the client below at once; a correct
implementation never bugchecks under verifier with Special pool enabled.

```c
// configtest.c  —  cl /W4 configtest.c     ; start four copies at once: 2 writers, 2 readers
#include <windows.h>
#include <stdio.h>

typedef struct { WCHAR Name[24]; ULONG Value; } QUERY;
#define GET CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_READ_DATA)
#define SET CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_WRITE_DATA)

int main(int argc, char** argv)
{
    QUERY q = { L"MaxRetries", 0 };
    DWORD bytes = 0, i;
    BOOL write = (argc > 1);
    HANDLE dev = CreateFileW(L"\\\\.\\ConfigStore", GENERIC_READ | GENERIC_WRITE, 0,
                             NULL, OPEN_EXISTING, 0, NULL);
    if (dev == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }
    for (i = 0; i < 20000; ++i) {
        q.Value = i;
        if (!DeviceIoControl(dev, write ? SET : GET, &q, sizeof(q), &q, sizeof(q),
                             &bytes, NULL)) {
            printf("ioctl failed: %lu\n", GetLastError());
            CloseHandle(dev);
            return 1;
        }
    }
    CloseHandle(dev);
    printf("%s finished, last value %lu\n", write ? "writer" : "reader", q.Value);
    return 0;
}
```

Debugger checks that prove the design rather than one run: `!locks` empty when idle and
`!analyze -hang` against a forced hang reporting no cycle.
