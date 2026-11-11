---
name: wk-spinlocks
description: Use when data is touched at DISPATCH_LEVEL or above: KSPIN_LOCK acquire and release with IRQL save and restore, in-stack queued spinlocks and KLOCK_QUEUE_HANDLE, interrupt spinlocks and KeSynchronizeExecution under DIRQL, hold-time and cache-line rules, and debugging 0xC2, 0xA, 0xD1 and lock-order hangs with !analyze and !stacks.
---

# Spinlocks: exclusion above PASSIVE_LEVEL

A spin lock is a pointer-sized word that a processor exchanges atomically; the loser keeps
reading it until the winner clears it. Because no thread can be put to sleep at
`DISPATCH_LEVEL` or above, waiting means spinning in place, and that is only tolerable when
the protected work is a few dozen instructions. The lock alone is not the whole mechanism:
acquiring one also raises the current processor's IRQL to `DISPATCH_LEVEL`, which stops a
DPC on the same processor from running and deadlocking against itself. Two consequences
follow, and they account for nearly every spinlock bug: the release must restore the IRQL
that was saved at acquisition, and nothing inside the critical section may take a hard fault
or block. A third rule covers hardware: if an ISR touches the same data, a plain spinlock
cannot exclude it, because an interrupt at `DIRQL` preempts `DISPATCH_LEVEL` — that case
needs `KeSynchronizeExecution` or the interrupt spinlock.

## When to use

- Two paths touch a device structure and at least one of them runs at `DISPATCH_LEVEL`:
  a DPC, a timer callback, a `StartIo` path, an `IoCompleteRequest` completion routine.
- A passive dispatch routine shares a queue or counters with a DPC.
- An ISR and a passive routine must read-modify-write the same hardware register.
- You need to update several fields as one atomic unit, which `Interlocked*` cannot do.
- A hang bugcheck or a `!stacks` dump shows a thread spinning at elevated IRQL.
- You are deciding between a spinlock and a lock that can block (`ERESOURCE`, mutex).

Do not use for: code that runs only at `PASSIVE_LEVEL` and may block (`wk-synchronization`),
data shared with an ISR through a driver-owned lock (see the DIRQL rule below), or a single
counter (`InterlockedIncrement` is cheaper and cannot deadlock).

## Prerequisites and related skills

- `<ntddk.h>` provides `KSPIN_LOCK`, `KeInitializeSpinLock`, `KeAcquireSpinLock`,
  `KeReleaseSpinLock`, the queued variants, `KLOCK_QUEUE_HANDLE`, `KeSynchronizeExecution`,
  and the `KDPC` and `PKINTERRUPT` types. `IoConnectInterruptEx` is in `<wdm.h>`.
- The lock word and everything it protects must be in **non-paged memory**: the lock is
  touched at `DISPATCH_LEVEL` and above, and a paging fault there is a bugcheck.
- The acquiring code must be at or below `DISPATCH_LEVEL`, and it must be able to finish
  without the scheduler.
- Related: `wk-irql` (what each IRQL permits), `wk-dpcs` (DPC and timer mechanics),
  `wk-synchronization` (the same data reachable only at `PASSIVE_LEVEL`),
  `wk-kernel-linked-lists` (the structures usually protected),
  `wk-irp-queues-cancel` (queues protected by these locks).

## Mental model

A spinlock does two jobs at once, and you need both:

1. **Cross-processor exclusion.** The lock word is the only thing all processors agree on;
   the atomic exchange guarantees one winner, and everyone else spins.
2. **Local preemption masking.** Raising IRQL to `DISPATCH_LEVEL` prevents a DPC on the
   *same* processor from running. Without that raise, a DPC could preempt the owner, try to
   acquire the same lock, and spin forever, because the owner can never be scheduled again
   while the DPC occupies the processor. This is why the API pairs every acquire with an
   IRQL save and every release with a restore, and why the saved value must be threaded
   through unchanged.

