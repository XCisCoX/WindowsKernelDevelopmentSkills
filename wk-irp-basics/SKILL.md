---
name: wk-irp-basics
description: Use when writing or auditing any WDM dispatch, completion, or pass-through routine. Covers the IRP and IO_STACK_LOCATION field layout and ownership, the request path from issuer to IoCompleteRequest, StackSize rules, IoSkipCurrentIrpStackLocation versus IoCopyCurrentIrpStackLocationToNext, pending and cancel discipline, STATUS_MORE_PROCESSING_REQUIRED, and debugger inspection with !irp, dt nt!_IRP, !irpfind, and !devstack.
---

# IRPs: the request object, the stack, and the completion path

Every request a driver serves arrives as an **IRP**: a non-paged header followed by an array of
`IO_STACK_LOCATION` records, one per device in the chain. The I/O manager fills the header and the
first record; each driver reads its own record, optionally prepares the next one, and either completes
the request or hands the IRP down with `IoCallDriver`. Completion runs the same chain in reverse.

The rule that prevents most disasters: **whoever allocates the IRP frees it, and after
`IoCompleteRequest` returns the IRP pointer is dead — never read `Irp->IoStatus.Status` to decide what
to return.** Capture the status in a local first. The second rule: **an IRP that is returned with
`STATUS_PENDING` must have been marked with `IoMarkIrpPending` before the first instruction that could
complete it.**

## When to use

- You are writing any `DRIVER_DISPATCH` routine: create, close, read, write, device control, PnP, power.
- You need to forward a request to the device below and inspect the result afterwards.
- A client hangs in `ReadFile`/`DeviceIoControl` and the IRP is never completed.
- A bugcheck points at a dispatch or completion routine (`0x8E`, `0xA`, `0xD1`, `0x50`, `0xCE`).
- You need to explain what `!irp` prints, or how many stack locations an IRP should have.
- You are building a filter or bus driver that must not disturb the stack locations it does not own.
- You are deciding between `IoSkipCurrentIrpStackLocation` and `IoCopyCurrentIrpStackLocationToNext`.

Do not use for buffer selection and MDLs (`wk-irp-buffering`), cancel-safe queues
(`wk-irp-queues-cancel`), or the device and symbolic-link lifecycle (`wk-device-objects`).

## Prerequisites and related skills

- **Headers:** `<ntddk.h>` (or `<wdm.h>` inside a miniport). The macros discussed here
  (`IoGetCurrentIrpStackLocation`, `IoGetNextIrpStackLocation`, `IoSkipCurrentIrpStackLocation`,
  `IoCopyCurrentIrpStackLocationToNext`, `IoMarkIrpPending`) are defined in `wdm.h`.
- **IRQL:** a dispatch routine usually runs at `PASSIVE_LEVEL` in the requesting thread, but the
  contract allows `DISPATCH_LEVEL` in an arbitrary thread (paging I/O, a lower filter that queued the
  request, a DPC that sent its own IRP). A completion routine runs at `IRQL <= DISPATCH_LEVEL` in an
  arbitrary thread. **Design every routine for the worst case**: no pageable code, no blocking waits,
  no `PAGED_CODE()` assumption unless you know the request source.
- **WDK:** `IoSetCompletionRoutineEx` requires Windows XP SP1 or later; `IoAllocateIrp` with
  quota charging is ancient and safe everywhere. Field layouts below are x64 (`sizeof(IRP) == 0x90`,
  `sizeof(IO_STACK_LOCATION) == 0x48`, so a one-location IRP is `Size == 0xD8`).
- Siblings: `wk-irp-buffering` (where the bytes live), `wk-irp-queues-cancel` (pending queues and
  cancellation), `wk-driver-anatomy` (device and unload lifecycle), `wk-kernel-debugging` (breakpoints
  and stepping in a dispatch routine), `wk-irql` (what may be called at each level).

## Mental model

### One allocation, two structures

```
PIRP ----------------------------------------------------------------+
 | Type | Size | AllocationProcessorNumber | MdlAddress | Flags        |
 | AssociatedIrp (SystemBuffer | MasterIrp | IrpCount)                |
 | IoStatus { Status, Information } | RequestorMode | PendingReturned |
 | StackCount | CurrentLocation | Cancel | CancelIrql | CancelRoutine  |
 | UserBuffer | Tail.Overlay { CurrentStackLocation, Thread, ... }    |
 +--------------------------------------------------------------------+
   ... immediately followed in the same pool block by:
   [ IO_STACK_LOCATION 0 ][ 1 ][ 2 ] ... [ StackCount - 1 ]
     ^ CurrentLocation = 0      ...          ^ bottom of the stack (PDO)
```

The header holds facts about the *request as a whole*: who asked, in which mode, with which buffer,
what the final status was. The stack array holds facts about *one layer*: which major function this
layer sees, that layer's parameters, that layer's device and file object, and the completion routine
the layer above installed.

`CurrentLocation` indexes the array. `IoCallDriver(TargetDevice, Irp)` decrements it and invokes
`TargetDevice->DriverObject->MajorFunction[IrpSp->MajorFunction]`. `IoCompleteRequest` walks it back
up, one step per layer, invoking each layer's completion routine on the way.

### The path of one request

