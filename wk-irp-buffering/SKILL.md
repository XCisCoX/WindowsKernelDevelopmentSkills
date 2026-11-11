---
name: wk-irp-buffering
description: Use when a driver must read or write caller data safely: choosing DO_BUFFERED_IO, DO_DIRECT_IO, or neither, decoding METHOD_BUFFERED, METHOD_IN_DIRECT, METHOD_OUT_DIRECT, and METHOD_NEITHER, using SystemBuffer, MdlAddress, MmGetSystemAddressForMdlSafe, MmProbeAndLockPages, UserBuffer and RequestorMode, probing with SEH, and setting IoStatus.Information for full and partial transfers.
---

# Moving bytes through an IRP: buffered, direct, and neither

A driver never receives a pointer it can trust. The buffer described by an IRP is one of three things:
a system-space copy the I/O manager made (`AssociatedIrp.SystemBuffer`), a locked user buffer described
by an MDL (`MdlAddress`) that you map on demand, or a raw user virtual address (`UserBuffer`,
`Type3InputBuffer`) that is valid only in one process, at one IRQL, for one instant.

The rule that prevents most disasters: **never dereference a caller-supplied address unless
`Irp->RequestorMode` says user mode, the current thread is the requesting process, and the IRQL is below
`DISPATCH_LEVEL` — and even then probe the range and wrap the copy in `__try`.** Buffered and direct I/O
exist so that most handlers never need that check at all.

## When to use

- A read or write handler returns garbage, zeros, or bugchecks when it touches its buffer.
- You are choosing device flags, or explaining why an IOCTL code's `Method` bits ignore those flags.
- A `0xD1`, `0x50`, `0xA`, or `0x8E` points at buffer access in a dispatch or completion routine.
- A client sees a truncated transfer, or `ReadFile` returns success with fewer bytes than requested.
- You must build and lock your own MDL for a buffer that did not arrive in an IRP.
- You are reviewing a `METHOD_NEITHER` handler for double fetch, unchecked lengths, or a user pointer
  treated as a kernel pointer.
- You need the exact contents of `IoStatus.Information` for a given major function.

Do not use for IRP ownership, stack locations, or completion rules (`wk-irp-basics`), for IOCTL protocol
design (`wk-ioctl-design`), or for the user-mode side (`wk-driver-client-protocol`).

## Prerequisites and related skills

- **Headers:** `<ntddk.h>`. `CTL_CODE` and `METHOD_*` come from the WDK headers; a shared protocol header
  must be written so the client includes `<windows.h>` first and the driver includes `<ntddk.h>` first.
- **IRQL:** the system buffer is non-paged, so buffered access is legal at any IRQL. A mapping obtained
  with `MmGetSystemAddressForMdlSafe` is legal at any IRQL. **A raw user pointer is legal only at
  `PASSIVE_LEVEL` or `APC_LEVEL`, in the requesting process.** `MmProbeAndLockPages` with `UserMode`
  requires `IRQL <= APC_LEVEL`; with `KernelMode` it may run at `DISPATCH_LEVEL`.
- **WDK:** `MmGetSystemAddressForMdlSafe` is the only mapping call to use; the older
  `MmGetSystemAddressForMdl` bugchecks when it fails and must not appear in new code.
- Siblings: `wk-irp-basics` (who owns the IRP and when it completes), `wk-kernel-exceptions`
  (`ProbeForRead`/`ProbeForWrite`, `__try`/`__except`, C2712), `wk-ioctl-design`, `wk-irql`.

## Mental model

Three transfer mechanisms, and the I/O manager does something different in each:

```
BUFFERED (DO_BUFFERED_IO / METHOD_BUFFERED)
  user VA --copy in--> SystemBuffer (non-paged pool) --driver--> --copy out--> user VA
  copied by the I/O manager; readable at any IRQL, from any process. Cost: one or two full copies.

DIRECT (DO_DIRECT_IO / METHOD_IN_DIRECT / METHOD_OUT_DIRECT)
  user VA --probe+lock--> MDL (locked frames) --MmGetSystemAddressForMdlSafe--> system VA
  copied by nobody; both addresses are the same physical memory. Cost: lock and PTE work.

NEITHER (no device flag / METHOD_NEITHER)
  raw user VA (UserBuffer, Type3InputBuffer) --you--> probe, __try, bounds check, copy
  copied by you; valid only in the requesting process, below DISPATCH_LEVEL.
```

### The flag and the method are separate decisions

This asymmetry causes more confusion than any other part of the topic:

| What you are handling | Who chooses the mechanism |
|---|---|
| `IRP_MJ_READ`, `IRP_MJ_WRITE` | the **device flags**: `DO_BUFFERED_IO`, `DO_DIRECT_IO`, or neither |
| `IRP_MJ_DEVICE_CONTROL`, `IRP_MJ_INTERNAL_DEVICE_CONTROL` | the **`Method` bits of each `CTL_CODE`**, whatever the device flags say |
| `IRP_MJ_QUERY_INFORMATION` and relatives | the method chosen by that information class |

One device can be `DO_DIRECT_IO` for reads and still receive `METHOD_BUFFERED` IOCTLs whose payload
lands in `SystemBuffer`. Read the flags for read/write, decode the control code for an IOCTL, and never
let one imply the other.

### Who prepares what, and when it goes away

