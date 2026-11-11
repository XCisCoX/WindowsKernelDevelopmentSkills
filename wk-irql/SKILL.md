---
name: wk-irql
description: Use when a driver bugchecks with IRQL_NOT_LESS_OR_EQUAL (0xA), DRIVER_IRQL_NOT_LESS_OR_EQUAL (0xD1), or PAGE_FAULT_IN_NONPAGED_AREA (0x50), or before calling any API from a DPC, ISR, timer, or completion routine. Covers level values, KeGetCurrentIrql, KeRaiseIrql/KeLowerIrql save-restore discipline, preemption and interrupt masking, per-processor semantics, an API-to-IRQL matrix, and a can-I-call-this-here procedure.
---

# IRQL: the execution-context contract every kernel routine obeys

IRQL is a priority stored in each processor, not in each thread, and it decides what the code running on that processor may legally do. Two rules cause almost every crash: memory that is not resident must never be touched at `DISPATCH_LEVEL` or above, and a processor at `DISPATCH_LEVEL` or above can never be rescheduled, so it can never wait. Read the current level before writing a line of code that allocates, waits, pages, or calls into the I/O manager. Raise only when required, save the old level in a local, restore it on every path including error paths, and never return from a routine at a higher level than it was entered.

## When to use

- Triage of `0xA`, `0xD1`, `0x50`, or a verifier `0xC4` where the faulting address looks valid.
- Deciding whether a routine may block, page, allocate paged pool, touch user memory, or call `Zw*`.
- Writing or reviewing a DPC, ISR, timer callback, `IO_COMPLETION_ROUTINE`, or cancel routine.
- Adding `KeRaiseIrql`/`KeLowerIrql` or a spinlock around a shared counter or list.
- Choosing between per-processor state (raise only) and shared state (lock plus raise).
- Enabling verifier IRQL checking before a test pass.
- Do not use for: choosing a lock primitive (`wk-spinlocks`, `wk-synchronization`); the mechanics of deferred work (`wk-dpcs`, `wk-work-items`); APC delivery rules (`wk-apcs`).

## Prerequisites and related skills

- Kernel C with `<ntddk.h>`; WDK 10 targeting Windows 7 or later. All values below are stable across those versions.
- You must know the entry IRQL of the routine being edited. If the routine is reachable from a DPC, an ISR, a completion routine, or a cancel routine, assume `DISPATCH_LEVEL`.
- `KIRQL` is `UCHAR`; `PASSIVE_LEVEL` is `0`, so a `KIRQL` cannot be negative and comparisons are unsigned.
- Sibling skills: `wk-spinlocks` (spinlock acquisition raises IRQL and restores it), `wk-dpcs` (the main source of DISPATCH code), `wk-apcs` (what `APC_LEVEL` blocks), `wk-work-items` (how to get back down to `PASSIVE_LEVEL`), `wk-kernel-debugging` (`!irql`), `wk-crash-dump-analysis` (reading the dump).

## Mental model

Each processor holds one IRQL value. That value is saved and restored around every interrupt: the interrupt controller delivers a vector, the CPU raises its level to the vector's IRQL, runs the ISR, and drops back. Code therefore never "holds" an IRQL across a context switch, because a context switch cannot happen at `DISPATCH_LEVEL` or above in the first place.

```text
  IRQL   x64/ARM64  x86   who runs here                    switched?  paged access?
  HIGH      15        31    interlocked list primitives            no                 no
  IPI       14        14    cross-processor interrupt (reserved)   no                 no
  CLOCK     13        13    clock/quantum ISR (reserved)           no                 no
  DIRQL   3..11      3..26  device ISRs from the HAL              no                 no
  DISPATCH   2         2    DPCs, scheduler, deferred work          no                 no
  APC        1         1    special kernel APCs                     yes                yes
  PASSIVE    0         0    threads, user mode, most driver code    yes                yes
```

Three consequences follow directly from that picture.