```
CPU 0                                    CPU 1
  KeAcquireSpinLock(&L, &old)              KeAcquireSpinLock(&L, &old)
    IRQL 0 -> 2  (DPCs masked here)          IRQL 2 -> 2 (no change, already there)
    exchange L: got it                       exchange L: failed
    ... work ...                             ... spins, reading L ...
  KeReleaseSpinLock(&L, old)                 exchange L: got it
    IRQL 2 -> 0                              ... work ...
```

`KSPIN_LOCK` is `ULONG_PTR`, so the lock is one aligned word. The value saved by
`KeAcquireSpinLock` is the IRQL the caller had *before* the raise; passing anything else to
`KeReleaseSpinLock` — `PASSIVE_LEVEL`, `DISPATCH_LEVEL` because it "looks right", or a
variable reused by a nested acquisition — leaves the processor at the wrong level and the
damage is a later `IRQL_NOT_LESS_OR_EQUAL` or a silent loss of DPC delivery.

A queued spinlock is the same word with a different protocol: each waiter publishes a
per-processor queue node and spins on *its own* node, so the cache line holding the lock is
touched only by the head of the queue. That removes the thundering herd on the lock word,
gives FIFO ordering (no barging), and scales on many-core machines. The queue node is the
`KLOCK_QUEUE_HANDLE` you declare on the stack; it carries both the queue linkage and the
saved IRQL, so it must live in the acquiring function's frame for the whole critical section.

## Key APIs and structures

| API | Purpose | IRQL on entry | Notes |
|---|---|---|---|
| `KeInitializeSpinLock(&lock)` | put the lock in the unowned state | any | once, before first use, non-paged storage |
| `KeAcquireSpinLock(&lock, &oldIrql)` | acquire, raise to `DISPATCH_LEVEL` | ≤ `DISPATCH_LEVEL` | stores the previous IRQL in `oldIrql` |
| `KeReleaseSpinLock(&lock, oldIrql)` | release, restore that IRQL | `DISPATCH_LEVEL` | pass the **saved** value, never a literal |
| `KeAcquireSpinLockRaiseToDpc(&lock)` | acquire, raise, return the IRQL to release with | ≤ `DISPATCH_LEVEL` | return value feeds `KeReleaseSpinLock` |
| `KeAcquireSpinLockAtDpcLevel(&lock)` | acquire without an IRQL change | exactly `DISPATCH_LEVEL` or above | for DPCs and ISRs; fastest form |
| `KeReleaseSpinLockFromDpcLevel(&lock)` | release without an IRQL change | same | pair with the AtDpcLevel acquire |
| `KeTestSpinLock(&lock)` | try once, no spinning | `DISPATCH_LEVEL` | debug or optional work; not a production path |
| `KeAcquireInStackQueuedSpinLock(&lock, &handle)` | queued acquire, raise to `DISPATCH_LEVEL` | ≤ `DISPATCH_LEVEL` | FIFO, scales; handle is stack-allocated |
| `KeReleaseInStackQueuedSpinLock(&handle)` | queued release | `DISPATCH_LEVEL` | same handle, same frame |
| `KeAcquireInStackQueuedSpinLockAtDpcLevel` | queued acquire, no IRQL change | `DISPATCH_LEVEL`+ | use inside a DPC |
| `KeReleaseInStackQueuedSpinLockFromDpcLevel` | queued release, no IRQL change | same | pair with the above |
| `KeAcquireInterruptSpinLock(&intr, &oldIrql)` | raise to the interrupt's `DIRQL` and mask its line | ≤ `DIRQL` | serializes against the ISR |
| `KeReleaseInterruptSpinLock(&intr, oldIrql)` | release and restore | `DIRQL` | pass the saved IRQL |
| `KeSynchronizeExecution(&intr, routine, context)` | run a callback at `DIRQL` with the interrupt locked | ≤ `DIRQL` | returns the callback's `BOOLEAN` |

Contexts, and what each may do:

| Context | IRQL | May block? | May page? | Correct way to take a driver lock |
|---|---|---|---|---|
| dispatch routine, `PASSIVE_LEVEL` | 0 | yes, but not while holding | yes | `KeAcquireSpinLock` or the queued form |
| DPC, timer callback, `StartIo` | 2 | no | no | `...AtDpcLevel` / `...FromDpcLevel` |
| ISR | `DIRQL`, 3-26 | no | no | only `...AtDpcLevel`, and see the DIRQL rule |
| completion routine | 2 | no | no | `...AtDpcLevel` if already at 2 |