| Item | Created by | Freed by | Valid until |
|---|---|---|---|
| `AssociatedIrp.SystemBuffer` | I/O manager | I/O manager, after the copy-back | end of `IoCompleteRequest` |
| `Irp->MdlAddress` (direct-I/O read/write) | I/O manager | I/O manager | end of `IoCompleteRequest` |
| System mapping of that MDL | `MmGetSystemAddressForMdlSafe`, cached in the MDL | I/O manager (it holds `MDL_MAPPED_TO_SYSTEM_VA`) | end of `IoCompleteRequest` |
| Your MDL from `IoAllocateMdl` + `MmProbeAndLockPages` | you | you: `MmUnlockPages`, then `IoFreeMdl` | until you free it |
| An MDL you attached with `IoAllocateMdl(..., Irp)` | you | the I/O manager, with the IRP | end of `IoCompleteRequest` |
| A mapping you made with `MmMapLockedPagesSpecifyCache` | you | **you**: `MmUnmapLockedPages` | until you unmap |

The mapping is deliberately lazy: the I/O manager locks the pages but does not map them, because a
driver that rejects the request on a length check never needs the mapping. **Call
`MmGetSystemAddressForMdlSafe` only after validation passes** and treat `NULL` as
`STATUS_INSUFFICIENT_RESOURCES`.

### Direction: what each method puts in each field

| Method | Input buffer | Output buffer |
|---|---|---|
| `METHOD_BUFFERED` | `SystemBuffer`, `InputBufferLength` bytes | `SystemBuffer` (same allocation), `OutputBufferLength` bytes |
| `METHOD_IN_DIRECT` | `SystemBuffer` | `MdlAddress`, readable |
| `METHOD_OUT_DIRECT` | `SystemBuffer` | `MdlAddress`, writable |
| `METHOD_NEITHER` | `Type3InputBuffer`, raw user VA | `UserBuffer`, raw user VA |

For `METHOD_BUFFERED` the allocation is `max(InputBufferLength, OutputBufferLength)` bytes and it is
**not zeroed**. A bidirectional request keeps its input at the start of the same memory that the output
overwrites, so copy every input field you need into locals before writing the first output byte.

### Reading the fields correctly

- `Parameters.Read.Length` / `Parameters.Write.Length` is the byte count; it must equal
  `MmGetMdlByteCount(Irp->MdlAddress)` for a direct-I/O request. If they disagree, fail rather than
  trust either.
- `Parameters.DeviceIoControl.InputBufferLength` / `OutputBufferLength` are the caller's declared sizes.
  For `METHOD_BUFFERED`, a non-zero length with `SystemBuffer == NULL` means the allocation failed.
- `Irp->UserBuffer` is the raw pointer for `METHOD_NEITHER` output **and** for a neither-I/O read or
  write. `Parameters.DeviceIoControl.Type3InputBuffer` is the raw pointer for `METHOD_NEITHER` input.
- `Irp->RequestorMode` is `UserMode` (1) for user callers and `KernelMode` (0) for requests built by
  other drivers, including all paging I/O. **`ProbeForRead`/`ProbeForWrite` reject kernel addresses**,
  so branch on `RequestorMode` before probing, and use `IoGetRequestorProcess(Irp)` or
  `IoIsSystemThread(Irp->Tail.Overlay.Thread)` to check that this thread sees the caller's address space.

### Status and Information

| Request | Meaning of `IoStatus.Information` on success |
|---|---|
| `IRP_MJ_READ` | bytes copied into the caller's buffer; `0` with `STATUS_END_OF_FILE` at end of data |
| `IRP_MJ_WRITE` | bytes consumed from the caller's buffer |
| `IRP_MJ_DEVICE_CONTROL` | bytes written to the output buffer |
| `IRP_MJ_QUERY_INFORMATION` | bytes written to the output buffer |
| `IRP_MJ_CREATE`, `IRP_MJ_CLOSE`, `IRP_MJ_CLEANUP`, `IRP_MJ_FLUSH_BUFFERS` | `0` |
| `IRP_MJ_PNP`, `IRP_MJ_POWER` | `0` |
| any request completed with a failing status | `0` |

1. **Zero `Information` on failure.** A caller that sees a failure status and a non-zero count cannot
   interpret it, and some callers use the count anyway.
2. **Never report more than the buffer holds.** For `METHOD_BUFFERED` the copy-back length comes from
   `Information`; a value above `OutputBufferLength` makes the I/O manager read past the system buffer
   and hand pooled kernel memory to the caller.
3. **A warning status is still a success.** `NT_SUCCESS(STATUS_BUFFER_OVERFLOW)` is `TRUE`. Check
   `Information` as well as the status, and treat a short transfer as part of the protocol.