1. **Interrupt masking is local and inclusive.** A processor at level *N* ignores every interrupt whose IRQL is `<= N` and services only higher ones. So raising to `DISPATCH_LEVEL` blocks the clock and every device interrupt at `DISPATCH_LEVEL` or below, but it does not stop a `DIRQL` ISR from preempting you.
2. **Preemption requires a scheduling decision, and the dispatcher runs at `DISPATCH_LEVEL`.** At `APC_LEVEL` the thread can still be switched out; from `DISPATCH_LEVEL` up it cannot, so the thread has an effectively infinite quantum until it lowers.
3. **Paging requires the memory manager, which requires a context switch.** A not-present address faults, the fault handler must block the faulting thread while the pager reads the disk, and blocking needs a context switch. At `DISPATCH_LEVEL` and above that is impossible, so the kernel bugchecks instead.

Because level is a processor attribute, raising it gives you **no protection against another processor**. Two CPUs can both be at `PASSIVE_LEVEL` in your routine; raising on one does nothing to the other. Use a raise to protect per-processor state and to block same-processor ISR/DPC re-entry; use a spinlock (which also raises) to protect state shared across processors.

What each level actually permits, in one matrix:

| Capability | PASSIVE (0) | APC (1) | DISPATCH (2) | DIRQL (3+) |
|---|---|---|---|---|
| Access paged pool, pageable code, pageable data | yes | yes | **no** | **no** |
| Survive a fault on a not-present address | recoverable | recoverable | **fatal** | **fatal** |
| Dereference a raw user pointer | only with probe and `__try` | only with probe and `__try` | **no** | **no** |
| Wait on a dispatcher object with a timeout | yes | yes, deadlock-prone | **no** (zero timeout only) | **no** |
| `KeDelayExecutionThread` | yes | no | no | no |
| File, registry, `Zw*` calls | yes | no | no | no |
| Allocate paged pool | yes | yes | **no** | **no** |
| Allocate non-paged pool | yes | yes | yes | no |
| Acquire a spinlock | yes | yes | yes | use the interrupt lock |
| Take a fast mutex | yes | yes | no | no |
| Take a mutex or `ERESOURCE` | yes | no | no | no |
| Receive a user or normal kernel APC | yes | **no** | no | no |
| Be preempted by the scheduler | yes | yes | **no** | **no** |
| Call `IoCompleteRequest` | yes | yes | yes | **no** |

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `KeGetCurrentIrql()` | Read this processor's level | any | `UCHAR` return; cheap, one load |
| `KeRaiseIrql(New, &Old)` | Raise locally, save previous | any | Contract: `New >= current`; never lowers |
| `KeLowerIrql(New)` | Restore a saved level | any | Contract: `New <= current`; fatal otherwise |
| `KeRaiseIrqlToDpcLevel()` | Raise to `DISPATCH_LEVEL` | any | Returns the old level directly |
| `KeGetCurrentProcessorNumber()` | Processor index | any | `<= 63` on x64; never assume `0` |
| `KeStallExecutionProcessor(Us)` | Busy-wait microseconds | any | Never use it to wait for an event |
| `NT_ASSERT(KeGetCurrentIrql() == X)` | Checked-build assertion | any | Free builds compile it out |
| `KIRQL` / `PKIRQL` | Type of a saved level | n/a | `UCHAR`; always initialize the local |

Level names and constants that matter:

| Constant | Value | Meaning in practice |
|---|---|---|
| `PASSIVE_LEVEL` | 0 | Threads run, paging and blocking allowed |
| `LOW_LEVEL` | 0 | Alias used as a "no requirement" parameter, never returned |
| `APC_LEVEL` | 1 | Special kernel APCs; all APC delivery is blocked at or above this |
| `DISPATCH_LEVEL` | 2 | DPCs and the scheduler; no paging, no blocking waits |
| `CMCI_LEVEL` | 5 | Corrected machine-check interrupt on modern x64 |
| `DIRQL` | 3..11 / 3..26 | Per-device, assigned by the HAL; not a compile-time constant |
| `HIGH_LEVEL` | 15 / 31 | Masks everything; only internal primitives run here |