```
client thread      I/O manager                 top driver             lower driver
    |                   |                          |                       |
    | ReadFile(...)     |                          |                       |
    |------------------>| NtReadFile               |                       |
    |                   | resolve handle->FILE_OBJECT->top DEVICE_OBJECT    |
    |                   | IoAllocateIrp(top->StackSize, FALSE)              |
    |                   | fill header: RequestorMode, UserBuffer/MDL,      |
    |                   |   IoStatus, Tail.Overlay.Thread                  |
    |                   | fill stack[0]; CurrentLocation = 0                |
    |                   |------------------------->| Dispatch(DevObj, Irp) |
    |                   |                          | sp = IoGetCurrentIrpStackLocation
    |                   |                          | validate, log         |
    |                   |                          | IoCopyCurrentIrpStackLocationToNext
    |                   |                          | IoSetCompletionRoutine(next)
    |                   |                          | IoCallDriver(lower, Irp)
    |                   |                          |---------------------->| CurrentLocation = 1
    |                   |                          |                       | Dispatch -> set IoStatus
    |                   |                          |                       | IoCompleteRequest
    |                   |                          |<----------------------| walk back up
    |                   |                          | completion routine runs (owns stack[1])
    |                   |<-------------------------| return status         |
    |<------------------| copy back SystemBuffer, signal event, free IRP   |
```

Four invariants follow from that picture:

1. **Only the issuer initialises the header.** The I/O manager initialises stack[0] only; every driver
   below that must be handed a prepared stack location by the layer above — that is what the copy and
   skip macros are for.
2. **Every layer owns exactly one stack location while it executes.** Write only to
   `IoGetNextIrpStackLocation(Irp)`, never to your own after you have called `IoCallDriver`.
3. **The completing driver decides the status**, and every layer between it and the issuer sees it in
   `IoStatus.Status`. Intermediate layers may change it; turning a success into a failure after a
   partial transfer confuses both the client and the I/O manager.
4. **Ownership is single-threaded through the stack.** At any instant exactly one layer is responsible
   for the IRP: executing in its dispatch, queued in it, or queued in a lower layer. Unload is safe
   only when no IRP is in any of those states (`0xCE`).

### Stack size: the number that must grow when you attach

`DEVICE_OBJECT.StackSize` is the count of stack locations an IRP needs to reach this device and
everything below it, counting this device's own location. `IoCreateDevice` sets it to 1, and a driver
that attaches on top of a lower device must raise it **before** the device can receive traffic:

```c
FilterDevice->StackSize = LowerDevice->StackSize + 1;
```

Rules that keep this correct:

- Set `StackSize` immediately after `IoAttachDeviceToDeviceStack` returns and before
  `IoCreateSymbolicLink` or `DO_DEVICE_INITIALIZING` is cleared; never change it after that.
- A `StackSize` that is too small makes `IoCallDriver` write past the last record and corrupt the pool
  block that follows. The symptom is rarely at the attach site: expect `0xC2`/`0x50` in an unrelated
  allocation later, or an `0x8E` inside `IofCallDriver`.
- For an IRP you allocate with `IoAllocateIrp(StackSize, FALSE)`, pass the target device's `StackSize`
  and fill `IoGetNextIrpStackLocation(Irp)` before the first `IoCallDriver` — after allocation
  `CurrentLocation == StackCount`, so "next" is the last record.

### Skip versus copy

| | `IoSkipCurrentIrpStackLocation` | `IoCopyCurrentIrpStackLocationToNext` |
|---|---|---|
| Effect | decrements `CurrentLocation` so the layer below sees the **same** record this layer saw | copies major/minor/flags/parameters/device/file to the next record, zeroes that record's `Control` |
| Cost | one decrement | `RtlCopyMemory` of everything before `CompletionRoutine` |
| Completion routine | **cannot** be installed in the removed record — do not skip when you need one | install it after the copy, in the next record |
| Who completes | the layer below completes and returns up as if this layer never existed | this layer's completion routine runs on the way back |
| Return value | return exactly what `IoCallDriver` returned, including `STATUS_PENDING` | return what `IoCallDriver` returned; propagate pending in the completion routine |
| Use for | a filter that does not care about the result; PnP and power passthrough | logging, fixing up `IoStatus`, retry, resource release after lower completion |

Two ordering facts that cause real bugs:

- Call `IoCopyCurrentIrpStackLocationToNext` **first**, then `IoSetCompletionRoutine`. The copy stops
  before the `CompletionRoutine`/`Context` fields (so it does not erase them) but it **zeroes
  `Control`**, which would wipe the `SL_INVOKE_ON_*` bits that `IoSetCompletionRoutine` just set.
- Do not call both macros on the same IRP, and do not copy a stack location you already skipped.

### Pending

`STATUS_PENDING` and `IoMarkIrpPending` always travel together:

- Mark **before** the first point at which the IRP could be completed by someone else (typically before
  queueing it or before releasing a lock that protects the queue).
- Return `STATUS_PENDING` from the dispatch routine.
- If a lower driver returned `STATUS_PENDING` and you used copy-with-completion, the completion
  routine must do `if (Irp->PendingReturned) IoMarkIrpPending(Irp);` — otherwise the I/O manager
  believes the request completed synchronously, frees the caller's stack frame assumptions, and the
  eventual `IoCompleteRequest` touches a dead `IO_STATUS_BLOCK`.
- If you skipped, you removed your own record, so the lower layer's pending state propagates without
  your help; just return the status you got.

A dispatch routine may return `STATUS_PENDING` **only** when it marked the IRP; returning it from a
routine that completes the request synchronously is a race the I/O manager can lose.

### Completion routines

`IO_COMPLETION_ROUTINE` has the prototype
`NTSTATUS (*)(PDEVICE_OBJECT DeviceObject, PIRP Irp, PVOID Context)`; `DeviceObject` is the device the
IRP was sent to, not necessarily yours, and `Context` is the pointer given to `IoSetCompletionRoutine`.
While it runs, `IoGetCurrentIrpStackLocation(Irp)` returns **your** record, so your `Context` and your
`MajorFunction` are both available.