Partial transfers must be described, not hidden: `STATUS_SUCCESS` with `Information < Length` means a
genuinely short transfer; `STATUS_END_OF_FILE` (`0xC0000011`) with `Information == 0` means no data is
left; `STATUS_BUFFER_OVERFLOW` (`0x80000005`) with `Information` set to the bytes actually produced
means the caller's buffer was too small for what the device has.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `MmGetSystemAddressForMdlSafe(Mdl, Priority)` | map locked frames into system space | any | returns `NULL` on failure; **check it** |
| `MmGetSystemAddressForMdl(Mdl)` | legacy mapping | any | **do not use**; bugchecks when it fails |
| `MmGetMdlByteCount(Mdl)` | bytes described by the MDL | any | compare with `Parameters.*.Length` |
| `MmGetMdlByteOffset(Mdl)` | offset of the first byte inside the first frame | any | the system VA already includes it |
| `IoAllocateMdl(Va, Length, Secondary, ChargeQuota, Irp)` | describe a buffer | any | with `Irp != NULL` the IRP owns it |
| `MmProbeAndLockPages(Mdl, AccessMode, Operation)` | make frames resident and locked | `<= APC_LEVEL` for `UserMode` | **raises** on failure; wrap in `__try` |
| `MmUnlockPages(Mdl)` | release the lock | any | only for MDLs you locked |
| `MmBuildMdlForNonPagedPool(Mdl)` | describe a pool buffer | any | no probing; those pages cannot be paged |
| `MmMapLockedPagesSpecifyCache(Mdl, Mode, Cache, Addr, BugCheck, Prio)` | map with control | any | `UserMode` maps into the **current process** |
| `MmUnmapLockedPages(BaseAddress, Mdl)` | undo the above | any | pairs one-for-one with your own mapping |
| `IoFreeMdl(Mdl)` | free an MDL you allocated | any | not if it is attached to an IRP |
| `ProbeForRead(Address, Length, Alignment)` | validate a user read range | `< DISPATCH_LEVEL` | raises; must run inside `__try` |
| `ProbeForWrite(Address, Length, Alignment)` | validate a user write range | `< DISPATCH_LEVEL` | raises; also makes the range writable |
| `IoIs32bitProcess(Irp)` | caller is a 32-bit process | any | matters for structure layout, not pointer width |
| `IoGetRequestorProcess(Irp)` | process that issued the request | any | may be `NULL` for kernel callers |
| `KeGetCurrentIrql()` | the level you are at | any | below `APC_LEVEL` is the gate for user memory |

Read and write parameters:

| Field | Type | Meaning |
|---|---|---|
| `Parameters.Read.Length` / `Write.Length` | `ULONG` | requested byte count; the authority for `Information` |
| `Parameters.Read.Key` / `Write.Key` | `ULONG` | driver-defined key passed by the caller |
| `Parameters.Read.ByteOffset` / `Write.ByteOffset` | `LARGE_INTEGER` | requested offset; `-1` in `LowPart` means "use `FileObject->CurrentByteOffset`" |
| `IrpSp->FileObject->CurrentByteOffset` | `LARGE_INTEGER` | the handle's position, advanced by the I/O manager |

Device-control parameters:

| Field | Type | Meaning |
|---|---|---|
| `Parameters.DeviceIoControl.IoControlCode` | `ULONG` | the `CTL_CODE` value; switch on it |
| `Parameters.DeviceIoControl.InputBufferLength` | `ULONG` | declared input size |
| `Parameters.DeviceIoControl.OutputBufferLength` | `ULONG` | declared output size |
| `Parameters.DeviceIoControl.Type3InputBuffer` | `PVOID` | raw input pointer, `METHOD_NEITHER` only |
| `Irp->UserBuffer` | `PVOID` | raw output pointer, `METHOD_NEITHER` only |

### Choosing a method per use case

| Use case | Choose | Why |
|---|---|---|
| Fixed-size control struct, both directions, a few KB | `METHOD_BUFFERED` | one copy, no lock, no raw-pointer risk |
| Large data, no DMA, driver needs random access | `DO_DIRECT_IO` / `METHOD_OUT_DIRECT` | no copy; frames stay coherent until completion |
| Hardware DMA into or out of the caller's buffer | direct I/O, then `MmGetMdlPfnArray` | the MDL already holds locked physical page numbers |
| Command with no payload (reset, flush) | `METHOD_BUFFERED`, zero lengths | smallest safe path |
| An opaque value the driver never dereferences | `METHOD_NEITHER` is tolerable | it must never be used as an address |
| A buffer that must stay mapped after the call returns | direct I/O | the MDL keeps the frames locked until completion |
| Reachable by an unprivileged caller with attacker-controlled lengths | `METHOD_BUFFERED` or direct I/O | both remove the raw-pointer bug class |

## Step-by-step

1. **Choose the transfer class before writing dispatch code.** For read/write, set exactly one of
   `DO_BUFFERED_IO` or `DO_DIRECT_IO` at device creation, or neither deliberately. For each IOCTL,
   choose the `Method` in `CTL_CODE` and write the handler for that method only.
2. **Validate the declared length first**, using the field that belongs to the method: `Read.Length`,
   `Write.Length`, `InputBufferLength`, `OutputBufferLength`. Reject impossible sizes with
   `STATUS_INVALID_BUFFER_SIZE` (`0xC0000206`) and short output with `STATUS_BUFFER_TOO_SMALL`
   (`0xC0000023`).
3. **Buffered:** use `Irp->AssociatedIrp.SystemBuffer` and nothing else; there is nothing to probe. Copy
   inputs to locals before writing outputs, then set `Information` to the bytes produced.
4. **Direct:** require `Irp->MdlAddress != NULL` and `MmGetMdlByteCount >= Length`, then call
   `MmGetSystemAddressForMdlSafe(Irp->MdlAddress, NormalPagePriority)`. On `NULL`, complete with
   `STATUS_INSUFFICIENT_RESOURCES`. Never unmap, unlock, or free the I/O manager's MDL.