API-to-IRQL matrix. "Min" is the lowest level the call is legal at, "Max" the highest. Where a call has a level-dependent variant, both rows are shown.

| API | Min | Max | Notes |
|---|---|---|---|
| `KeGetCurrentIrql` | any | any | Pure read |
| `KeRaiseIrql` | any | any | `New >= current` |
| `KeLowerIrql` | any | any | `New <= current` |
| `KeAcquireSpinLock` | 0 | 2 | Raises to 2, returns old level in the out parameter |
| `KeReleaseSpinLock` | 2 | 2 | Call at the level the acquire established |
| `KeAcquireSpinLockAtDpcLevel` | 2 | any | Assumes you are already at `>= 2` |
| `KeReleaseSpinLockFromDpcLevel` | 2 | any | Does not touch IRQL |
| `KeAcquireInStackQueuedSpinLock` | 0 | 2 | Preferred over the legacy lock on SMP |
| `KeAcquireInterruptSpinLock` | 0 | DIRQL | Raises to that interrupt's DIRQL |
| `KeSynchronizeExecution` | 0 | DIRQL | Runs your callback at the interrupt's DIRQL |
| `ExAcquireFastMutex` | 0 | 1 | Raises to 1; no recursion |
| `ExReleaseFastMutex` | 1 | 1 | Must be the owner |
| `KeEnterCriticalRegion` | 0 | 1 | Blocks normal kernel and user APCs |
| `KeEnterGuardedRegion` | 0 | 1 | Blocks all APCs, including special |
| `KeAreApcsDisabled` | any | any | Nonzero when in either region |
| `KeWaitForSingleObject` | 0 | 2 | At 2 only a zero timeout is allowed |
| `KeDelayExecutionThread` | 0 | 0 | `PASSIVE_LEVEL` only |
| `KeStallExecutionProcessor` | any | any | Busy wait, no scheduler involvement |
| `KeQueryPerformanceCounter` | any | any | Monotonic, no allocation |
| `KeQueryInterruptTime` | any | any | Does not change with clock adjustments |
| `ExAllocatePool2` non-paged | 0 | 2 | `POOL_FLAG_NON_PAGED` |
| `ExAllocatePool2` paged | 0 | 1 | Never at `DISPATCH_LEVEL` |
| `ExFreePoolWithTag` non-paged | any | 2 | Must match the allocation site |
| `ExFreePoolWithTag` paged | 0 | 1 | Freeing paged pool at 2 is a bugcheck |
| `IoAllocateIrp` | 0 | 2 | Also `IoAllocateMdl`, `IoFreeIrp` |
| `IoCallDriver` | 0 | 2 | Lower driver inherits your level |
| `IoCompleteRequest` | 0 | 2 | The reason DPCs exist |
| `MmProbeAndLockPages` | 0 | 1 | Wrap in `__try`/`__except` |
| `MmGetSystemAddressForMdlSafe` | 0 | 2 | No new mapping at 2, may return `NULL` |
| `IoAcquireCancelSpinLock` | 0 | 2 | Deprecated; use `IO_CSQ` |
| `ZwCreateFile` / `ZwReadFile` / `ZwWriteFile` | 0 | 0 | `PASSIVE_LEVEL` only |
| `ZwSetValueKey` / `ZwQueryValueKey` | 0 | 0 | Registry calls are passive |
| `ZwQuerySystemInformation` | 0 | 0 | Also `ZwClose` |
| `PsCreateSystemThread` | 0 | 0 | Thread start routines run at 0 |
| `KeSetTimer` / `KeCancelTimer` | 0 | 2 | Safe from inside a DPC |
| `KeInitializeDpc` / `KeInitializeTimer` | any | any | Initialization only |
| `DbgPrint` / `DbgPrintEx` | any | any | Serialized and slow; keep it out of hot ISRs |

## Step-by-step