**The DIRQL rule.** If the ISR takes a driver-owned spinlock, every other acquirer of that
lock must hold it at the interrupt's `DIRQL`, not at `DISPATCH_LEVEL`. Otherwise this
sequence deadlocks the processor: passive code acquires the lock (IRQL rises to 2), the
device interrupts (IRQL rises to `DIRQL`, preempting the owner), the ISR tries the same lock
and spins, and the owner can never run again. The lock you pass to `IoConnectInterruptEx` is
one of these locks — the kernel holds it while the ISR runs, so nothing may take it at
`DISPATCH_LEVEL` either. Choose one of two designs and never mix them: keep the ISR away from
the driver lock entirely (it reads hardware and queues a DPC, which takes the lock at
`DISPATCH_LEVEL`), or protect the shared register with `KeSynchronizeExecution` on every
path, including the DPC.

## Step-by-step

1. **Write down the IRQL of every path** that will touch the data. Check: at least one path
   is at `DISPATCH_LEVEL` or above, otherwise a blocking lock is simpler and safer.
2. **Allocate the lock inside the non-paged device extension**, align it, and call
   `KeInitializeSpinLock` exactly once before the device is exposed. Check:
   `InitializeListHead`, counters, and pointers that the lock protects are initialized in the
   same routine, before the lock is ever acquired.
3. **Choose the variant per context**: `KeAcquireSpinLock` from dispatch code,
   `...AtDpcLevel` inside a DPC, queued variants when contention across many cores is
   expected, interrupt spinlock or `KeSynchronizeExecution` when an ISR shares the data.
4. **Keep the critical section to register reads and memory writes.** Move allocation,
   logging, `IoCompleteRequest`, and any call that might block outside the lock.
5. **Release on every path**, including error paths, using the saved IRQL. Check: no early
   `return` between acquire and release.
6. **Prove the lock cannot be taken twice** on the same path. Check: no helper called with
   the lock held re-acquires it; a nested acquisition is an immediate self-deadlock.
7. **For ISR-shared registers**, wrap every access in `KeSynchronizeExecution` instead of
   taking a lock by hand. Check: the callback does only register work and returns `TRUE`.
8. **Tear down in order**: disconnect the interrupt, flush queued DPCs
   (`KeFlushQueuedDpcs`), then delete the device. Check: no DPC can run after the extension
   is freed.

## Complete example: a queued spinlock in a device extension, ISR to DPC handoff

The extension holds a queued spinlock that protects counters shared by a DPC and a passive
IOCTL. The ISR deliberately does **not** take that lock: it acknowledges the device and
queues the DPC, which is enough because `KeInsertQueueDpc` orders the ISR's writes before the
DPC runs. The interrupt object has its own lock, passed to `IoConnectInterruptEx`; that lock
belongs to the kernel's ISR wrapper and is never acquired at `DISPATCH_LEVEL` — the driver
reaches the registers it guards only through `KeSynchronizeExecution`. The control register,
which the ISR also reads, is written the same way, so the read-modify-write is atomic with
respect to the ISR. In a PnP driver the vector and IRQL come from the translated resource
list at `IRP_MN_START_DEVICE`; here they are fixed for a test device whose assignment is
known.