5. **Neither:** branch on `RequestorMode`; for `UserMode` require `KeGetCurrentIrql() <= APC_LEVEL` and
   `IoGetRequestorProcess(Irp) == PsGetCurrentProcess()`. Probe inside `__try`, copy inside the same
   `__try`, and convert the raised code to a failing status.
6. **Copy parameters once.** For a `METHOD_NEITHER` IOCTL, copy the input structure into a local before
   any validation that reads its fields. Re-reading it later is a double fetch.
7. **Do overflow-safe arithmetic:** bound with subtraction (`if (Offset > Total || Len > Total - Offset)`)
   and never multiply a caller-supplied count by an element size without a check.
8. **Set `IoStatus`, then complete once:** `Status`, `Information`,
   `IoCompleteRequest(Irp, IO_NO_INCREMENT)`, then return the status captured before the call.
9. **Prove it in the debugger.** `!irp <addr> 1` prints `System buffer=` or `Mdl=` in its header line;
   that single line shows which mechanism the request actually used.

## Complete example

One behaviour — a read fills the caller's buffer with zeros, a write discards it — exposed three times,
once per mechanism. A bidirectional `METHOD_BUFFERED` IOCTL configures the maximum transfer size, and a
`METHOD_NEITHER` IOCTL fills a caller-chosen subrange of the output buffer.

```c
/* ZeroModes.h - shared by driver and client.
   Include <ntddk.h> before this header in the driver, <windows.h> in the client. */
#pragma once

#define ZERO_DEVICE_TYPE 0x8330

#define IOCTL_ZERO_CONFIGURE \
    CTL_CODE(ZERO_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_ZERO_FILL \
    CTL_CODE(ZERO_DEVICE_TYPE, 0x801, METHOD_NEITHER, FILE_ANY_ACCESS)

typedef struct _ZERO_CONFIG {
    ULONG MaxTransfer;      /* in: requested limit, out: applied limit */
    ULONG Flags;            /* reserved, must be zero on input */
} ZERO_CONFIG;

typedef struct _ZERO_FILL_REQUEST {
    ULONG Offset;
    ULONG Length;
} ZERO_FILL_REQUEST;
```