1. **Read the entry level.** Put `KIRQL entryIrql = KeGetCurrentIrql();` at the top of the routine and, in a checked build, `NT_ASSERT(entryIrql <= DISPATCH_LEVEL);` if the routine is documented to run at `DISPATCH_LEVEL` or below. Do this before any other work.
2. **Classify the routine.** Which callers exist? A dispatch routine reachable from user mode is `PASSIVE`. Anything called from `DeferredRoutine`, `IO_COMPLETION_ROUTINE`, a cancel routine, or a timer is `DISPATCH`. Anything called under `KeSynchronizeExecution` is at `DIRQL` and must never touch paged memory or take another lock.
3. **Pick the requirement, then the mechanism.** If the work pages, blocks, or calls `Zw*`, it must run at `PASSIVE_LEVEL`: move it to a work item or a system thread (`wk-work-items`, `wk-kernel-threads`) instead of raising. If it only needs to be atomic against this processor's ISR/DPC, raise. If it must be atomic against other processors, take a spinlock, which raises for you.
4. **Save, work, restore.** Declare `KIRQL old;`, call `KeRaiseIrql(Level, &old);`, do the work, then `KeLowerIrql(old);`. Restore the saved value, never a hard-coded `PASSIVE_LEVEL`, because the routine may already have been called at a raised level.
5. **Assert both sides.** `NT_ASSERT(KeGetCurrentIrql() == Level)` after the raise and `NT_ASSERT(KeGetCurrentIrql() <= entryIrql)` after the restore.
6. **Audit every exit path.** An early `return` between the raise and the lower leaks a raised IRQL onto the processor; when the caller returns to `PASSIVE` code the mismatch eventually produces `0xA` or a hang. Use a single exit block or a `__try`/`__finally`.
7. **Re-check the callees against the matrix.** Every call inside the raised region must have a `Max` at or above your level. Pay attention to `IoCompleteRequest` (2), `ExFreePoolWithTag` (1 for paged), and any accessor that silently touches paged memory.

### "Can I call this here?" procedure

Use this five-question sequence in order; stop at the first "no".

1. **What is `KeGetCurrentIrql()` right now?** If you cannot answer, the API call is not ready to be written. In a routine whose callers are unknown, gate the call with a runtime check and fail cleanly.
2. **Is the required maximum `>=` current?** Look the API up in the matrix. A `Max` of 2 means it may run in a DPC but not in an ISR.
3. **Does it touch memory that can be paged?** Paged pool, pageable sections, user buffers, and pageable code are all illegal at `>= 2`. A `METHOD_BUFFERED` system buffer is non-paged and is the safe choice for DPC completion; a `METHOD_NEITHER` user buffer is not.
4. **Can it block, and does the implementation wait internally?** Anything that acquires a mutex, a fast mutex, an `ERESOURCE`, or a file object can block inside even if the prototype looks harmless. File I/O, registry I/O, and `Zw*` all wait.
5. **Does it lower IRQL, raise it, or change APC state?** A call that takes a spinlock restores the level on release; a call that enters a critical region changes APC delivery, which matters when the caller is already inside one.

```c
typedef enum _IRQL_NEED {
    IrqlNeedAny = 0,      // callable anywhere
    IrqlNeedApc,          // must be at or below APC_LEVEL
    IrqlNeedPassive       // must be exactly PASSIVE_LEVEL
} IRQL_NEED;

static BOOLEAN IrqlAllows(_In_ IRQL_NEED Need)
{
    KIRQL now = KeGetCurrentIrql();

    switch (Need) {
    case IrqlNeedAny:
        return TRUE;
    case IrqlNeedApc:
        return now <= APC_LEVEL;
    case IrqlNeedPassive:
        return now == PASSIVE_LEVEL;
    default:
        return FALSE;
    }
}
```

Call it as `if (!IrqlAllows(IrqlNeedPassive)) { return STATUS_INVALID_DEVICE_STATE; }` before the operation, and log the level so the failure is diagnosable instead of mysterious.

## Complete example

A legacy WDM driver that keeps a per-processor counter (protected by a raise) and a global counter (protected by a spinlock), and reports the current level to a client.