```c
// SpinDev.c  —  queued spinlock, DIRQL-safe register access, ISR to DPC handoff
#include <ntddk.h>

#define SPIN_TAG       'nips'
#define REG_STATUS     0
#define REG_CONTROL    1
#define REG_COUNT      4
#define TEST_VECTOR    0x2B     // must match the vector the device was assigned
#define TEST_IRQL      0x0B     // must match the IRQL the device was assigned
#define STATUS_ACTIVE  0x00000001
#define CONTROL_ARM    0x00000002

typedef struct _REG_OP {
    struct _DEVICE_EXTENSION* dx;
    ULONG Index;
    ULONG Value;
} REG_OP, *PREG_OP;

typedef struct _DEVICE_EXTENSION {
    KSPIN_LOCK      StateLock;      // queued spinlock, protects the two fields below
    KSPIN_LOCK      InterruptLock;  // owned by the interrupt object, never taken directly
    ULONG           Interrupts;     // written by the DPC, read by dispatch
    ULONG           LastStatus;     // written by the DPC, read by dispatch
    PULONG          Registers;      // stands in for a mapped BAR
    PKINTERRUPT     InterruptObject;
    KDPC            IsrDpc;
    PDEVICE_OBJECT  Self;
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

static BOOLEAN DeviceIsr(PKINTERRUPT Interrupt, PVOID Context)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)Context;
    ULONG status;

    UNREFERENCED_PARAMETER(Interrupt);
    status = READ_REGISTER_ULONG(&dx->Registers[REG_STATUS]);
    if ((status & STATUS_ACTIVE) == 0) {
        return FALSE;                                   // not our interrupt: never ack
    }
    WRITE_REGISTER_ULONG(&dx->Registers[REG_STATUS], status);   // ack clears the line
    KeInsertQueueDpc(&dx->IsrDpc, ULongToPtr(status), NULL);
    return TRUE;
}

static VOID IsrDpcRoutine(PKDPC Dpc, PVOID DeferredContext,
                          PVOID SystemArgument1, PVOID SystemArgument2)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)DeferredContext;
    KLOCK_QUEUE_HANDLE handle;

    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(SystemArgument2);

    KeAcquireInStackQueuedSpinLockAtDpcLevel(&dx->StateLock, &handle);
    dx->Interrupts += 1;
    dx->LastStatus = PtrToUlong(SystemArgument1);
    KeReleaseInStackQueuedSpinLockFromDpcLevel(&handle);
}

// Runs at DIRQL with the interrupt spinlock held: register work only.
static BOOLEAN WriteRegAtDirql(PVOID Context)
{
    PREG_OP op = (PREG_OP)Context;

    WRITE_REGISTER_ULONG(&op->dx->Registers[op->Index], op->Value);
    return TRUE;
}
```

The dispatch routine reads the counters under the same queued lock at `PASSIVE_LEVEL`, and
writes the control register through `KeSynchronizeExecution` so that the ISR can never
observe a half-written control word.

```c
typedef struct _SPIN_STATS {
    ULONG Interrupts;
    ULONG LastStatus;
} SPIN_STATS, *PSPIN_STATS;

#define IOCTL_SPIN_GET_STATS \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x900, METHOD_BUFFERED, FILE_READ_DATA)
#define IOCTL_SPIN_ARM \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x901, METHOD_BUFFERED, FILE_WRITE_DATA)

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
    ULONG inLen  = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    PVOID buffer = Irp->AssociatedIrp.SystemBuffer;
    NTSTATUS status = STATUS_SUCCESS;
    ULONG bytes = 0;

    switch (sp->Parameters.DeviceIoControl.IoControlCode) {
    case IOCTL_SPIN_GET_STATS:
        if (outLen < sizeof(SPIN_STATS)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        {
            KLOCK_QUEUE_HANDLE handle;
            PSPIN_STATS stats = (PSPIN_STATS)buffer;

            KeAcquireInStackQueuedSpinLock(&dx->StateLock, &handle);
            stats->Interrupts = dx->Interrupts;
            stats->LastStatus = dx->LastStatus;
            KeReleaseInStackQueuedSpinLock(&handle);
            bytes = sizeof(SPIN_STATS);
        }
        break;

    case IOCTL_SPIN_ARM:
        if (inLen < sizeof(ULONG) || dx->InterruptObject == NULL) {
            status = STATUS_INVALID_DEVICE_REQUEST;
            break;
        }
        {
            REG_OP op;

            op.dx = dx;
            op.Index = REG_CONTROL;
            op.Value = *(PULONG)buffer | CONTROL_ARM;
            if (!KeSynchronizeExecution(dx->InterruptObject, WriteRegAtDirql, &op)) {
                status = STATUS_UNSUCCESSFUL;
            }
        }
        break;

    default:
        status = STATUS_INVALID_DEVICE_REQUEST;
        break;
    }

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = bytes;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

The interrupt is connected from a legacy driver by reading a vector from the service key;
connecting is optional, so the driver still loads and serves the stats IOCTL when no vector
is configured.

```c
static NTSTATUS ConnectDeviceInterrupt(PDEVICE_EXTENSION dx, ULONG Vector, KIRQL Irql)
{
    IO_CONNECT_INTERRUPT_PARAMETERS params;

    RtlZeroMemory(&params, sizeof(params));
    params.Version = CONNECT_FULLY_SPECIFIED;
    params.FullySpecified.PhysicalDeviceObject = dx->Self;
    params.FullySpecified.InterruptObject = &dx->InterruptObject;
    params.FullySpecified.ServiceRoutine = DeviceIsr;
    params.FullySpecified.ServiceContext = dx;
    params.FullySpecified.SpinLock = &dx->InterruptLock;
    params.FullySpecified.SynchronizeIrql = Irql;
    params.FullySpecified.InterruptMode = LevelSensitive;
    params.FullySpecified.ShareVector = FALSE;
    params.FullySpecified.Vector = Vector;
    params.FullySpecified.Irql = Irql;
    params.FullySpecified.Affinity = (KAFFINITY)-1;     // any processor
    params.FullySpecified.FloatingSave = FALSE;
    return IoConnectInterruptEx(&params);
}