```c
/* ZeroModes.c - the same zero device over buffered, direct, and neither I/O. */
#include <ntddk.h>
#include "ZeroModes.h"

#define ZERO_MAX_TRANSFER (1024UL * 1024UL)

typedef enum _ZERO_MODE {
    ZeroBuffered = 0,
    ZeroDirect,
    ZeroNeither
} ZERO_MODE;

typedef struct _ZERO_EXTENSION {
    ZERO_MODE      Mode;
    ULONG          MaxTransfer;
    ULONGLONG      TotalRead;
    ULONGLONG      TotalWritten;
} ZERO_EXTENSION, *PZERO_EXTENSION;

typedef struct _ZERO_MODE_ENTRY {
    const WCHAR* DevicePath;
    const WCHAR* LinkPath;
    ULONG        Flags;
    ZERO_MODE    Mode;
} ZERO_MODE_ENTRY;

static const ZERO_MODE_ENTRY g_Modes[] = {
    { L"\\Device\\ZeroBuf", L"\\??\\ZeroBuf", DO_BUFFERED_IO, ZeroBuffered },
    { L"\\Device\\ZeroDir", L"\\??\\ZeroDir", DO_DIRECT_IO,   ZeroDirect   },
    { L"\\Device\\ZeroRaw", L"\\??\\ZeroRaw", 0,              ZeroNeither  },
};

#define ZERO_MODE_COUNT (sizeof(g_Modes) / sizeof(g_Modes[0]))

static NTSTATUS ZeroComplete(PIRP Irp, NTSTATUS Status, ULONG_PTR Information)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = NT_SUCCESS(Status) ? Information : 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return Status;
}

static NTSTATUS ZeroDefault(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return ZeroComplete(Irp, STATUS_INVALID_DEVICE_REQUEST, 0);
}

static NTSTATUS ZeroCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return ZeroComplete(Irp, STATUS_SUCCESS, 0);
}

/* Direct I/O: map only after the length checks, and never unmap what you did not map. */
static NTSTATUS ZeroZeroMdl(PIRP Irp, ULONG Length)
{
    PVOID systemAddress;

    if (Irp->MdlAddress == NULL) {
        return STATUS_INVALID_PARAMETER;        /* DO_DIRECT_IO was not set */
    }
    if (MmGetMdlByteCount(Irp->MdlAddress) < Length) {
        return STATUS_INVALID_BUFFER_SIZE;
    }
    systemAddress = MmGetSystemAddressForMdlSafe(Irp->MdlAddress, NormalPagePriority);
    if (systemAddress == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    RtlZeroMemory(systemAddress, Length);
    return STATUS_SUCCESS;
}

/* Neither I/O: the only path where a raw user pointer is dereferenced. */
static NTSTATUS ZeroZeroUserBuffer(PIRP Irp, ULONG Length)
{
    PVOID buffer = Irp->UserBuffer;
    NTSTATUS status = STATUS_SUCCESS;

    if (buffer == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    if (Irp->RequestorMode == UserMode) {
        if (KeGetCurrentIrql() > APC_LEVEL ||
            IoGetRequestorProcess(Irp) != PsGetCurrentProcess()) {
            return STATUS_INVALID_DEVICE_REQUEST;
        }
    }

    __try {
        if (Irp->RequestorMode == UserMode) {
            ProbeForWrite(buffer, Length, sizeof(UCHAR));
        }
        RtlZeroMemory(buffer, Length);
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
        if (NT_SUCCESS(status)) {
            status = STATUS_ACCESS_VIOLATION;
        }
    }
    return status;
}

static NTSTATUS ZeroRead(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PZERO_EXTENSION ext = (PZERO_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);
    ULONG length = irpSp->Parameters.Read.Length;
    NTSTATUS status;

    if (length == 0 || length > ext->MaxTransfer) {
        return ZeroComplete(Irp, STATUS_INVALID_BUFFER_SIZE, 0);
    }

    switch (ext->Mode) {
    case ZeroBuffered:
        if (Irp->AssociatedIrp.SystemBuffer == NULL) {
            return ZeroComplete(Irp, STATUS_INVALID_PARAMETER, 0);
        }
        RtlZeroMemory(Irp->AssociatedIrp.SystemBuffer, length);
        break;

    case ZeroDirect:
        status = ZeroZeroMdl(Irp, length);
        if (!NT_SUCCESS(status)) {
            return ZeroComplete(Irp, status, 0);
        }
        break;

    case ZeroNeither:
        status = ZeroZeroUserBuffer(Irp, length);
        if (!NT_SUCCESS(status)) {
            return ZeroComplete(Irp, status, 0);
        }
        break;

    default:
        return ZeroComplete(Irp, STATUS_INVALID_DEVICE_REQUEST, 0);
    }

    InterlockedAdd64((volatile LONG64*)&ext->TotalRead, length);
    return ZeroComplete(Irp, STATUS_SUCCESS, length);
}

static NTSTATUS ZeroWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PZERO_EXTENSION ext = (PZERO_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);
    ULONG length = irpSp->Parameters.Write.Length;

    if (length == 0 || length > ext->MaxTransfer) {
        return ZeroComplete(Irp, STATUS_INVALID_BUFFER_SIZE, 0);
    }
    if (ext->Mode == ZeroDirect && Irp->MdlAddress == NULL) {
        return ZeroComplete(Irp, STATUS_INVALID_PARAMETER, 0);
    }
    if (ext->Mode == ZeroNeither && Irp->UserBuffer == NULL) {
        return ZeroComplete(Irp, STATUS_INVALID_PARAMETER, 0);
    }

    /* The discard path reads nothing, so it maps nothing: that is exactly why the
       I/O manager leaves the system mapping of a direct-I/O buffer to the driver. */
    InterlockedAdd64((volatile LONG64*)&ext->TotalWritten, length);
    return ZeroComplete(Irp, STATUS_SUCCESS, length);
}

/* METHOD_BUFFERED, bidirectional: input and output share one allocation. */
static NTSTATUS ZeroConfigure(PZERO_EXTENSION Ext, PIRP Irp,
                              PIO_STACK_LOCATION IrpSp, ULONG_PTR* Information)
{
    ZERO_CONFIG input;
    ZERO_CONFIG output;
    ULONG outputLength = IrpSp->Parameters.DeviceIoControl.OutputBufferLength;

    if (IrpSp->Parameters.DeviceIoControl.InputBufferLength < sizeof(input) ||
        outputLength < sizeof(output)) {
        return STATUS_BUFFER_TOO_SMALL;
    }
    if (Irp->AssociatedIrp.SystemBuffer == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    /* Read the input out of the shared allocation before writing the output to it. */
    RtlCopyMemory(&input, Irp->AssociatedIrp.SystemBuffer, sizeof(input));

    Ext->MaxTransfer = input.MaxTransfer < ZERO_MAX_TRANSFER
                     ? input.MaxTransfer : ZERO_MAX_TRANSFER;

    output.MaxTransfer = Ext->MaxTransfer;
    output.Flags = 0;
    RtlCopyMemory(Irp->AssociatedIrp.SystemBuffer, &output, sizeof(output));
    *Information = sizeof(output);
    return STATUS_SUCCESS;
}

/* METHOD_NEITHER done defensively: copy once, bound by subtraction, probe, SEH. */
static NTSTATUS ZeroFill(PIRP Irp, PIO_STACK_LOCATION IrpSp, ULONG_PTR* Information)
{
    ZERO_FILL_REQUEST request;
    PUCHAR output = (PUCHAR)Irp->UserBuffer;
    ULONG outputLength = IrpSp->Parameters.DeviceIoControl.OutputBufferLength;
    NTSTATUS status = STATUS_SUCCESS;

    if (IrpSp->Parameters.DeviceIoControl.InputBufferLength < sizeof(request)) {
        return STATUS_INVALID_PARAMETER;
    }
    if (IrpSp->Parameters.DeviceIoControl.Type3InputBuffer == NULL || output == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    __try {
        if (Irp->RequestorMode == UserMode) {
            ProbeForRead(IrpSp->Parameters.DeviceIoControl.Type3InputBuffer,
                         sizeof(request), sizeof(ULONG));
        }
        RtlCopyMemory(&request, IrpSp->Parameters.DeviceIoControl.Type3InputBuffer,
                      sizeof(request));
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        return GetExceptionCode();
    }

    /* Offset + Length can overflow; Total - Offset cannot. */
    if (request.Offset > outputLength || request.Length > outputLength - request.Offset) {
        return STATUS_BUFFER_TOO_SMALL;
    }
    if (request.Length == 0) {
        *Information = 0;
        return STATUS_SUCCESS;
    }
    if (Irp->RequestorMode == UserMode) {
        if (KeGetCurrentIrql() > APC_LEVEL ||
            IoGetRequestorProcess(Irp) != PsGetCurrentProcess()) {
            return STATUS_INVALID_DEVICE_REQUEST;
        }
    }

    __try {
        if (Irp->RequestorMode == UserMode) {
            ProbeForWrite(output + request.Offset, request.Length, sizeof(UCHAR));
        }
        RtlZeroMemory(output + request.Offset, request.Length);
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
        if (NT_SUCCESS(status)) {
            status = STATUS_ACCESS_VIOLATION;
        }
    }

    if (NT_SUCCESS(status)) {
        *Information = request.Length;
    }
    return status;
}

static NTSTATUS ZeroDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PZERO_EXTENSION ext = (PZERO_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION irpSp = IoGetCurrentIrpStackLocation(Irp);
    ULONG_PTR information = 0;
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;

    switch (irpSp->Parameters.DeviceIoControl.IoControlCode) {
    case IOCTL_ZERO_CONFIGURE:
        status = ZeroConfigure(ext, Irp, irpSp, &information);
        break;
    case IOCTL_ZERO_FILL:
        status = ZeroFill(Irp, irpSp, &information);
        break;
    default:
        break;
    }
    return ZeroComplete(Irp, status, information);
}

static VOID ZeroUnload(PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT device = DriverObject->DeviceObject;
    ULONG index;

    for (index = 0; index < ZERO_MODE_COUNT; index++) {
        UNICODE_STRING link;
        RtlInitUnicodeString(&link, g_Modes[index].LinkPath);
        (VOID)IoDeleteSymbolicLink(&link);
    }
    while (device != NULL) {
        PDEVICE_OBJECT next = device->NextDevice;
        IoDeleteDevice(device);
        device = next;
    }
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT devices[ZERO_MODE_COUNT] = { NULL };
    ULONG deviceCount = 0;
    ULONG linkCount = 0;
    ULONG index;
    NTSTATUS status = STATUS_SUCCESS;

    UNREFERENCED_PARAMETER(RegistryPath);

    for (index = 0; index <= IRP_MJ_MAXIMUM_FUNCTION; index++) {
        DriverObject->MajorFunction[index] = ZeroDefault;
    }
    DriverObject->MajorFunction[IRP_MJ_CREATE] = ZeroCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = ZeroCreateClose;
    DriverObject->MajorFunction[IRP_MJ_READ] = ZeroRead;
    DriverObject->MajorFunction[IRP_MJ_WRITE] = ZeroWrite;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = ZeroDeviceControl;
    DriverObject->DriverUnload = ZeroUnload;

    for (index = 0; index < ZERO_MODE_COUNT; index++) {
        UNICODE_STRING name;
        UNICODE_STRING link;
        PZERO_EXTENSION ext;

        RtlInitUnicodeString(&name, g_Modes[index].DevicePath);
        RtlInitUnicodeString(&link, g_Modes[index].LinkPath);

        status = IoCreateDevice(DriverObject, sizeof(ZERO_EXTENSION), &name,
                                FILE_DEVICE_UNKNOWN, 0, FALSE, &devices[index]);
        if (!NT_SUCCESS(status)) {
            goto Cleanup;
        }
        deviceCount++;

        ext = (PZERO_EXTENSION)devices[index]->DeviceExtension;
        RtlZeroMemory(ext, sizeof(*ext));
        ext->Mode = g_Modes[index].Mode;
        ext->MaxTransfer = ZERO_MAX_TRANSFER;

        devices[index]->Flags |= g_Modes[index].Flags;

        status = IoCreateSymbolicLink(&link, &name);
        if (!NT_SUCCESS(status)) {
            goto Cleanup;
        }
        linkCount++;

        devices[index]->Flags &= ~DO_DEVICE_INITIALIZING;
    }
    return STATUS_SUCCESS;

Cleanup:
    for (index = 0; index < linkCount; index++) {
        UNICODE_STRING link;
        RtlInitUnicodeString(&link, g_Modes[index].LinkPath);
        (VOID)IoDeleteSymbolicLink(&link);
    }
    for (index = 0; index < deviceCount; index++) {
        IoDeleteDevice(devices[index]);
    }
    return status;
}
```