```c
//
// irqldemo.c -- IRQL discipline: save/restore, legality checks, per-processor state.
//
#include <ntddk.h>

#define IRQLDEMO_TAG      'mQrI'
#define IRQLDEMO_DEVNAME  L"\\Device\\IrqlDemo"
#define IRQLDEMO_LINK     L"\\DosDevices\\IrqlDemo"
#define IRQLDEMO_MAX_CPU  64

#define IOCTL_IRQL_QUERY \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_IRQL_BUMP_CPU \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_IRQL_BUMP_GLOBAL \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x802, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _IRQL_QUERY {
    ULONG CurrentIrql;
    ULONG ProcessorNumber;
    ULONG CpuCount;
    ULONG GlobalCount;
} IRQL_QUERY, *PIRQL_QUERY;

typedef struct _IRQL_EXTENSION {
    KSPIN_LOCK    GlobalLock;
    LONG          GlobalCount;                    /* guarded by GlobalLock */
    volatile LONG CpuCount[IRQLDEMO_MAX_CPU];     /* one slot per processor */
} IRQL_EXTENSION, *PIRQL_EXTENSION;

/*
 * Per-processor state needs only a raise. The slot belongs to this processor,
 * so masking this processor's DPCs is the entire mutual-exclusion requirement.
 */
static LONG BumpThisProcessor(_In_ PIRQL_EXTENSION Ext)
{
    KIRQL irql;
    ULONG cpu;
    LONG  value;

    if (KeGetCurrentIrql() > DISPATCH_LEVEL) {
        return -1;                      /* raising downward is never legal */
    }

    cpu = KeGetCurrentProcessorNumber();
    if (cpu >= IRQLDEMO_MAX_CPU) {
        return -1;
    }

    KeRaiseIrql(DISPATCH_LEVEL, &irql);
    NT_ASSERT(KeGetCurrentIrql() == DISPATCH_LEVEL);

    value = InterlockedIncrement(&Ext->CpuCount[cpu]);

    KeLowerIrql(irql);
    NT_ASSERT(KeGetCurrentIrql() <= DISPATCH_LEVEL);
    return value;
}

/*
 * Shared state needs a lock. KeAcquireSpinLock raises to DISPATCH_LEVEL and
 * hands back the previous level, which is what must be restored on release.
 */
static LONG BumpGlobal(_In_ PIRQL_EXTENSION Ext)
{
    KIRQL irql;
    LONG  value;

    KeAcquireSpinLock(&Ext->GlobalLock, &irql);
    value = InterlockedIncrement(&Ext->GlobalCount);
    KeReleaseSpinLock(&Ext->GlobalLock, irql);
    return value;
}

static NTSTATUS IrqlDemoCreateClose(_In_ PDEVICE_OBJECT DeviceObject,
                                    _Inout_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS IrqlDemoControl(_In_ PDEVICE_OBJECT DeviceObject,
                                _Inout_ PIRP Irp)
{
    PIRQL_EXTENSION   ext = (PIRQL_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    PIRQL_QUERY       out;
    NTSTATUS          status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG             code = stack->Parameters.DeviceIoControl.IoControlCode;
    ULONG             outLen = stack->Parameters.DeviceIoControl.OutputBufferLength;
    ULONG             info = 0;

    if (KeGetCurrentIrql() != PASSIVE_LEVEL) {
        status = STATUS_INVALID_DEVICE_STATE;      /* dispatch must be passive */
        goto done;
    }

    if (code == IOCTL_IRQL_BUMP_CPU) {
        status = (BumpThisProcessor(ext) < 0) ? STATUS_INVALID_DEVICE_STATE
                                              : STATUS_SUCCESS;
        goto done;
    }

    if (code == IOCTL_IRQL_BUMP_GLOBAL) {
        (VOID)BumpGlobal(ext);
        status = STATUS_SUCCESS;
        goto done;
    }

    if (code != IOCTL_IRQL_QUERY) {
        goto done;
    }

    if (Irp->AssociatedIrp.SystemBuffer == NULL || outLen < sizeof(IRQL_QUERY)) {
        status = STATUS_BUFFER_TOO_SMALL;
        goto done;
    }

    out = (PIRQL_QUERY)Irp->AssociatedIrp.SystemBuffer;
    RtlZeroMemory(out, sizeof(*out));
    out->CurrentIrql = (ULONG)KeGetCurrentIrql();
    out->ProcessorNumber = KeGetCurrentProcessorNumber();
    out->CpuCount = (ULONG)InterlockedCompareExchange(
                        &ext->CpuCount[out->ProcessorNumber], 0, 0);
    out->GlobalCount = (ULONG)InterlockedCompareExchange(&ext->GlobalCount, 0, 0);
    info = sizeof(*out);
    status = STATUS_SUCCESS;

done:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = info;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static VOID IrqlDemoUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING link;

    RtlInitUnicodeString(&link, IRQLDEMO_LINK);
    IoDeleteSymbolicLink(&link);
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                    _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT   device = NULL;
    PIRQL_EXTENSION  ext;
    UNICODE_STRING   devName;
    UNICODE_STRING   link;
    NTSTATUS         status;
    ULONG            i;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, IRQLDEMO_DEVNAME);
    status = IoCreateDevice(DriverObject, sizeof(IRQL_EXTENSION), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PIRQL_EXTENSION)device->DeviceExtension;
    KeInitializeSpinLock(&ext->GlobalLock);
    ext->GlobalCount = 0;
    for (i = 0; i < IRQLDEMO_MAX_CPU; i++) {
        ext->CpuCount[i] = 0;
    }

    RtlInitUnicodeString(&link, IRQLDEMO_LINK);
    status = IoCreateSymbolicLink(&link, &devName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = IrqlDemoCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = IrqlDemoCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IrqlDemoControl;
    DriverObject->DriverUnload = IrqlDemoUnload;

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

`IOCTL_IRQL_QUERY` runs in a dispatch path entered from user mode, so `CurrentIrql` comes back as `0`. The same routine reached from a DPC would see `2` and the guard at the top would reject the call with `STATUS_INVALID_DEVICE_STATE` instead of raising into an illegal direction. That guard is exactly the kind of check to leave in while the driver is new and the callers are still being enumerated.

## Variations and advanced use

- **`KeRaiseIrqlToDpcLevel()` vs `KeRaiseIrql(DISPATCH_LEVEL, &old)`.** Identical effect; the first returns the old level as its value and is convenient inside a helper that already returns a `KIRQL`. Both must be paired with `KeLowerIrql(old)`.
- **Raise as a lock replacement.** Legal only for per-processor data. If the object can be reached from another processor, use a spinlock; the lock raises to `DISPATCH_LEVEL` internally, so the two techniques compose: `KeAcquireSpinLock` is a raise plus cross-processor exclusion.
- **`DIRQL` code.** Inside `KeSynchronizeExecution` or an ISR, the lock protecting the device is already held by the interrupt. Never call `KeAcquireSpinLock` there: it raises to `DISPATCH_LEVEL`, which is *lower* than the current level and therefore an illegal raise. Use the interrupt spinlock semantics from `wk-spinlocks`.
- **`HIGH_LEVEL`.** Only internal interlocked list primitives run there. There is no reason for a driver to raise to it, and no safe way to do work at it.
- **Fault-tolerant probing.** Use `IrqlAllows` to reject work with `STATUS_INVALID_DEVICE_STATE` rather than bugchecking. A driver that degrades at the wrong IRQL is diagnosable; one that bugchecks is not.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `0xA IRQL_NOT_LESS_OR_EQUAL`, arg2 = 2 | Paged or invalid address touched at `DISPATCH_LEVEL` | `!analyze -v`, then `!irql`; move the access to a DPC-safe copy in non-paged memory |
| `0xD1 DRIVER_IRQL_NOT_LESS_OR_EQUAL` | Same fault, but the instruction is attributed to your driver | `!analyze -v` shows the offending module; check that pointer's allocation site |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA`, arg1 = freed address | Use-after-free, frequently after a raise | `!pool <addr>`, enable special pool |
| `0xC4` verifier violation with an IRQL subcode | Verifier caught an IRQL rule before hardware did | `verifier /flags 0x4 /driver yourdrv.sys`, reboot, rerun |
| `0xC2` `BAD_POOL_CALLER` | Paged-pool free or allocation at `DISPATCH_LEVEL` | `!analyze -v` argument 1; move allocation to `PASSIVE_LEVEL` |
| `0x1A` `MEMORY_MANAGEMENT` | PTE corruption from touching paged memory at high IRQL | `!pte`, `!analyze -v`; look for a raise without a restore |
| `0x133` `DPC_WATCHDOG_VIOLATION` | A DPC ran far too long, often because it blocked or looped | `!analyze -v`, `!dpcs`; shorten the DPC and defer to a work item |
| `0x101` `CLOCK_WATCHDOG_TIMEOUT` | A processor stuck at high IRQL, usually a lost `KeLowerIrql` | `!analyze -v`, `!irql` on every processor |
| Client call hangs instead of failing | Raise without a matching lower; the thread never runs again | `!irql`; look for an early `return` between raise and lower |
| `0x1E` / `0x8E` with a driver frame | An exception raised inside a raised region, with no handler | Wrap user-memory access in `__try`/`__except` (`wk-kernel-exceptions`) |