static VOID DriverUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\SpinDev");
    PDEVICE_OBJECT dev = DriverObject->DeviceObject;

    IoDeleteSymbolicLink(&linkName);
    while (dev != NULL) {
        PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)dev->DeviceExtension;
        PDEVICE_OBJECT next = dev->NextDevice;

        if (dx->InterruptObject != NULL) {
            IoDisconnectInterrupt(dx->InterruptObject);   // no new ISR from here on
            dx->InterruptObject = NULL;
            KeFlushQueuedDpcs();                          // let the queued DPC finish
        }
        if (dx->Registers != NULL) {
            ExFreePoolWithTag(dx->Registers, SPIN_TAG);
        }
        IoDeleteDevice(dev);
        dev = next;
    }
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING devName  = RTL_CONSTANT_STRING(L"\\Device\\SpinDev");
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(L"\\??\\SpinDev");
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
    dx->Self = dev;
    KeInitializeSpinLock(&dx->StateLock);
    KeInitializeSpinLock(&dx->InterruptLock);
    KeInitializeDpc(&dx->IsrDpc, IsrDpcRoutine, dx);

    dx->Registers = (PULONG)ExAllocatePool2(POOL_FLAG_NON_PAGED,
                                            REG_COUNT * sizeof(ULONG), SPIN_TAG);
    if (dx->Registers == NULL) {
        IoDeleteDevice(dev);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    status = IoCreateSymbolicLink(&linkName, &devName);
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(dx->Registers, SPIN_TAG);
        IoDeleteDevice(dev);
        return status;
    }

    // Optional interrupt: a missing vector leaves the stats path usable for testing.
    status = ConnectDeviceInterrupt(dx, TEST_VECTOR, TEST_IRQL);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_WARNING_LEVEL,
                   "SpinDev: no interrupt (%08X), stats only\n", status);
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE]         = DispatchCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]          = DispatchCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DispatchDeviceControl;

    dev->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

## Variations and advanced use

- **Queued versus classic.** Use `KeAcquireInStackQueuedSpinLock` for any lock that several
  processors can contend for: waiters spin on their own queue node instead of the shared
  word, which avoids the cache-line convoy and gives FIFO service. Use the classic
  `KeAcquireSpinLock` for a lock that is almost never contended or that must be taken from
  an ISR, where the queue handle would have to be built at `DIRQL`.
- **The AT-DPC-LEVEL forms.** Inside a DPC the IRQL is already 2, so raising it again is
  pure overhead. `KeAcquireSpinLockAtDpcLevel` and the queued `...AtDpcLevel` variants skip
  that bookkeeping and are the correct form; the matching `FromDpcLevel` release does not
  lower IRQL. Never use them from `PASSIVE_LEVEL` — they do not raise IRQL, so a DPC could
  preempt the owner and deadlock.