Driver-built MDL, for a buffer that did not arrive in an IRP (an inverted call, a section view, a DMA
staging buffer supplied by another driver):

```c
/* Lock, map, use, unlock, free - in that order. The lock is tracked separately
   because MmProbeAndLockPages raises on failure and must not be unlocked then. */
static NTSTATUS ZeroFillCallerBuffer(PVOID UserBuffer, ULONG Length)
{
    PMDL mdl = IoAllocateMdl(UserBuffer, Length, FALSE, FALSE, NULL);
    PVOID mapped = NULL;
    BOOLEAN locked = FALSE;
    NTSTATUS status = STATUS_SUCCESS;

    if (mdl == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    __try {
        MmProbeAndLockPages(mdl, UserMode, IoWriteAccess);
        locked = TRUE;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
    }

    if (NT_SUCCESS(status)) {
        mapped = MmGetSystemAddressForMdlSafe(mdl, NormalPagePriority);
        if (mapped == NULL) {
            status = STATUS_INSUFFICIENT_RESOURCES;
        } else {
            RtlZeroMemory(mapped, Length);
        }
    }
    if (locked) {
        MmUnlockPages(mdl);
    }
    IoFreeMdl(mdl);             /* only because it was not attached to an IRP */
    return status;
}
```

A client opens `\\.\ZeroBuf`, `\\.\ZeroDir`, and `\\.\ZeroRaw`, calls `ReadFile` and `WriteFile` on
each, and sends `IOCTL_ZERO_CONFIGURE` with `MaxTransfer = 4096`; the pattern is the one in
`wk-driver-client-protocol`. `IOCTL_ZERO_FILL` is `METHOD_NEITHER`, so `DeviceIoControl` passes `&fill`
as the raw input pointer and `buffer` as the raw output pointer, and the driver receives
`Type3InputBuffer` and `UserBuffer` and performs every check itself.