| Return value | Meaning |
|---|---|
| `STATUS_SUCCESS` (a.k.a. `STATUS_CONTINUE_COMPLETION`) | keep bubbling up; the I/O manager continues the walk and will free the IRP |
| `STATUS_MORE_PROCESSING_REQUIRED` | stop the walk; **you now own the IRP** and must eventually call `IoCompleteRequest` again (to resume) or `IoFreeIrp` (if you allocated it) |
| `STATUS_MORE_PROCESSING_REQUIRED` after re-issuing the IRP | legal: complete the old one only when the new one finishes |

`IoSetCompletionRoutine`'s three booleans register which outcomes invoke you: success
(`NT_SUCCESS(IoStatus.Status)`), error, and cancel (`Irp->Cancel`). Setting all three to `TRUE` is the
common case; setting only success is how a driver avoids running cleanup twice on a cancellation.

`IoSetCompletionRoutineEx` exists for IRPs your driver allocates: it takes a reference on your driver
object so the completion routine cannot execute after the image is unloaded, and it **returns
`NTSTATUS`** — check it and do not send the IRP if it fails. `IoSetCompletionRoutine` has no such
protection.

### Priority boost

`IoCompleteRequest(Irp, Increment)` temporarily raises the priority of the issuing thread; a boost
never lifts a thread above priority 15 and has no effect above 15. A software driver completes on the
requesting thread, so `IO_NO_INCREMENT` (0) is right.

## Key APIs and structures

### `IRP` fields

| Field | Type (x64) | Meaning | Who writes it |
|---|---|---|---|
| `Type` | `CSHORT` | `IO_TYPE_IRP` (6) | I/O manager |
| `Size` | `CSHORT` | total bytes allocated: `sizeof(IRP) + StackCount * sizeof(IO_STACK_LOCATION)` | I/O manager; read-only |
| `AllocationProcessorNumber` | `ULONG` | CPU that allocated it (pool/NUMA heuristics) | read-only |
| `MdlAddress` | `PMDL` | MDL for Direct I/O, or one you built | I/O manager for read/write; you otherwise |
| `Flags` | `ULONG` | `IRP_BUFFERED_IO` 0x01, `IRP_DEALLOCATE_BUFFER` 0x02, `IRP_INPUT_OPERATION` 0x04, `IRP_SYNCHRONOUS_API` 0x08, `IRP_ASSOCIATED_IRP` 0x10 | I/O manager; treat as read-only |
| `AssociatedIrp.SystemBuffer` | `PVOID` | non-paged buffer for buffered I/O and `METHOD_BUFFERED` | I/O manager allocates and frees |
| `AssociatedIrp.MasterIrp` | `PIRP` | master of an associated-IRP group | I/O manager |
| `AssociatedIrp.IrpCount` | `LONG` | number of outstanding associated IRPs | I/O manager |
| `IoStatus.Status` | `NTSTATUS` | final status seen by the issuer | the completing driver |
| `IoStatus.Information` | `ULONG_PTR` | polymorphic: bytes transferred, bytes returned, or a code | the completing driver |
| `RequestorMode` | `KPROCESSOR_MODE` | `UserMode` (1) or `KernelMode` (0) of the original caller | I/O manager; read-only |
| `PendingReturned` | `BOOLEAN` | a lower layer returned `STATUS_PENDING` | I/O manager and completion propagation; you mirror it with `IoMarkIrpPending` |
| `StackCount` | `CHAR` | number of stack locations in this allocation | I/O manager; read-only |
| `CurrentLocation` | `CHAR` | index of the record now in use | I/O manager, `IoCallDriver`, `IoCompleteRequest`; never assign |
| `Cancel` | `BOOLEAN` | `TRUE` once `IoCancelIrp` has started | I/O manager |
| `CancelIrql` | `KIRQL` | IRQL at which the cancel routine was entered | the cancel path |
| `CancelRoutine` | `PDRIVER_CANCEL` | set by `IoSetCancelRoutine` | the driver that pends the IRP |
| `UserBuffer` | `PVOID` | raw user virtual address of the data buffer or IOCTL output | I/O manager sets; you must probe before use |
| `Tail.Overlay.CurrentStackLocation` | `PIO_STACK_LOCATION` | pointer into the array | I/O manager; use the macros only |
| `Tail.Overlay.DriverContext[4]` | `PVOID[4]` | four scratch slots for the driver that owns the IRP | you, while you own it |
| `Tail.Overlay.Thread` | `PETHREAD` | thread that issued the request (`NULL` if none) | I/O manager; read-only |
| `Tail.Overlay.ListEntry` | `LIST_ENTRY` | linkage when you queue the IRP yourself | you, if you queue |
| `Tail.Overlay.OriginalFileObject` | `PFILE_OBJECT` | file object that started the request | I/O manager; read-only |
| `Tail.Apc`, `Tail.CompletionKey` | | special-kernel-APC completion and IOCTL completion key | reserved for the I/O manager |
| `Tail.Overlay.AuxiliaryBuffer` | `PCHAR` | internal bookkeeping for buffered transfers | reserved |

### `IO_STACK_LOCATION` fields

| Field | Type | Meaning | Who writes it |
|---|---|---|---|
| `MajorFunction` | `UCHAR` | request type, an index into `DriverObject->MajorFunction[]` | issuer; read-only for you |
| `MinorFunction` | `UCHAR` | sub-code for PnP, power, and WMI requests | issuer |
| `Flags` | `UCHAR` | reserved; visible in `!irp` output | system |
| `Control` | `UCHAR` | `SL_PENDING_RETURNED` 0x01, `SL_ERROR_RETURNED` 0x02, `SL_INVOKE_ON_CANCEL` 0x20, `SL_INVOKE_ON_SUCCESS` 0x40, `SL_INVOKE_ON_ERROR` 0x80 | `IoMarkIrpPending` and `IoSetCompletionRoutine`; zeroed by the copy macro |
| `Parameters` | union | per-major-function arguments | issuer; you may prepare the next record |
| `DeviceObject` | `PDEVICE_OBJECT` | device whose driver serves this record | filled by `IoCallDriver` |
| `FileObject` | `PFILE_OBJECT` | file object for this request; `NULL` for PnP and power | issuer |
| `CompletionRoutine` | `PIO_COMPLETION_ROUTINE` | routine installed by the layer above | that layer, via `IoSetCompletionRoutine` |
| `Context` | `PVOID` | argument handed to that routine | same |