- **`KeSynchronizeExecution` versus `KeAcquireInterruptSpinLock`.** Both raise to the
  interrupt's `DIRQL` and mask its line. The first runs one callback and is the right choice
  for a register read-modify-write; the second lets a longer sequence run with the interrupt
  blocked, at the cost of holding `DIRQL` for its whole duration, which delays the ISR.
  Prefer the callback.
- **Per-processor data instead of a shared lock.** A counter that only needs eventual
  aggregation can be an array indexed by `KeGetCurrentProcessorNumber()`, with no lock at
  all; sum the slots when someone asks. This removes the contention entirely and is the
  standard answer for statistics.
- **Hold time and cache lines.** Keep a critical section under a few dozen instructions;
  an unbounded loop, a `DbgPrint` at a high filter level, or a cold pointer dereference
  inside the lock multiplies every waiter's latency. Put the lock in its own 64-byte cache
  line (`DECLSPEC_ALIGN(64)`) when it is heavily used and the neighboring fields are
  read-mostly, so readers do not invalidate the lock line and waiters do not invalidate the
  data line.
- **Backoff is not yours to implement.** The kernel's spin loop already uses a
  pause/`yield` strategy tuned per architecture. Adding your own retry loop around
  `KeTestSpinLock` produces worse throughput and can starve.

## Failure modes and diagnosis

| Symptom | Likely cause | First action |
|---|---|---|
| Hang at high IRQL, no bugcheck; one processor busy in `KeAcquireSpinLock` | recursive acquisition on the same processor, or a DPC interrupted the owner | `!analyze -hang`; `!stacks 2`; `!irql 0` |
| Bugcheck 0xA `IRQL_NOT_LESS_OR_EQUAL` after a release | release passed a literal IRQL instead of the saved value, so the processor ran pageable code at 2 | `!analyze -v`; check every `KeReleaseSpinLock` second argument |
| Bugcheck 0xD1 `DRIVER_IRQL_NOT_LESS_OR_EQUAL` inside a critical section | the protected code touched paged memory, a user buffer, or a pageable routine | `!analyze -v`; `!pool` the faulting address |
| Bugcheck 0xC2 `BAD_POOL_CALLER` | the extension was freed while a lock inside it was still held, or an overrun corrupted the block | `!analyze -v`; `!pool` the tag; special pool |
| Bugcheck 0xC1 `SPECIAL_POOL_DETECTED_MEMORY_CORRUPTION` | a write past the end of a locked structure, caught at free by special pool | `!analyze -v`; `!pool` both addresses; enable special pool for the tag |
| Bugcheck 0x101 `CLOCK_WATCHDOG_TIMEOUT` | a processor has held `DIRQL`/`DISPATCH_LEVEL` for tens of seconds, usually spinning on a lock | `!analyze -v`; `!stacks 2`; identify the spinning frame on the stuck CPU |
| Bugcheck 0x133 `DPC_WATCHDOG_VIOLATION` | a DPC ran far too long, often a loop that retries a lock or scans a list under it | `!analyze -v`; `!dpcs`; `!thread` on the DPC's processor |
| Bugcheck 0x1E right after `IoConnectInterruptEx` | `SpinLock` in the connect parameters was not initialized, or the ISR used a pageable routine | `!analyze -v`; verify `KeInitializeSpinLock` precedes the connect |
| Bugcheck 0x8E in an ISR or DPC | an exception (often an access violation on a freed extension) at `DIRQL` | `!analyze -v`; `!pool` the extension |
| Bugcheck 0x9F on sleep | an ISR still connected, or a DPC queued, while the device powered down | `!analyze -v`; `!irp`; check the power path ordering |
| `!locks` shows a resource owned by a thread that is also blocked | a spinlock is held while calling something that waits; the hang is not in the spinlock | `!locks`; `!stacks 2`; look for a wait inside the critical section |
| Verifier reports a spinlock violation (0xC4 with the spinlock rule) | double acquisition, release by a different processor, or uninitialized lock | `!analyze -v`; run `verifier /standard /driver SpinDev.sys` |
| Interrupts stop being delivered after a while | the ISR returned `TRUE` for a foreign interrupt and acknowledged it | `!analyze -v`; make the ISR check its own status bit first |