## Variations and advanced use

- **Mapping into the current process instead of system space.** `MmMapLockedPagesSpecifyCache(Mdl,
  UserMode, MmCached, NULL, FALSE, NormalPagePriority)` returns an address valid only in the current
  process; the `KernelMode` form behaves like `MmGetSystemAddressForMdlSafe`. Use the `UserMode` form
  only in the caller's context, hand the address back deliberately, and `MmUnmapLockedPages` before
  leaving. A system thread has no user address space, so the `UserMode` form fails there.
- **DMA from a direct-I/O buffer.** `MmGetMdlPfnArray(Mdl)` exposes the locked physical page numbers and
  `MmGetMdlByteOffset` the offset into the first frame; program the hardware from those and never map
  the buffer first, which would do work the transfer does not need.
- **Zero-length requests.** A zero-length read or write is legal at the I/O manager level and does reach
  the dispatch routine. Decide between success with `Information == 0` and
  `STATUS_INVALID_BUFFER_SIZE`, and document the choice for the client author.
- **Cache behaviour.** The safe mapping is `MmCached`. A device that needs write-combined or uncached
  access must use `MmMapLockedPagesSpecifyCache` with `MmWriteCombined` or `MmNonCached`, and must unmap
  it before the IRP that owns the MDL completes.

## Failure modes and diagnosis