`Parameters` members you will use, with the major functions they belong to:

| Member | Fields | Major function |
|---|---|---|
| `Parameters.Create` | `SecurityContext`, `Options`, `FileAttributes`, `ShareAccess`, `EaLength` | `IRP_MJ_CREATE` |
| `Parameters.Read` | `Length` (`ULONG`), `Key` (`ULONG`), `ByteOffset` (`LARGE_INTEGER`) | `IRP_MJ_READ` |
| `Parameters.Write` | `Length`, `Key`, `ByteOffset` | `IRP_MJ_WRITE` |
| `Parameters.DeviceIoControl` | `OutputBufferLength`, `InputBufferLength`, `IoControlCode`, `Type3InputBuffer` | `IRP_MJ_DEVICE_CONTROL`, `IRP_MJ_INTERNAL_DEVICE_CONTROL` |
| `Parameters.QueryFile` / `SetFile` / `QueryVolume` | `Length`, `FileInformationClass` / `FsInformationClass`, `Type3InputBuffer` | file-system requests |
| `Parameters.Pnp` | `MinorFunction` (duplicated) | `IRP_MJ_PNP` |
| `Parameters.Power` | `SystemContext`, `Type`, `State`, `ShutdownType` | `IRP_MJ_POWER` |
| `Parameters.Others.Argument1..4` | four pointer-sized slots | anything not covered above; also what `!irp` prints as `Args:` |

### Functions

| Function | Purpose | IRQL | Notes |
|---|---|---|---|
| `IoGetCurrentIrpStackLocation(Irp)` | your record | any | macro; returns `Irp->Tail.Overlay.CurrentStackLocation` |
| `IoGetNextIrpStackLocation(Irp)` | the record below | any | macro; may be one past the end if `CurrentLocation == 0` |
| `IoCallDriver(DevObj, Irp)` | hand the IRP down | any | decrements `CurrentLocation`, sets `DeviceObject`, returns the lower dispatch's status |
| `PoCallDriver(DevObj, Irp)` | hand a power IRP down | any | required for `IRP_MJ_POWER` instead of `IoCallDriver` |
| `IoSkipCurrentIrpStackLocation(Irp)` | pass the same record down | any | macro; incompatible with a completion routine |
| `IoCopyCurrentIrpStackLocationToNext(Irp)` | duplicate the record below | any | macro; zeroes the next `Control` |
| `IoSetCompletionRoutine(Irp, Routine, Context, Ok, Err, Cancel)` | register a callback | any | `VOID`; writes into the **next** record |
| `IoSetCompletionRoutineEx(DriverObject, Irp, ...)` | same plus unload protection | any | returns `NTSTATUS`; check it |
| `IoMarkIrpPending(Irp)` | set `SL_PENDING_RETURNED` in your record | any | call before the request can complete |
| `IoCompleteRequest(Irp, Increment)` | finish or resume completion | `<= DISPATCH_LEVEL` | IRP invalid on return |
| `IoAllocateIrp(StackSize, ChargeQuota)` | build your own IRP | any | `StackSize` of the target device; you free it |
| `IoFreeIrp(Irp)` | release an IRP you allocated | any | only after you own it, never after `IoCompleteRequest` returned it to the manager |
| `IoReuseIrp(Irp, Status)` | re-arm a recycled IRP | any | resets header; stack locations are yours to refill |
| `IoForwardIrpSynchronously(DevObj, Irp)` | send down and wait | `PASSIVE_LEVEL` | returns when the lower stack is done; check `IoStatus.Status` |
| `IoGetRequestorProcess(Irp)` | process that issued it | any | may be `NULL` for kernel-originated requests |
| `IoGetCurrentProcess()` | current process | any | different from the requestor after a hand-off |

## Step-by-step

1. **Decide the entry point.** Point the major function index in `DriverObject->MajorFunction[]` at a
   routine with the `DRIVER_DISPATCH` shape `NTSTATUS Routine(PDEVICE_OBJECT DeviceObject, PIRP Irp)`.
   Every unused index must still hold a real routine (a shared one completing with
   `STATUS_INVALID_DEVICE_REQUEST`): a `NULL` slot bugchecks the first time that request arrives.
2. **Fetch your record once** into a local (`PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp)`)
   and take every parameter from it; check `sp->MajorFunction` when one routine serves several indices.
3. **Validate before acting.** Check lengths, control codes, buffer presence, and `RequestorMode`;
   complete with `STATUS_INVALID_PARAMETER`, `STATUS_INVALID_DEVICE_REQUEST`, or
   `STATUS_BUFFER_TOO_SMALL`. Never dereference a user pointer here (`wk-irp-buffering`).
4. **Check the IRQL you are actually at** if the routine can be reached from a paging path, a DPC, or a
   lower filter: anything that blocks, waits, or touches paged memory needs `PASSIVE_LEVEL`, so queue
   that work instead of calling it.
5. **Choose one of three behaviours** so that exactly one of them happens on every path:
   - complete now: set `IoStatus`, call `IoCompleteRequest(Irp, IO_NO_INCREMENT)`, return the captured status;
   - forward: `IoCallDriver(LowerDevice, Irp)` and return its result;
   - pend: `IoMarkIrpPending(Irp)`, queue the IRP, return `STATUS_PENDING`.