Diagnosis sequence for a suspected spinlock hang. Representative output, abbreviated:

```
kd> !analyze -hang
*** STACK VERIFICATION FAILED: a processor is spinning with interrupts disabled
kd> !stacks 2
 0.0   ffff8001a2b3c080 0x2f1  nt!KiSpinWithYieldHint
                                nt!KeAcquireSpinLockAtDpcLevel
                                SpinDev!IsrDpcRoutine+0x28
kd> !locks
KD: Scanning for held locks...
Resource @ ffff8001a2c45680    Exclusively owned
    Contention Count = 1
     Threads: ffff8001a2b3c080-01
```

Read it as: a DPC is spinning on a lock that another path already owns, and the owner cannot
run. `!locks` covers `ERESOURCE` objects; it will not show a bare `KSPIN_LOCK`, so the stack
of the spinning processor is the real evidence. Cross-check with `!thread` and `!irql` on
that processor, then find who holds the lock by inspection of the extension.

## Safety, IRQL, and cleanup checklist

- [ ] The lock and everything it protects are non-paged, and the lock is initialized once,
      before the device is exposed to any path that can acquire it.
- [ ] Every acquire has exactly one release with the **saved** IRQL, on every exit path.
- [ ] No lock is acquired twice on one path; no helper called under the lock re-acquires it.
- [ ] Nothing inside a critical section allocates, logs at high volume, completes an IRP,
      calls a driver below, touches user memory, or waits.
- [ ] No ISR-shared data is protected by a lock taken at `DISPATCH_LEVEL`; either the ISR
      avoids the lock or every path uses `KeSynchronizeExecution`.
- [ ] The ISR checks its own status bit and returns `FALSE` for interrupts it did not own.
- [ ] Unload order: disconnect the interrupt, `KeFlushQueuedDpcs`, then free the extension.
- [ ] Heavily used locks are separated from read-mostly fields by cache-line alignment.
- [ ] `Interlocked*` is used instead of a lock wherever one word is the whole story.

## Verification

```
msbuild SpinDev.vcxproj /p:Configuration=Debug /p:Platform=x64
sc create SpinDev type= kernel binPath= C:\test\SpinDev.sys
sc start SpinDev
```

Load with Driver Verifier enabled (`verifier /standard /driver SpinDev.sys`) so double
acquisition and release-by-non-owner are reported as 0xC4 instead of becoming a hang. Then
run the client below from four processes at once; the stats IOCTL takes the queued lock at
`PASSIVE_LEVEL` while the DPC takes it at `DISPATCH_LEVEL`, which is the pairing that must
not deadlock.

```c
// spintest.c  —  cl /W4 spintest.c
#include <windows.h>
#include <stdio.h>

typedef struct { ULONG Interrupts; ULONG LastStatus; } SPIN_STATS;
#define GET_STATS CTL_CODE(FILE_DEVICE_UNKNOWN, 0x900, METHOD_BUFFERED, FILE_READ_DATA)

int main(void)
{
    SPIN_STATS stats = { 0 };
    DWORD bytes = 0, i;
    HANDLE dev = CreateFileW(L"\\\\.\\SpinDev", GENERIC_READ | GENERIC_WRITE, 0,
                             NULL, OPEN_EXISTING, 0, NULL);

    if (dev == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }
    for (i = 0; i < 100000; ++i) {
        if (!DeviceIoControl(dev, GET_STATS, NULL, 0, &stats, sizeof(stats),
                             &bytes, NULL)) {
            printf("ioctl failed: %lu\n", GetLastError());
            CloseHandle(dev);
            return 1;
        }
    }
    CloseHandle(dev);
    printf("interrupts=%lu last=%08lx\n", stats.Interrupts, stats.LastStatus);
    return 0;
}
```

Expected results: the client completes with no timeout, `!stacks 2` shows no thread parked
inside `KeAcquireSpinLockAtDpcLevel`, and `!analyze -hang` on a forced hang reports no
spinning processor. `!irql` on each processor must read 0 when idle — a processor left at 2
after a driver call means a release restored the wrong IRQL.