## Safety, IRQL, and cleanup checklist

- [ ] Every `KeRaiseIrql` has exactly one `KeLowerIrql` on every path, including every error return.
- [ ] The lower restores the saved `KIRQL`, never `PASSIVE_LEVEL` literally.
- [ ] No `return` between raise and lower without going through the cleanup block.
- [ ] Nothing inside a `>= DISPATCH_LEVEL` region touches paged pool, a pageable section, a user buffer, or pageable code (`PAGED_CODE()` is absent from routines that can run there).
- [ ] No wait, mutex, fast mutex, `ERESOURCE`, file I/O, registry call, or `Zw*` at `>= DISPATCH_LEVEL`.
- [ ] `IoCompleteRequest` only at `<= DISPATCH_LEVEL`.
- [ ] Buffers used from a DPC are `METHOD_BUFFERED` system buffers or explicitly locked non-paged memory.
- [ ] Any state reachable from another processor is protected by a lock, not only by a raise.
- [ ] `KeEnterCriticalRegion` and `KeEnterGuardedRegion` are balanced, and no blocking call happens inside either.
- [ ] Checks are compiled out in release: `NT_ASSERT` alone is not a safety net, so runtime `if` guards exist for the paths that matter.

## Verification

1. Build: `msbuild irqldemo.vcxproj /p:Configuration=Debug /p:Platform=x64`, then sign and install with your test certificate (see `wk-driver-build-deploy`).
2. Enable IRQL checking so violations become `0xC4` before they become bugchecks:
   `verifier /flags 0x4 /driver irqldemo.sys` then reboot.
3. Start the driver and run the client against `\\.\IrqlDemo`, issuing `IOCTL_IRQL_QUERY` before and after `IOCTL_IRQL_BUMP_CPU`. Expected: `CurrentIrql` = 0 both times (the IOCTL runs at passive), and `CpuCount` increments by exactly 1 per call.
4. Attach a kernel debugger (`wk-kernel-debugging`), set `bp irqldemo!BumpThisProcessor`, and on the break run `!irql`; it must print `PASSIVE_LEVEL`. Step over the `KeRaiseIrql` and run `!irql` again; it must print `DISPATCH_LEVEL`. Step over the `KeLowerIrql` and confirm it returns to `PASSIVE_LEVEL`.
5. Stress it: run four client threads calling both bump IOCTLs one million times, then `!irql 0`, `!irql 1`, `!irql 2`, `!irql 3` on a four-processor VM. Every processor must report `PASSIVE_LEVEL` when idle. A processor stuck at `2` is a leaked raise.
6. Confirm the global counter equals the total bump calls and each per-processor slot equals the calls made on that processor; a mismatch means the protection is wrong, not the counter.