6. **Forwarding with interest**: `IoCopyCurrentIrpStackLocationToNext(Irp)`, then
   `IoSetCompletionRoutine(Irp, YourComplete, YourContext, TRUE, TRUE, TRUE)`, then
   `return IoCallDriver(LowerDevice, Irp);`. Do not touch `Irp` after that unless your completion
   routine returned `STATUS_MORE_PROCESSING_REQUIRED`.
7. **Forwarding without interest**: `IoSkipCurrentIrpStackLocation(Irp); return IoCallDriver(...);`.
   `IRP_MJ_POWER` uses `PoCallDriver` with the same skip.
8. **In the completion routine**: read `Irp->IoStatus` first, mirror pending with
   `if (Irp->PendingReturned) IoMarkIrpPending(Irp);`, release what the dispatch acquired, return
   `STATUS_SUCCESS` unless you are deliberately stopping the walk.
9. **Unload**: prove no IRP is in flight before `IoDeleteDevice` and before `DriverUnload` returns
   (`0xCE` otherwise). A filter that only forwards still has IRPs outstanding whenever a lower driver
   returned `STATUS_PENDING`.
10. **Verify in the debugger**: `!devstack` for the stack shape, `!irp` per request, `!irpfind` for
    anything still referenced.

## Complete example

`IrpTap` is a pass-through filter that sits above an existing device, logs every major function, and
forwards everything. It installs a completion routine on the data path (so completion status is
logged) and skips the stack location on PnP and power (where the result is not interesting).
Replace `IRPTAP_TARGET` with the device you want to observe; the null device is present on every
Windows system and accepts create and write, which makes the example runnable without hardware.