| Symptom | Likely cause | First command / fix |
|---|---|---|
| `0xD1 DRIVER_IRQL_NOT_LESS_OR_EQUAL` in a read/write handler | neither flag set but the code assumed `SystemBuffer`, or a stale MDL pointer used at `DISPATCH_LEVEL` | `!irp <addr> 1` shows `Mdl=` or `System buffer=`; `!analyze -v` for the faulting address |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA` reading `Mdl->MappedSystemVa` | mapping never made by `MmGetSystemAddressForMdlSafe` | map before use and check for `NULL` |
| `0xA IRQL_NOT_LESS_OR_EQUAL` touching `UserBuffer` | neither-I/O handler dereferenced a user pointer above `APC_LEVEL` | gate on `KeGetCurrentIrql() <= APC_LEVEL` |
| `0x8E KERNEL_MODE_EXCEPTION_NOT_HANDLED`, `0xC0000005` | raw pointer dereferenced without `__try`, or in the wrong process | `ProbeForWrite` plus `__try`; check `IoGetRequestorProcess` |
| `0xC0000206 STATUS_INVALID_BUFFER_SIZE` | declared length rejected: zero, or above the MDL size | compare `Parameters.Read.Length` with `MmGetMdlByteCount` |
| `0xC0000023 STATUS_BUFFER_TOO_SMALL` | output buffer shorter than the structure the IOCTL returns | client must size the buffer from a documented structure |
| `0xC00000BB STATUS_NOT_SUPPORTED` | method mismatch: the handler read `SystemBuffer` for a `METHOD_NEITHER` code, found `NULL`, refused | decode the `CTL_CODE` and fix the handler, not the client |
| Transfer truncated at a 4 KB boundary | `MmGetMdlByteCount` ignored, or `Length` read from the wrong field | assert `Length == MmGetMdlByteCount` in debug builds |
| Caller's buffer partly filled, no error | `Information` unset, or set to the requested length instead of the actual one | set `Information` on every success path |
| Pooled kernel memory in a client's output buffer | `Information > OutputBufferLength` on a buffered IOCTL | clamp `Information` to the declared output length |
| Bugcheck inside `IoCompleteRequest` | `Information` larger than the system buffer allocation; the copy-back uses it as a length | same fix |
| `0xC2 BAD_POOL_CALLER` after a direct-I/O request | driver called `MmUnlockPages`/`IoFreeMdl` on the I/O manager's MDL | remove both; that MDL is not yours |

### Attack table: what a hostile caller does to a buffer handler

| Attack | How it works | Defence |
|---|---|---|
| Double fetch | the handler reads a length or offset from user memory, validates it, then re-reads it and uses a different value | copy the whole input structure into a local **once**, then use only the local |
| Integer overflow in bounds math | `Offset + Length > BufferLength` wraps and passes | compare by subtraction: `Offset > Total \|\| Length > Total - Offset` |
| Multiplication overflow | `Count * sizeof(Element)` wraps in 32-bit arithmetic | compute in `SIZE_T`/`ULONG64` and check `Count > MAX / sizeof(Element)` |
| Declared size larger than the mapping | `InputBufferLength` claims 64 bytes, the caller mapped 4 | probe exactly the range you read; never trust the declared length for `METHOD_NEITHER` |
| Kernel pointer from a user caller | the "buffer" is a kernel VA or a device physical mapping | probe, which rejects kernel addresses, and reject `RequestorMode == UserMode` with an address above the user limit |
| Unaligned structure access | a 32-bit client passes an odd address; the driver reads `ULONG`/`ULONGLONG` fields in place | copy into aligned locals with `RtlCopyMemory`; probe with the structure's alignment |
| Information disclosure | `Information` above the output length, or a buffer copied out before it was initialized | zero the output buffer, then set `Information` to the exact byte count |
| Use after completion | the handler stored a user pointer or mapping and used it after `IoCompleteRequest` | finish all copies during the request; never keep a user VA past completion |
| 32-bit caller, 64-bit kernel | the shared structure contains pointers, so field layout differs | never put pointers in a shared IOCTL structure; use fixed-width integers |
| Method confusion | the client sends a `METHOD_BUFFERED` code while the handler reads `Type3InputBuffer` | decode the method bits and switch on the control code, not on an assumption |

## Safety, IRQL, and cleanup checklist

- [ ] Exactly one of `DO_BUFFERED_IO` / `DO_DIRECT_IO` is set, or neither is set and every read/write
      path handles a raw pointer.
- [ ] Every IOCTL handler switches on `IoControlCode` and knows its method from `CTL_CODE`, independent
      of the device flags.
- [ ] `SystemBuffer` is checked for `NULL` and treated as `max(InputBufferLength, OutputBufferLength)`
      bytes, not zeroed.
- [ ] Input fields are copied to locals before output fields are written in a `METHOD_BUFFERED` handler.
- [ ] `MmGetSystemAddressForMdlSafe` is called after length validation, and `NULL` maps to
      `STATUS_INSUFFICIENT_RESOURCES`.
- [ ] The I/O manager's MDL is never unlocked or freed by the driver.
- [ ] Each driver-built MDL is locked once, unlocked once on success, and freed once; a failed
      `MmProbeAndLockPages` is never followed by `MmUnlockPages`.
- [ ] Every `MmMapLockedPagesSpecifyCache` mapping is paired with `MmUnmapLockedPages`.
- [ ] Raw user pointers are touched only in `UserMode` requests, at `IRQL <= APC_LEVEL`, in the
      requesting process.
- [ ] Every raw dereference is inside `__try`, with the probe inside the same `__try`, and the raised
      code becomes a failing `NTSTATUS`.
- [ ] All size arithmetic is overflow-safe and `Information` is clamped to the declared buffer length.
- [ ] `Information` is zero on every failure path and exactly the transferred count on success.
- [ ] No user VA, MDL, or mapping survives `IoCompleteRequest`.
- [ ] Shared structures use fixed-width types, explicit sizes, no pointers, and a size or version field
      the driver validates.

## Verification

1. **Build** the driver and a client that opens `\\.\ZeroBuf`, `\\.\ZeroDir`, and `\\.\ZeroRaw`, reads,
   writes, and sends `IOCTL_ZERO_CONFIGURE` with `MaxTransfer = 4096`:

   ```
   msbuild ZeroModes.vcxproj /p:Configuration=Debug /p:Platform=x64
   sc create ZeroModes type= kernel binPath= C:\drivers\ZeroModes.sys
   sc start ZeroModes
   ```

2. **Prove which mechanism each handle used.** The header line of `!irp` names the buffer the request
   actually carries:

   ```
   kd> !irp <IRP address> 1
   ```

   `No Mdl: System buffer=...` means buffered, `Mdl=...: No System Buffer` means direct, and
   `No Mdl: No System Buffer` means neither. A mismatch against the device flags explains most "the
   buffer is garbage" reports.

3. **Prove the fields** in the dispatch routine (the IRP is the second x64 argument, in `rdx`):

   ```
   kd> bp ZeroModes!ZeroRead
   kd> g
   kd> dt nt!_IRP @rdx MdlAddress AssociatedIrp.SystemBuffer UserBuffer RequestorMode
   kd> dt nt!_IO_STACK_LOCATION poi(nt!_IRP @rdx Tail.Overlay.CurrentStackLocation) Parameters.Read
   ```

4. **Attack your own handler.** Send `IOCTL_ZERO_FILL` with `Offset = 0xFFFFFFF0` and `Length = 0x40`;
   the subtraction check must reject it with `ERROR_INVALID_PARAMETER` (87) and no bugcheck. Then send
   `Offset = 250`, `Length = 16` against a 256-byte buffer and confirm success with `Information == 16`.

5. **Run under Driver Verifier**, which enables special pool and catches unmapping or freeing the I/O
   manager's MDL. A clean stop with verifier loaded, and no `0xC2`/`0x50` in the log, is the acceptance
   test:

   ```
   verifier /standard /driver ZeroModes.sys
   sc stop ZeroModes
   sc delete ZeroModes
   ```