```c
/* IrpTap.c - pass-through filter: log every major function, forward all of them. */
#include <ntddk.h>

#define IRPTAP_TAG      'paTI'                  /* 'ITap' in memory order */
#define IRPTAP_TARGET   L"\\Device\\Null"
#define IRPTAP_DEVICE   L"\\Device\\IrpTap"
#define IRPTAP_LINK     L"\\??\\IrpTap"

typedef struct _IRPTAP_EXTENSION {
    PDEVICE_OBJECT LowerDevice;
    volatile LONG  Forwarded;
    volatile LONG  Completed;
} IRPTAP_EXTENSION, *PIRPTAP_EXTENSION;

static PDEVICE_OBJECT g_TapDevice;
static LONG           g_SymLinkLive;

static VOID IrpTapDropSymLink(VOID)
{
    UNICODE_STRING link = RTL_CONSTANT_STRING(IRPTAP_LINK);

    if (InterlockedExchange(&g_SymLinkLive, 0) != 0) {
        (VOID)IoDeleteSymbolicLink(&link);
    }
}

/* One line per request; length means different things per major function. */
static ULONG IrpTapRequestLength(_In_ PIO_STACK_LOCATION IrpSp)
{
    switch (IrpSp->MajorFunction) {
    case IRP_MJ_READ:
        return IrpSp->Parameters.Read.Length;
    case IRP_MJ_WRITE:
        return IrpSp->Parameters.Write.Length;
    case IRP_MJ_DEVICE_CONTROL:
    case IRP_MJ_INTERNAL_DEVICE_CONTROL:
        return IrpSp->Parameters.DeviceIoControl.OutputBufferLength;
    default:
        return 0;
    }
}

static IO_COMPLETION_ROUTINE IrpTapComplete;

static NTSTATUS IrpTapComplete(_In_ PDEVICE_OBJECT DeviceObject,
                               _In_ PIRP Irp,
                               _In_ PVOID Context)
{
    PIRPTAP_EXTENSION ext = (PIRPTAP_EXTENSION)Context;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);

    UNREFERENCED_PARAMETER(DeviceObject);

    /* A lower layer pended: the I/O manager must see the same pending state
       on this layer's record, or the caller's IO_STATUS_BLOCK is freed early. */
    if (Irp->PendingReturned) {
        IoMarkIrpPending(Irp);
    }

    InterlockedIncrement(&ext->Completed);
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "IrpTap: done   mj=0x%02X status=0x%08X info=%I64u pending=%u\n",
               irpSp->MajorFunction,
               (ULONG)Irp->IoStatus.Status,
               (unsigned long long)Irp->IoStatus.Information,
               (ULONG)Irp->PendingReturned);

    return STATUS_SUCCESS;              /* keep the completion walk going */
}

/* Data path: copy the record down so a completion routine can be installed. */
static NTSTATUS IrpTapForward(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIRPTAP_EXTENSION ext = (PIRPTAP_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);

    InterlockedIncrement(&ext->Forwarded);
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_TRACE_LEVEL,
               "IrpTap: enter  mj=0x%02X mn=0x%02X len=0x%X mode=%u\n",
               irpSp->MajorFunction, irpSp->MinorFunction,
               IrpTapRequestLength(irpSp), (ULONG)Irp->RequestorMode);

    /* Copy first: the copy zeroes the next record's Control byte, which would
       erase the SL_INVOKE_ON_* bits if the completion routine ran first. */
    IoCopyCurrentIrpStackLocationToNext(Irp);
    IoSetCompletionRoutine(Irp, IrpTapComplete, ext, TRUE, TRUE, TRUE);

    return IoCallDriver(ext->LowerDevice, Irp);
}

/* Power path: skip our record, and hand the IRP to PoCallDriver, not IoCallDriver. */
static NTSTATUS IrpTapPower(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIRPTAP_EXTENSION ext = (PIRPTAP_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);

    UNREFERENCED_PARAMETER(DeviceObject);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_TRACE_LEVEL,
               "IrpTap: power  mn=0x%02X type=%u state=0x%X\n",
               irpSp->MinorFunction,
               (ULONG)irpSp->Parameters.Power.Type,
               (ULONG)irpSp->Parameters.Power.State.SystemState);

    IoSkipCurrentIrpStackLocation(Irp);
    return PoCallDriver(ext->LowerDevice, Irp);
}

/* PnP path: passthrough, except remove, where the filter must detach and die. */
static NTSTATUS IrpTapPnp(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIRPTAP_EXTENSION ext = (PIRPTAP_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status;

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_TRACE_LEVEL,
               "IrpTap: pnp    mn=0x%02X\n", irpSp->MinorFunction);

    if (irpSp->MinorFunction != IRP_MN_REMOVE_DEVICE) {
        IoSkipCurrentIrpStackLocation(Irp);
        return IoCallDriver(ext->LowerDevice, Irp);
    }

    /* Remove the name first: no new handle can be opened while the device dies. */
    IrpTapDropSymLink();

    IoSkipCurrentIrpStackLocation(Irp);
    status = IoCallDriver(ext->LowerDevice, Irp);

    if (g_TapDevice == DeviceObject) {
        g_TapDevice = NULL;
    }
    IoDetachDevice(ext->LowerDevice);
    IoDeleteDevice(DeviceObject);       /* frees the extension too */
    return status;
}

static VOID IrpTapUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT device = g_TapDevice;

    UNREFERENCED_PARAMETER(DriverObject);

    IrpTapDropSymLink();

    if (device != NULL) {
        PIRPTAP_EXTENSION ext = (PIRPTAP_EXTENSION)device->DeviceExtension;

        g_TapDevice = NULL;
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "IrpTap: unload forwarded=%ld completed=%ld\n",
                   ext->Forwarded, ext->Completed);
        IoDetachDevice(ext->LowerDevice);
        IoDeleteDevice(device);
    }
}

static NTSTATUS IrpTapAttach(_In_ PDRIVER_OBJECT DriverObject)
{
    NTSTATUS status;
    PDEVICE_OBJECT tapDevice = NULL;
    PDEVICE_OBJECT targetDevice = NULL;
    PFILE_OBJECT targetFile = NULL;
    PIRPTAP_EXTENSION ext;
    UNICODE_STRING targetName = RTL_CONSTANT_STRING(IRPTAP_TARGET);
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(IRPTAP_DEVICE);
    UNICODE_STRING linkName = RTL_CONSTANT_STRING(IRPTAP_LINK);

    status = IoGetDeviceObjectPointer(&targetName, FILE_ALL_ACCESS,
                                      &targetFile, &targetDevice);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "IrpTap: cannot open %wZ (0x%08X)\n", &targetName, status);
        return status;
    }

    status = IoCreateDevice(DriverObject, sizeof(IRPTAP_EXTENSION), &deviceName,
                            targetDevice->DeviceType, 0, FALSE, &tapDevice);
    if (!NT_SUCCESS(status)) {
        goto Cleanup;
    }

    ext = (PIRPTAP_EXTENSION)tapDevice->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));

    /* Inherit the buffering contract: the I/O manager prepared buffers using
       the top device's flags, so a mismatch corrupts every read and write. */
    tapDevice->Flags |= (targetDevice->Flags & (DO_BUFFERED_IO | DO_DIRECT_IO));
    tapDevice->AlignmentRequirement = targetDevice->AlignmentRequirement;

    ext->LowerDevice = IoAttachDeviceToDeviceStack(tapDevice, targetDevice);
    if (ext->LowerDevice == NULL) {
        status = STATUS_NO_SUCH_DEVICE;
        goto Cleanup;
    }

    /* One more layer than everything below needs. Too small corrupts the pool. */
    tapDevice->StackSize = ext->LowerDevice->StackSize + 1;

    status = IoCreateSymbolicLink(&linkName, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDetachDevice(ext->LowerDevice);
        goto Cleanup;
    }
    InterlockedExchange(&g_SymLinkLive, 1);

    tapDevice->Flags &= ~DO_DEVICE_INITIALIZING;
    g_TapDevice = tapDevice;
    tapDevice = NULL;                   /* ownership moved to the global */
    status = STATUS_SUCCESS;

Cleanup:
    if (targetFile != NULL) {
        ObDereferenceObject(targetFile); /* attachment keeps the device alive */
    }
    if (tapDevice != NULL) {
        IoDeleteDevice(tapDevice);
    }
    return status;
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    ULONG index;

    UNREFERENCED_PARAMETER(RegistryPath);

    /* Every slot must be callable; a NULL slot bugchecks on first use. */
    for (index = 0; index <= IRP_MJ_MAXIMUM_FUNCTION; index++) {
        DriverObject->MajorFunction[index] = IrpTapForward;
    }
    DriverObject->MajorFunction[IRP_MJ_PNP] = IrpTapPnp;
    DriverObject->MajorFunction[IRP_MJ_POWER] = IrpTapPower;
    DriverObject->DriverUnload = IrpTapUnload;

    return IrpTapAttach(DriverObject);
}
```

The dispatch routine is declared through the array, so no prototype is needed for it; the completion
routine is forward-declared because `IrpTapComplete` and the routines that install it are mutually
visible only in that order. `IrpTapComplete` has the `IO_COMPLETION_ROUTINE` shape and
`IrpTapPnp`/`IrpTapPower`/`IrpTapForward` have the `DRIVER_DISPATCH` shape.

A minimal client that exercises the filter (`cl /W4 tapclient.c`):

```c
/* tapclient.c */
#include <windows.h>
#include <stdio.h>

int main(void)
{
    HANDLE device = CreateFileW(L"\\\\.\\IrpTap", GENERIC_READ | GENERIC_WRITE,
                                FILE_SHARE_READ | FILE_SHARE_WRITE, NULL,
                                OPEN_EXISTING, 0, NULL);
    DWORD bytes = 0;

    if (device == INVALID_HANDLE_VALUE) {
        printf("CreateFile failed, error=%lu\n", GetLastError());
        return 1;
    }
    /* Unsupported control code: the filter logs the failure status, unchanged. */
    if (!DeviceIoControl(device, 0x00222000, NULL, 0, NULL, 0, &bytes, NULL)) {
        printf("ioctl rejected as expected, error=%lu\n", GetLastError());
    }
    CloseHandle(device);
    return 0;
}
```

## Variations and advanced use

- **Recycling and driver-originated IRPs.** `IoAllocateIrp(Target->StackSize, FALSE)` returns an IRP
  with `CurrentLocation == StackCount`; fill `IoGetNextIrpStackLocation(Irp)`, install a completion
  routine, `IoCallDriver`, and free it yourself with `IoFreeIrp` once you own it again (which is exactly
  what `STATUS_MORE_PROCESSING_REQUIRED` leaves you holding). `IoBuildDeviceIoControlRequest` and
  `IoBuildSynchronousFsdRequest` fill the parameters for you; `IoReuseIrp(Irp, STATUS_SUCCESS)` resets a
  driver-allocated IRP for another trip but does **not** reset your stack locations.
- **Completion context without allocation.** `Irp->Tail.Overlay.DriverContext[0..3]` is four
  pointer-sized slots reserved for the driver that owns the IRP. Storing a small context there avoids a
  pool allocation per request, but the slots belong to whoever holds the IRP: do not stash a pointer and
  then forward the IRP to a layer that may reuse them.
- **Status rewriting.** A completion routine may replace a lower driver's status (mapping
  `STATUS_INVALID_DEVICE_REQUEST` to `STATUS_NOT_SUPPORTED`, for example). Do it before returning, keep
  `IoStatus.Information` consistent, and never turn a success into a failure after bytes moved.
- **Performance.** The copy macro costs one 0x30-byte `RtlCopyMemory`; the skip macro costs one
  decrement. A filter that only counts requests should skip everywhere and do its accounting in the
  dispatch routine. A filter that needs the result must copy, and then its completion routine runs at
  `DISPATCH_LEVEL` often enough that the work there must be non-paged and short.

## Failure modes and diagnosis

| Symptom | Likely cause | First command / fix |
|---|---|---|
| `0x8E KERNEL_MODE_EXCEPTION_NOT_HANDLED`, parameter 1 `0xC0000005` | dispatch or completion dereferenced a user VA (in the wrong process, or at raised IRQL) | `!analyze -v`, `kv`; move the access into `wk-irp-buffering` territory and probe |
| `0xA IRQL_NOT_LESS_OR_EQUAL` | a `PASSIVE_LEVEL`-only call (`Zw*`, `KeWaitForSingleObject`, `IoCreateSymbolicLink`) inside a path that ran at `DISPATCH_LEVEL` | `!irql`, then find who sent the IRP: `!irp` shows the originating layer |
| `0xD1 DRIVER_IRQL_NOT_LESS_OR_EQUAL` | touching paged pool or a paged code section at `DISPATCH_LEVEL` | `!analyze -v` gives the faulting address; check the extension's pool type and `PAGED_CODE()` |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA` | completion routine used a device extension or context freed by unload or by `IRP_MN_REMOVE_DEVICE` | `!irpfind` for the IRP, `!devobj` on the device in the stack |
| `0xCE DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | unload returned while an IRP was still queued or pended | `!irpfind` before `sc stop`; drain or refuse to unload |
| `0x9F DRIVER_POWER_STATE_FAILURE` | power IRP completed without being forwarded, or forwarded with `IoCallDriver` instead of `PoCallDriver` | check every `IRP_MJ_POWER` path |
| `0xC0000010 STATUS_INVALID_DEVICE_REQUEST` returned to the client | the major function index still points at the default stub, or no dispatch was installed | inspect `!drvobj <driver> 2` and the `MajorFunction` table |
| `0xC00000BB STATUS_NOT_SUPPORTED` | the request was understood but deliberately refused (wrong minor function, unsupported IOCTL) | expected; confirm the client checks `GetLastError` |
| `0xC0000206 STATUS_INVALID_BUFFER_SIZE` | zero-length or oversized transfer rejected by the driver | compare `Parameters.Read.Length` with what the client sent |
| Client never returns; no bugcheck | IRP returned `STATUS_PENDING` without `IoMarkIrpPending`, or never completed at all | `!irp <addr> 1` shows `pending` and the owning thread; check every exit path calls `IoCompleteRequest` once |
| Status changes between layers | a completion routine rewrote `IoStatus.Status` without adjusting `Information` | log at every layer (`!irp` per stack index) |
| Pool corruption (`0xC2`, `0x50`) after adding a filter | `StackSize` was not incremented, so `IoCallDriver` wrote past the array | `dt nt!_DEVICE_OBJECT <addr> StackSize` on every device in `!devstack` |

### Reading the debugger

```
kd> !devstack ffff9e0b1a2c3d40
  !DevObj           !DrvObj            !DevExt           ObjectName
  ffff9e0b1a2c3d40  \Driver\IrpTap     ffff9e0b1a2c3e50  \Device\IrpTap
> ffff9e0b1c4a5b60  \Driver\Null       ffff9e0b1c4a5c70  \Device\Null
```

(Addresses and column spacing vary; the `>` marks the device object named in the command.) Read it
bottom-up for the physical order of the stack; confirm `!DevExt` points at a live block, not `Free`.

```
kd> dt nt!_IRP ffff9e0b1d2e3f60 MdlAddress Flags AssociatedIrp.SystemBuffer IoStatus RequestorMode StackCount CurrentLocation
   +0x008 MdlAddress            : (null)
   +0x010 Flags                 : 0x7
   +0x018 AssociatedIrp.SystemBuffer : 0xffff9e0b1e0a2010
   +0x030 IoStatus              : _IO_STATUS_BLOCK
   +0x048 RequestorMode         : 1 (UserMode)
   +0x05a StackCount            : 6
   +0x05b CurrentLocation       : 2
```

`Flags == 0x7` is `IRP_BUFFERED_IO | IRP_DEALLOCATE_BUFFER | IRP_INPUT_OPERATION` (bits 0x01, 0x02,
0x04): a buffered read that will be copied back to the caller, so the buffering decision matched the
request direction. `CurrentLocation` tells you which record `!irp` will mark with `>`.

```
kd> !irp ffff9e0b1d2e3f60
Irp is active with 6 stacks 2 is current (= 0xffff9e0b1d2e40a0)
 No Mdl: System buffer=ffff9e0b1e0a2010: Thread ffff9e0b19c8a080:  Irp stack trace.
     cmd  flg cl Device   File     Completion-Context
 [ 0, 0]   0  0 00000000 00000000 00000000-00000000
             Args: 00000000 00000000 00000000 00000000
 [ 0, 0]   0  0 ffff9e0b1a2c3d40 ffff9e0b1c2d1e30 00000000-00000000
        \Driver\IrpTap
> [IRP_MJ_READ(3), N/A(0)]
            0  1 ffff9e0b1c4a5b60 ffff9e0b1c2d1e30 fffff8051a2b3c40-ffff9e0b1a2c3e50 Success Error Cancel pending
        \Driver\Null
             Args: 00000080 00000000 00000000 00000000
```

What to look for: the `>` line is the current record; `flg` non-zero with `pending` in the flags column
means `SL_PENDING_RETURNED`; a non-zero `Completion-Context` on the current record is the routine
installed by the layer **above** it, and it runs when the request completes below that point; `Args:`
for a read shows `Length` in the first slot. A `!irp` line saying
`Irp is complete (CurrentLocation n > StackCount m)` is a completed IRP still in a queue — evidence of
a double completion or a lost dequeue. `!irpfind` scans non-paged pool for every IRP allocation still
alive, with the owning thread; run it after a hang.

## Safety, IRQL, and cleanup checklist

- [ ] Every `MajorFunction[]` index 0 through `IRP_MJ_MAXIMUM_FUNCTION` holds a real routine.
- [ ] The dispatch routine reads parameters only from `IoGetCurrentIrpStackLocation`, never from
      `Tail.Overlay` arithmetic.
- [ ] `IoCompleteRequest` is called exactly once per IRP on every path, and the status returned by the
      dispatch routine was captured **before** that call.
- [ ] `IoMarkIrpPending` precedes any code that could complete or queue the IRP, and the routine then
      returns `STATUS_PENDING`.
- [ ] A completion routine that sees `Irp->PendingReturned` calls `IoMarkIrpPending`.
- [ ] Skip is used only where no completion routine is installed; copy is followed by
      `IoSetCompletionRoutine` **after** the copy.
- [ ] `StackSize == lower->StackSize + 1` for every filter, set before the device is reachable.
- [ ] `IRP_MJ_POWER` goes down through `PoCallDriver`; `IRP_MJ_PNP` handles `IRP_MN_REMOVE_DEVICE` by
      detaching and deleting.
- [ ] No blocking call, no file I/O, no registry access, and no paged code reachable from a routine that
      may run at `DISPATCH_LEVEL`; a completion routine uses only non-paged memory and never waits.
- [ ] Unload proves no IRP is in flight (`0xCE`), and no completion routine can run after the image is
      unmapped.
- [ ] Every IRP the driver allocates is freed by the driver (`IoFreeIrp`), or completed and returned to
      the I/O manager exactly once.
- [ ] A reference taken in the dispatch (file object, device object, process) is released in the
      completion routine, not at the end of the dispatch.

## Verification

1. **Build** for the target architecture with warnings as errors — a missing prototype for a dispatch
   or completion routine is a C4028/C2220 that must not be suppressed:

   ```
   msbuild IrpTap.vcxproj /p:Configuration=Debug /p:Platform=x64 /p:TargetVersion=Windows10
   ```

2. **Load and observe.** Start DebugView with `Capture Kernel` enabled, then
   `sc create IrpTap type= kernel binPath= C:\drivers\IrpTap.sys` and `sc start IrpTap`. The client
   must produce an `IrpTap: enter  mj=0x00 ...` line followed by
   `IrpTap: done   mj=0x00 status=0x00000000` for the create, then one pair per operation. A missing
   `done` line means the completion routine never ran; a mismatched major code means the stack
   location was prepared wrongly.

3. **Inspect the stack** while a handle is open. `StackSize` must be one greater than the device below,
   and `Flags` must carry the same buffering bits as the target:

   ```
   kd> !devstack <IrpTap device object>
   kd> dt nt!_DEVICE_OBJECT <IrpTap device object> StackSize Flags
   ```

4. **Break on a request.** The IRP is the second x64 argument, so it is in `rdx`; read the record
   through the field, never through a hand-computed offset:

   ```
   kd> bp IrpTap!IrpTapForward
   kd> g
   kd> !irp @rdx
   kd> dt nt!_IO_STACK_LOCATION poi(nt!_IRP @rdx Tail.Overlay.CurrentStackLocation) Parameters.Read
   ```

   The last command prints `Length`, `Key`, `ByteOffset`; compare `Length` with the client's request.

5. **Enable Driver Verifier** and repeat the run. Its `Irp` checks catch reuse of a completed IRP,
   completion at raised IRQL, and `STATUS_PENDING` returned without the pending bit:

   ```
   verifier /standard /driver IrpTap.sys
   ```

6. **Unload cleanly.** `sc stop IrpTap` must return without a bugcheck, and `!irpfind` afterwards must
   show no IRP whose stack contains `\Driver\IrpTap`; a `0xCE` means one was still in flight.
