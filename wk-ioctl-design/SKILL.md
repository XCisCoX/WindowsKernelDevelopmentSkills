---
name: wk-ioctl-design
description: Use when designing or reviewing a Windows driver IOCTL interface: CTL_CODE bit layout and function and device-type ranges, METHOD_BUFFERED versus the direct and NEITHER methods, access bits versus the device ACL, input and output length validation, versioned request structs, integer-overflow checks, safe Type3InputBuffer probing, and a complete four-command driver with a client that probes buffer sizes.
---

# IOCTL design: a control interface that survives versioning and attackers

An IOCTL is a 32-bit number plus two buffers, and the number alone decides who may call it, how the
buffers reach the driver, and how many bytes exist. That compactness is why interfaces rot: a struct
grows, an access bit is set to `FILE_ANY_ACCESS` for convenience, a `METHOD_NEITHER` pointer is
trusted because it worked in a debug build. The rule that prevents most disasters: **treat every
length, count, version, field, and pointer in an IOCTL as attacker-controlled until checked, and copy
user memory exactly once.** The driver below exposes four commands with fixed numeric codes, a
buffer-size probe contract, and a client that discovers sizes instead of assuming them.

## When to use

- You are defining a new `IRP_MJ_DEVICE_CONTROL` interface and must assign device type, function
  numbers, methods, and access bits.
- You are reviewing a dispatch routine and must prove no path reads or writes a buffer before its
  length is validated.
- A client gets `ERROR_INVALID_PARAMETER` (87), `ERROR_INSUFFICIENT_BUFFER` (122), or
  `ERROR_ACCESS_DENIED` (5) from `DeviceIoControl` and the cause is not obvious.
- You must choose between `METHOD_BUFFERED`, `METHOD_IN_DIRECT`, `METHOD_OUT_DIRECT`, and
  `METHOD_NEITHER` for a specific transfer size and caller context.
- A shared struct changed and older clients must keep working, or must fail cleanly.
- A `Type3InputBuffer` handler needs `ProbeForRead`/`ProbeForWrite`, SEH, and IRQL guards.
- You are hardening an interface: access bits, device DACL, and driver checks must agree.

Do not use for the user-mode side alone (`wk-driver-client-protocol`), the IRP field layout and
completion rules (`wk-irp-basics`), or queued and canceled IOCTLs (`wk-irp-queues-cancel`).

## Prerequisites and related skills

- **IRQL:** a dispatch routine usually runs at PASSIVE_LEVEL in the caller's thread, but a stacked
  driver may call it at DISPATCH_LEVEL. Buffered IOCTL code can run at DISPATCH; any access to a
  user-mode pointer cannot.
- **Headers:** `<ntddk.h>` or `<wdm.h>`; `<wdmsec.h>` plus `wdmsec.lib` for `IoCreateDeviceSecure`;
  `<ntintsafe.h>` for overflow-safe arithmetic.
- **Build target:** `ExAllocatePool2` requires Windows 10 2004 or later with a matching WDK. On older
  targets use `ExAllocatePoolWithTag` and check for NULL.
- **Device state:** the device object must exist, carry the same `DeviceType` used in the codes, and
  have a symbolic link and a security descriptor before a client can open it.
- Siblings: `wk-driver-anatomy` (the same protocol end to end), `wk-irp-basics` (buffer fields and
  completion), `wk-kernel-exceptions` (probe and SEH details), `wk-object-manager` (SDDL devices),
  `wk-driver-client-protocol` (the Win32 side).

## Mental model

The I/O manager does three things before the dispatch runs: it validates the access bits of the
control code against the handle's granted access, it builds the buffers the method field requests,
and it fills `Parameters.DeviceIoControl` with the two lengths. The driver never sees the user's
buffer for a buffered code and never sees a raw pointer for a direct code.

```
DeviceIoControl(code, in, inLen, out, outLen)
   |
   v  I/O manager: access check, then buffer setup per method
   +-- METHOD_BUFFERED ---> SystemBuffer = max(inLen, outLen); input copied in
   +-- METHOD_IN_DIRECT --> SystemBuffer = input copy; MDL over the second buffer (read)
   +-- METHOD_OUT_DIRECT -> SystemBuffer = input copy; MDL over the second buffer (write)
   +-- METHOD_NEITHER ----> Type3InputBuffer and UserBuffer raw; nothing copied, nothing probed
   |
   v  dispatch: validate lengths and versions, act, set IoStatus.Status and Information
   |
   v  completion: buffered output copied back, MDL unmapped, caller unblocked
```

Because the method is encoded per control code, one driver can mix methods safely: a status query
can be buffered while a multi-megabyte blob uses `METHOD_OUT_DIRECT`. `DO_BUFFERED_IO` and
`DO_DIRECT_IO` on the device object affect only `IRP_MJ_READ` and `IRP_MJ_WRITE`; they do not change
IOCTL buffering, which is a common source of confusion.

The four commands in this skill and the exact numbers they produce:

| Command | Device type | Access | Function | Method | Numeric value |
|---|---|---|---|---|---|
| `IOCTL_GET_VERSION` | `0x8C40` | 0 | `0x800` | 0 | `0x8C402000` |
| `IOCTL_READ_STATE` | `0x8C40` | 1 | `0x801` | 0 | `0x8C406004` |
| `IOCTL_WRITE_STATE` | `0x8C40` | 2 | `0x802` | 0 | `0x8C40A008` |
| `IOCTL_RESET_STATE` | `0x8C40` | 2 | `0x803` | 3 | `0x8C40A00F` |

Write those literals in a debugger, not in source. Source derives them from `CTL_CODE`, and
`!ioctldecode 0x8C40A00F` reproduces the fields when a log line shows only the number.

## Key APIs and structures

| API or structure | Purpose | IRQL or context | Notes |
|---|---|---|---|
| `CTL_CODE(DeviceType, Function, Method, Access)` | Builds the 32-bit code | compile time | `devioctl.h`, also `winioctl.h` |
| `IoGetCurrentIrpStackLocation(Irp)` | Current `IO_STACK_LOCATION` | any | macro; pointer changes per layer |
| `Parameters.DeviceIoControl` | `IoControlCode`, `InputBufferLength`, `OutputBufferLength`, `Type3InputBuffer` | any | one union member per major function |
| `Irp->AssociatedIrp.SystemBuffer` | Buffered input and output | any | NULL when both lengths are 0 |
| `Irp->MdlAddress` | Direct-method buffer | any | map with `MmGetSystemAddressForMdlSafe` |
| `Irp->UserBuffer` | Raw output pointer for `METHOD_NEITHER` | < DISPATCH_LEVEL | never dereference unchecked |
| `IoValidateDeviceIoControlAccess` | Run-time access re-check | any | succeeds for kernel requestors |
| `ProbeForRead` / `ProbeForWrite` | Validate a user address range | < DISPATCH_LEVEL | raises `STATUS_ACCESS_VIOLATION` |
| `RtlULongAdd` / `RtlULongMult` | Overflow-checked arithmetic | any | return `STATUS_INTEGER_OVERFLOW` |
| `IoCreateDeviceSecure` | Device with an SDDL DACL | PASSIVE_LEVEL | needs `wdmsec.h`, `wdmsec.lib` |

## CTL_CODE bit layout and numbering

```
 31 30 ............ 16 15 14 13 12 ............ 2 1 0
+--------------------+-------+-----+---------------+-----+
|   DeviceType (16)  |Access |Cust |  Function(12) |Meth |
+--------------------+-------+-----+---------------+-----+
value = (DeviceType << 16) | (Access << 14) | (Function << 2) | Method
```

| Field | Bits | Width | Mask | Legal values |
|---|---|---|---|---|
| DeviceType | 16-31 | 16 | `0xFFFF0000` | `0x0000`-`0x7FFF` Microsoft; `0x8000`-`0xFFFF` vendor |
| Access | 14-15 | 2 | `0x0000C000` | 0 any, 1 read, 2 write, 3 read and write |
| Function | 2-13 | 12 | `0x00003FFC` | `0x000`-`0x7FF` Microsoft; `0x800`-`0xFFF` vendor |
| Method | 0-1 | 2 | `0x00000003` | 0 buffered, 1 in-direct, 2 out-direct, 3 neither |

Two bits in that layout are flags rather than fields. Bit 31 is the top bit of `DeviceType` and marks
a vendor-assigned type; bit 13 is the top bit of `Function` and marks a vendor-defined function.
Choosing a device type of `0x8000` or above and a function of `0x800` or above sets both flags for
free, which is why those ranges are mandatory for private interfaces.

Decode with shifts, never with hand-written masks that drift:

```c
#define IOCTL_DEVTYPE(code)  (((code) >> 16) & 0xFFFF)
#define IOCTL_ACCESS(code)   (((code) >> 14) & 0x3)
#define IOCTL_FUNCTION(code) (((code) >> 2) & 0xFFF)
#define IOCTL_METHOD(code)   ((code) & 0x3)
```

`wdm.h` also provides `DEVICE_TYPE_FROM_CTL_CODE(code)` and `METHOD_FROM_CTL_CODE(code)`. Treat the
function number as permanent: once shipped, `0x802` must mean the same operation forever. Add new
commands at the next free number; never renumber to keep a list tidy.

- One device type per interface, equal to `DEVICE_OBJECT.DeviceType`, in the vendor range.
- Functions `0x800` and up, assigned once, never reused or reordered.
- The dispatch checks `(code & 0xFFFF0000) == (MY_TYPE << 16)` before the switch; the I/O manager
  does not enforce that match, so a foreign code can arrive on the same handle.
- `FILE_DEVICE_UNKNOWN` (`0x22`) is a Microsoft-assigned type. Using it with vendor functions works
  but collides with every other driver that made the same choice; prefer a vendor type.

## Choosing a method

| Method | Input | Output | Driver reads with | Right for |
|---|---|---|---|---|
| `METHOD_BUFFERED` | copied to `SystemBuffer` | same buffer, copied back | `SystemBuffer` | structs up to a few kilobytes; any case where a copy is cheaper than the complexity |
| `METHOD_IN_DIRECT` | copied to `SystemBuffer` | MDL over the second buffer, read-only | `SystemBuffer` and `MdlAddress` | large data flowing client to driver |
| `METHOD_OUT_DIRECT` | copied to `SystemBuffer` | MDL over the second buffer, writable | `SystemBuffer` and `MdlAddress` | large data flowing driver to client |
| `METHOD_NEITHER` | raw pointer in `Type3InputBuffer` | raw pointer in `UserBuffer` | neither | no buffers at all, or a highest-level driver that maps memory itself |

The two direct methods are the same mechanism: the input is always buffered, and the second buffer is
described by an MDL. The difference is the intended direction. `METHOD_IN_DIRECT` means the driver
reads that buffer, so it is locked for read access; `METHOD_OUT_DIRECT` means the driver writes it,
so it is locked for write access. `Parameters.DeviceIoControl.OutputBufferLength` is the size of the
MDL buffer in both cases.

`METHOD_NEITHER` is the only method with no safety net. Use it when there is no buffer (the code
alone carries the request) or when the driver is the top of the stack and must map memory itself. It
is wrong for a driver that can be called from an arbitrary thread at raised IRQL, because the
pointers are meaningful only in the requesting process.

## Access bits, ACLs, and who can call what

Two independent gates decide whether an IOCTL reaches the dispatch:

1. **The device DACL** decides who may open a handle and with which rights. `CreateFileW` asks for
   `GENERIC_READ`, `GENERIC_WRITE`, or both; the security descriptor grants or denies them. A denied
   open fails with 5 and no IRP is ever created.
2. **The access field of the code** decides which of the granted rights a particular IOCTL needs. If
   the code requires `FILE_WRITE_DATA` and the handle was opened read-only, the I/O manager rejects
   the request before the dispatch runs.

| Access value | Constant | Handle must have | Use for |
|---|---|---|---|
| 0 | `FILE_ANY_ACCESS` | any handle to the device | queries that disclose nothing secret and change nothing |
| 1 | `FILE_READ_ACCESS` (`FILE_READ_DATA`) | read access | state, counters, telemetry |
| 2 | `FILE_WRITE_ACCESS` (`FILE_WRITE_DATA`) | write access | changing state, reset, reconfiguration |
| 3 | both | read and write | commands that read and write in one call |

The access field is 2 bits, so `3` is legal and means both. A handle opened with neither read nor
write can still send `FILE_ANY_ACCESS` codes; that is exactly why such codes must be harmless.

The ACL is the stronger control. A device created with `IoCreateDeviceSecure` and the SDDL
`D:P(A;;GA;;;SY)(A;;GA;;;BA)` can only be opened by LocalSystem and administrators, so an unelevated
client gets error 5 on `CreateFileW` regardless of access bits. If a device grants read access to
`BU` (built-in users), a client sends every `FILE_READ_ACCESS` command unelevated and every
`FILE_WRITE_ACCESS` command fails in the I/O manager. Test both identities.

`IoValidateDeviceIoControlAccess(Irp, FILE_WRITE_ACCESS)` lets a driver demand more than the code
declares, for example to keep a diagnostic reset admin-only while the code must stay
`FILE_ANY_ACCESS` for compatibility. It returns `STATUS_SUCCESS` immediately when
`Irp->RequestorMode == KernelMode`, so it is not a defence against a malicious kernel component.

## Validating lengths, versions, and arithmetic

Every handler performs the same five checks in the same order, before touching any buffer:

1. **Device type** in the high 16 bits matches the driver.
2. **Input length** is at least `sizeof(header)` and, for a fixed struct, exactly `sizeof(struct)`.
3. **Output length** is at least the struct the driver intends to write.
4. **Version and Size fields** inside the struct are acceptable.
5. **All derived arithmetic** is overflow-checked, and every count is bounded by an absolute cap.

The probe contract used here: when the output buffer is too small, the driver completes with
`STATUS_BUFFER_TOO_SMALL` (`0xC0000023`, error 122 for the client) and, if at least four bytes of
output exist, writes the required size into the first `ULONG` and sets `Information` to four. A
client can call once with a four-byte buffer and learn the size without a second command.
`Information` must never exceed the output length: the completion path copies `Information` bytes
into the caller's buffer.

Never compute a size in `ULONG` and trust it:

```c
static BOOLEAN ComputePayloadSize(ULONG Count, ULONG ElementSize, ULONG *Total)
{
    ULONG bytes;

    if (ElementSize == 0 || Count > MAXULONG / ElementSize) {
        return FALSE;
    }
    bytes = Count * ElementSize;
    if (bytes > 1024 * 1024) {          /* absolute cap, independent of the multiply */
        return FALSE;
    }
    *Total = bytes;
    return TRUE;
}

/* Equivalent, with a status instead of a boolean, from ntintsafe.h: */
if (!RtlULongMult(count, elementSize, &bytes) ||
    !RtlULongAdd(bytes, FIELD_OFFSET(IOCTL_BULK, Data), &total)) {
    status = STATUS_INTEGER_OVERFLOW;   /* 0xC0000095 */
}
```

The overflow must be checked before the allocation, not after: a wrapped product produces a small
allocation followed by a large copy, which is a pool overrun, not a failed request.

## METHOD_NEITHER without a crash

The input pointer arrives in `Parameters.DeviceIoControl.Type3InputBuffer`, the output pointer in
`Irp->UserBuffer`, with lengths in the stack location. Neither has been validated. The safe sequence:

1. If `KeGetCurrentIrql() >= DISPATCH_LEVEL`, refuse. A user address cannot be touched at DISPATCH,
   and an upper driver may have raised IRQL.
2. If `Irp->RequestorMode == KernelMode` the pointer is a kernel address; still validate the length.
3. For user mode, reject a NULL pointer with a non-zero length.
4. `ProbeForRead(pointer, length, alignment)` inside `__try`, then copy into a local or pool buffer
   in the same `__try`.
5. Validate the **copy**, never the original. Re-reading the user buffer after validation is a double
   fetch: another thread in the client process can change the data between the two reads.
6. For output, build the result in kernel memory, then `ProbeForWrite` and copy once.
7. Map the failure to `STATUS_ACCESS_VIOLATION` (`0xC0000005`) or `STATUS_INVALID_PARAMETER`; never
   let the exception escape.

```c
static NTSTATUS CaptureRequest(PIRP Irp, PIOCTL_RESET_REQUEST Out)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PVOID input = sp->Parameters.DeviceIoControl.Type3InputBuffer;

    if (Irp->RequestorMode != KernelMode) {
        if (KeGetCurrentIrql() >= DISPATCH_LEVEL || input == NULL) {
            return STATUS_INVALID_DEVICE_REQUEST;
        }
        __try {
            ProbeForRead(input, sizeof(*Out), __alignof(IOCTL_RESET_REQUEST));
            RtlCopyMemory(Out, input, sizeof(*Out));
        } __except (EXCEPTION_EXECUTE_HANDLER) {
            return GetExceptionCode();
        }
        return STATUS_SUCCESS;
    }
    if (input == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlCopyMemory(Out, input, sizeof(*Out));
    return STATUS_SUCCESS;
}
```

If a struct ever contains a pointer-sized field, check `IoIs32bitProcess(Irp)` and reject the call
from a 32-bit client: the layout differs and the pointer values are unusable. Keeping pointers out of
IOCTL structs avoids the problem entirely.

## Step-by-step

1. **Pick the device type.** A vendor value in `0x8000`-`0xFFFF`, in one macro used by both
   `IoCreateDeviceSecure` and every `CTL_CODE`. Check: `!devobj` shows the same `DeviceType`.
2. **Assign function numbers.** Start at `0x800`, one per operation, in an order that still makes
   sense in five years. Check: no number is reused and none is below `0x800`.
3. **Choose a method per command.** Default to `METHOD_BUFFERED`; use `METHOD_OUT_DIRECT` only for
   large results, `METHOD_IN_DIRECT` only for large inputs, `METHOD_NEITHER` only for commands with
   no buffer or a top-of-stack driver. Check: every direct code has a matching
   `MmGetSystemAddressForMdlSafe` and a NULL check.
4. **Choose access bits from the data flow,** not from convenience. Check: every state-changing code
   requires `FILE_WRITE_DATA`, and `FILE_ANY_ACCESS` appears only on read-only queries.
5. **Write the shared header** with `Size` and `Version` first, fixed-width types, no pointers, and
   `C_ASSERT` on every `sizeof` and on the offset of the first variable field. Check: the header
   compiles in kernel and user mode.
6. **Decide the short-buffer contract:** fail with a required size, or fail flat. Check:
   `Information` never exceeds `OutputBufferLength` on any path.
7. **Write the dispatch** in the five-check order, with the device-type check before the switch.
   Check: no dereference of `SystemBuffer`, `Type3InputBuffer`, or `UserBuffer` appears above the
   length check that protects it.
8. **Create the device with a DACL** and test from an unelevated account. Check: the open fails with
   error 5 for the operations the DACL denies and succeeds for the ones it allows.
9. **Write the client** that calls `GET_VERSION`, probes with a short buffer, then calls the real
   command with the discovered size. Check: the client validates `Size` and `Version` in the reply.
10. **Verify** with Driver Verifier, `!ioctldecode`, and paired 32-bit and 64-bit client builds.

## Complete example

Shared protocol header, compiled by the driver and the client:

```c
// IoctlState.h
#pragma once

#ifdef _KERNEL_MODE
#include <ntddk.h>
#else
#include <windows.h>
#include <winioctl.h>
#endif

#define IOCTL_DEVICE_TYPE       0x8C40
#define IOCTL_PROTOCOL_VERSION  1
#define IOCTL_LABEL_CHARS       32
#define IOCTL_FLAG_PERSIST      0x00000001
#define IOCTL_FLAG_VALID_MASK   0x00000001

typedef struct _IOCTL_VERSION_INFO {
    ULONG Size;
    ULONG ProtocolVersion;
    ULONG StateSize;
    ULONG MaxStateSize;
    ULONG Capabilities;
    ULONG Reserved;
} IOCTL_VERSION_INFO, *PIOCTL_VERSION_INFO;

typedef struct _IOCTL_STATE {
    ULONG Size;
    ULONG Version;
    ULONGLONG Counter;
    ULONG Flags;
    ULONG LabelChars;
    WCHAR Label[IOCTL_LABEL_CHARS];
} IOCTL_STATE, *PIOCTL_STATE;

typedef struct _IOCTL_RESET_REQUEST {
    ULONG Size;
    ULONG Version;
    ULONG Flags;
} IOCTL_RESET_REQUEST, *PIOCTL_RESET_REQUEST;

#define IOCTL_GET_VERSION   CTL_CODE(IOCTL_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_READ_STATE    CTL_CODE(IOCTL_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_READ_ACCESS)
#define IOCTL_WRITE_STATE   CTL_CODE(IOCTL_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_WRITE_ACCESS)
#define IOCTL_RESET_STATE   CTL_CODE(IOCTL_DEVICE_TYPE, 0x803, METHOD_NEITHER,  FILE_WRITE_ACCESS)

C_ASSERT(sizeof(IOCTL_VERSION_INFO) == 24);
C_ASSERT(sizeof(IOCTL_STATE) == 88);
C_ASSERT(sizeof(IOCTL_RESET_REQUEST) == 12);
C_ASSERT(FIELD_OFFSET(IOCTL_STATE, Label) == 24);
```

The driver. It is a `.c` file, so no `extern "C"`; every routine is non-paged because the dispatch can
be entered above PASSIVE_LEVEL.

```c
// IoctlState.c
#include <wdmsec.h>
#include <ntintsafe.h>
#include "IoctlState.h"

#define IOCTL_TAG 'tSoI'

typedef struct _DEVICE_EXTENSION {
    KSPIN_LOCK Lock;
    IOCTL_STATE State;
    ULONGLONG Reads;
    ULONGLONG Writes;
    LONG Stopping;
} DEVICE_EXTENSION, *PDEVICE_EXTENSION;

static VOID IoctlUnload(PDRIVER_OBJECT DriverObject);

static NTSTATUS CompleteIrp(PIRP Irp, NTSTATUS Status, ULONG_PTR Information)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = Information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return Status;          /* never read Irp->IoStatus after completing */
}

static NTSTATUS ReturnRequiredSize(PIRP Irp, ULONG Required)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);

    if (sp->Parameters.DeviceIoControl.OutputBufferLength >= sizeof(ULONG)) {
        *(ULONG *)Irp->AssociatedIrp.SystemBuffer = Required;
        return CompleteIrp(Irp, STATUS_BUFFER_TOO_SMALL, sizeof(ULONG));
    }
    return CompleteIrp(Irp, STATUS_BUFFER_TOO_SMALL, 0);
}

static NTSTATUS IoctlCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return CompleteIrp(Irp, STATUS_SUCCESS, 0);
}

static NTSTATUS CaptureResetRequest(PIRP Irp, PIOCTL_RESET_REQUEST Request)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PVOID input = sp->Parameters.DeviceIoControl.Type3InputBuffer;

    if (Irp->RequestorMode != KernelMode) {
        if (KeGetCurrentIrql() >= DISPATCH_LEVEL || input == NULL) {
            return STATUS_INVALID_DEVICE_REQUEST;
        }
        __try {
            ProbeForRead(input, sizeof(*Request), __alignof(IOCTL_RESET_REQUEST));
            RtlCopyMemory(Request, input, sizeof(*Request));
        } __except (EXCEPTION_EXECUTE_HANDLER) {
            return GetExceptionCode();
        }
        return STATUS_SUCCESS;
    }
    if (input == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlCopyMemory(Request, input, sizeof(*Request));
    return STATUS_SUCCESS;
}

static NTSTATUS IoctlDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    ULONG code = sp->Parameters.DeviceIoControl.IoControlCode;
    ULONG inLen = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    PVOID systemBuffer = Irp->AssociatedIrp.SystemBuffer;
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR information = 0;
    KIRQL oldIrql;

    if ((code & 0xFFFF0000) != (IOCTL_DEVICE_TYPE << 16)) {
        return CompleteIrp(Irp, STATUS_INVALID_DEVICE_REQUEST, 0);
    }

    switch (code) {
    case IOCTL_GET_VERSION: {
        PIOCTL_VERSION_INFO info;

        if (outLen < sizeof(IOCTL_VERSION_INFO)) {
            return ReturnRequiredSize(Irp, sizeof(IOCTL_VERSION_INFO));
        }
        info = (PIOCTL_VERSION_INFO)systemBuffer;
        info->Size = sizeof(IOCTL_VERSION_INFO);
        info->ProtocolVersion = IOCTL_PROTOCOL_VERSION;
        info->StateSize = sizeof(IOCTL_STATE);
        info->MaxStateSize = sizeof(IOCTL_STATE);
        info->Capabilities = IOCTL_FLAG_VALID_MASK;
        info->Reserved = 0;
        status = STATUS_SUCCESS;
        information = sizeof(IOCTL_VERSION_INFO);
        break;
    }
    case IOCTL_READ_STATE: {
        PIOCTL_STATE out;

        if (outLen < sizeof(IOCTL_STATE)) {
            return ReturnRequiredSize(Irp, sizeof(IOCTL_STATE));
        }
        out = (PIOCTL_STATE)systemBuffer;
        KeAcquireSpinLock(&dx->Lock, &oldIrql);
        *out = dx->State;
        dx->Reads++;
        KeReleaseSpinLock(&dx->Lock, oldIrql);
        out->Size = sizeof(IOCTL_STATE);
        out->Version = IOCTL_PROTOCOL_VERSION;
        status = STATUS_SUCCESS;
        information = sizeof(IOCTL_STATE);
        break;
    }
    case IOCTL_WRITE_STATE: {
        PIOCTL_STATE in;
        ULONG labelBytes;

        if (inLen < sizeof(IOCTL_STATE)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        in = (PIOCTL_STATE)systemBuffer;
        if (in->Version != IOCTL_PROTOCOL_VERSION) {
            status = STATUS_REVISION_MISMATCH;
            break;
        }
        if (in->Size != sizeof(IOCTL_STATE) ||
            (in->Flags & ~IOCTL_FLAG_VALID_MASK) != 0 ||
            in->LabelChars >= IOCTL_LABEL_CHARS) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        if (!RtlULongMult(in->LabelChars, (ULONG)sizeof(WCHAR), &labelBytes)) {
            status = STATUS_INTEGER_OVERFLOW;
            break;
        }
        if (in->Label[in->LabelChars] != L'\0') {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        KeAcquireSpinLock(&dx->Lock, &oldIrql);
        dx->State.Counter++;
        dx->State.Flags = in->Flags;
        dx->State.LabelChars = in->LabelChars;
        RtlZeroMemory(dx->State.Label, sizeof(dx->State.Label));
        RtlCopyMemory(dx->State.Label, in->Label, labelBytes);
        dx->Writes++;
        KeReleaseSpinLock(&dx->Lock, oldIrql);
        status = STATUS_SUCCESS;
        break;
    }
    case IOCTL_RESET_STATE: {
        IOCTL_RESET_REQUEST request;

        if (inLen != 0) {
            if (inLen < sizeof(request)) {
                status = STATUS_INVALID_PARAMETER;
                break;
            }
            status = CaptureResetRequest(Irp, &request);
            if (!NT_SUCCESS(status)) {
                break;
            }
            if (request.Size != sizeof(request) ||
                request.Version != IOCTL_PROTOCOL_VERSION ||
                request.Flags != 0) {
                status = STATUS_INVALID_PARAMETER;
                break;
            }
        }
        KeAcquireSpinLock(&dx->Lock, &oldIrql);
        RtlZeroMemory(&dx->State, sizeof(dx->State));
        dx->State.Size = sizeof(IOCTL_STATE);
        dx->State.Version = IOCTL_PROTOCOL_VERSION;
        KeReleaseSpinLock(&dx->Lock, oldIrql);
        status = STATUS_SUCCESS;
        break;
    }
    default:
        break;              /* status already STATUS_INVALID_DEVICE_REQUEST */
    }
    return CompleteIrp(Irp, status, information);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(L"\\Device\\IoctlState");
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\IoctlState");
    UNICODE_STRING sddl = RTL_CONSTANT_STRING(L"D:P(A;;GA;;;SY)(A;;GA;;;BA)");
    static const GUID classGuid =
        { 0x6f2a1c88, 0x4b3d, 0x4a21, { 0x8e, 0x77, 0x1c, 0x9a, 0x44, 0x0b, 0x62, 0xd1 } };
    PDEVICE_OBJECT deviceObject = NULL;
    PDEVICE_EXTENSION dx;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    status = IoCreateDeviceSecure(DriverObject, sizeof(DEVICE_EXTENSION), &deviceName,
                                  IOCTL_DEVICE_TYPE, FILE_DEVICE_SECURE_OPEN, FALSE,
                                  &sddl, &classGuid, &deviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    dx = (PDEVICE_EXTENSION)deviceObject->DeviceExtension;
    KeInitializeSpinLock(&dx->Lock);
    dx->State.Size = sizeof(IOCTL_STATE);
    dx->State.Version = IOCTL_PROTOCOL_VERSION;

    status = IoCreateSymbolicLink(&symLink, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(deviceObject);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = IoctlCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = IoctlCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = IoctlDeviceControl;
    DriverObject->DriverUnload = IoctlUnload;
    return STATUS_SUCCESS;
}

static VOID IoctlUnload(PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT deviceObject = DriverObject->DeviceObject;
    PDEVICE_EXTENSION dx = (PDEVICE_EXTENSION)deviceObject->DeviceExtension;
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\IoctlState");

    InterlockedExchange(&dx->Stopping, 1);
    IoDeleteSymbolicLink(&symLink);
    IoDeleteDevice(deviceObject);
}
```

Client that discovers the state size and exercises every path:

```c
// IoctlClient.c
#include "IoctlState.h"
#include <stdio.h>
#include <string.h>
#include <wchar.h>

static HANDLE OpenDevice(void)
{
    return CreateFileW(L"\\\\.\\IoctlState", GENERIC_READ | GENERIC_WRITE,
                       FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, 0, NULL);
}

int wmain(void)
{
    HANDLE device;
    DWORD bytes = 0;
    DWORD error;
    IOCTL_VERSION_INFO version;
    IOCTL_STATE state;
    IOCTL_RESET_REQUEST reset;
    ULONG required = 0;

    device = OpenDevice();
    if (device == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }

    memset(&version, 0, sizeof(version));
    if (!DeviceIoControl(device, IOCTL_GET_VERSION, NULL, 0,
                         &version, sizeof(version), &bytes, NULL)) {
        printf("GET_VERSION failed: %lu\n", GetLastError());
        CloseHandle(device);
        return 1;
    }
    printf("protocol=%lu state=%lu capabilities=0x%08lX\n",
           version.ProtocolVersion, version.StateSize, version.Capabilities);

    /* Probe: four bytes are enough for the driver to report the real size. */
    if (!DeviceIoControl(device, IOCTL_READ_STATE, NULL, 0,
                         &required, sizeof(required), &bytes, NULL)) {
        error = GetLastError();
        if (error != ERROR_INSUFFICIENT_BUFFER) {
            printf("probe failed: %lu\n", error);
        }
    }
    printf("probe says READ_STATE needs %lu bytes\n", required);
    if (required != sizeof(IOCTL_STATE)) {
        required = sizeof(IOCTL_STATE);     /* fall back to the compiled-in size */
    }

    memset(&state, 0, sizeof(state));
    if (!DeviceIoControl(device, IOCTL_READ_STATE, NULL, 0,
                         &state, sizeof(state), &bytes, NULL)) {
        printf("READ_STATE failed: %lu\n", GetLastError());
    } else if (bytes != sizeof(state) || state.Size != sizeof(state)) {
        printf("READ_STATE returned %lu bytes, Size=%lu\n", bytes, state.Size);
    } else {
        printf("counter=%llu label=\"%ls\"\n", state.Counter, state.Label);
    }

    memset(&state, 0, sizeof(state));
    state.Size = sizeof(state);
    state.Version = IOCTL_PROTOCOL_VERSION;
    state.Flags = IOCTL_FLAG_PERSIST;
    wcscpy_s(state.Label, IOCTL_LABEL_CHARS, L"manual-test");
    state.LabelChars = (ULONG)wcslen(state.Label);
    if (!DeviceIoControl(device, IOCTL_WRITE_STATE, &state, sizeof(state),
                         NULL, 0, &bytes, NULL)) {
        printf("WRITE_STATE failed: %lu\n", GetLastError());
    }

    reset.Size = sizeof(reset);
    reset.Version = IOCTL_PROTOCOL_VERSION;
    reset.Flags = 0;
    if (!DeviceIoControl(device, IOCTL_RESET_STATE, &reset, sizeof(reset),
                         NULL, 0, &bytes, NULL)) {
        printf("RESET_STATE failed: %lu\n", GetLastError());
    }

    /* Negative test: a version the driver does not speak must be rejected. */
    reset.Version = 99;
    if (DeviceIoControl(device, IOCTL_RESET_STATE, &reset, sizeof(reset),
                        NULL, 0, &bytes, NULL)) {
        printf("RESET_STATE accepted version 99\n");
    } else if (GetLastError() != ERROR_INVALID_PARAMETER) {
        printf("RESET_STATE bad version returned %lu\n", GetLastError());
    }

    CloseHandle(device);
    return 0;
}
```

## Variations and advanced use

**Large output with `METHOD_OUT_DIRECT`.** Keep a small buffered header for the count and put the
payload in the MDL. Validate `Count`, cap it, then map and copy:

```c
if (Irp->MdlAddress == NULL) {
    status = STATUS_INVALID_PARAMETER;
} else {
    PVOID out = MmGetSystemAddressForMdlSafe(Irp->MdlAddress, NormalPagePriority);
    if (out == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
    } else {
        RtlCopyMemory(out, source, bytes);
        Irp->IoStatus.Information = bytes;
    }
}
```

**Tighten access at run time.** For a reset code that must stay `FILE_ANY_ACCESS` for an existing
client, keep the number and add a stricter check:

```c
status = IoValidateDeviceIoControlAccess(Irp, FILE_WRITE_ACCESS);
if (!NT_SUCCESS(status)) {
    return CompleteIrp(Irp, status, 0);     /* STATUS_ACCESS_DENIED to the caller */
}
```

**Append-only version 2.** Never edit version 1 in place. Define `IOCTL_STATE_V2` with `IOCTL_STATE
V1;` as its first member, accept `Size >= sizeof(IOCTL_STATE_V1)` and `Version <=
IOCTL_PROTOCOL_VERSION`, and zero the tail when the client sent a shorter struct.

**Kernel-only commands.** A command that must never be reachable from user mode belongs in
`IRP_MJ_INTERNAL_DEVICE_CONTROL`, which user mode cannot generate, with a `CTL_CODE` of the same
shape. Do not rely on a high function number for secrecy.

## Failure modes and diagnosis

| # | Design mistake | Failure you will see | Fix |
|---|---|---|---|
| 1 | Reads a fixed struct without comparing `InputBufferLength` to `sizeof` | Over-read of the system buffer; 0xC1 SPECIAL_POOL_DETECTED_MEMORY_CORRUPTION under verifier, otherwise silent garbage | Check the length before the cast |
| 2 | Writes the whole struct when `OutputBufferLength` is smaller | Pool overrun: 0xC2 BAD_POOL_CALLER, 0x13A KERNEL_MODE_HEAP_CORRUPTION; a correct check returns 0xC0000023 | Compare, or report the required size |
| 3 | Sets `Information` larger than the output buffer | Completion copies that many bytes into the client buffer: client-side corruption, call still "succeeds" | Information equals bytes written, capped at the buffer |
| 4 | METHOD_NEITHER without `ProbeForRead` | 0x50 PAGE_FAULT_IN_NONPAGED_AREA, 0x8E, or 0xD1; sometimes a read of another process's memory | Probe, copy once, validate the copy |
| 5 | Probes, validates, then reads the user buffer again | Double fetch; a changed length produces 0xC2 or 0x139 | Capture first, then validate |
| 6 | `FILE_ANY_ACCESS` on a state-changing code | No bugcheck: any handle holder can reset the device | `FILE_WRITE_DATA` for state changes |
| 7 | Access bits disagree with how the client opens | `DeviceIoControl` fails with 5 and 0xC0000022; the dispatch is never entered | Declare and request matching rights |
| 8 | Function below `0x800`, or a foreign device type | Collision; 0xC0000010 STATUS_INVALID_DEVICE_REQUEST, error 1 or 87 at the client | Vendor ranges plus a device-type check |
| 9 | No `Size`/`Version`, or they are ignored | 0xC000000D STATUS_INVALID_PARAMETER or silent field corruption after an upgrade | Both fields first, both validated |
| 10 | Kernel and client disagree on `sizeof` | 0xC000000D when checked, corruption when not; a 32-bit client shifts every field | Fixed-width types, no pointers, `C_ASSERT` |
| 11 | `count * elementSize` without an overflow check | Small allocation, large copy: 0xC2, 0x139, 0x50; a checked multiply returns 0xC0000095 | `RtlULongMult`, `RtlULongAdd`, absolute cap |
| 12 | Assumes a non-NULL `SystemBuffer` when both lengths are zero | 0x1E KMODE_EXCEPTION_NOT_HANDLED at PASSIVE, 0x50 at DISPATCH | Check the length before the pointer |
| 13 | Unknown code completed with `STATUS_SUCCESS` | Client reads a buffer the driver never wrote | 0xC0000010 STATUS_INVALID_DEVICE_REQUEST |
| 14 | Leaves an IOCTL pending, then unloads | 0xCE DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS, or 0x9F in a power transition; `!irpfind` shows the leftover | Remove lock, cancel, drain, delete |
| 15 | Completes a canceled request, or completes twice | Cancel/completion race: 0xC2, 0x139, 0x50; the client sees STATUS_CANCELLED (0xC0000120, error 995) and then success | One owner per IRP: whoever removes it completes it |

First commands for a suspect IOCTL: `.reload`, `!ioctldecode <code>`, `!irp <addr>`, `!devobj
<device>`, `!pool <address>`, `!verifier 0x80`, then `!analyze -v` on the dump.

Decode status codes exactly: `0xC000000D` is `STATUS_INVALID_PARAMETER`, `0xC0000023` is
`STATUS_BUFFER_TOO_SMALL`, `0xC0000010` is `STATUS_INVALID_DEVICE_REQUEST`, `0xC0000022` is
`STATUS_ACCESS_DENIED`, `0xC0000095` is `STATUS_INTEGER_OVERFLOW`, `0xC0000120` is
`STATUS_CANCELLED`, `0xC0000059` is `STATUS_REVISION_MISMATCH`.

## Safety, IRQL, and cleanup checklist

- [ ] Every buffer is length-checked on the same path that dereferences it; no cast precedes the check.
- [ ] `Information` is the number of bytes actually written and never exceeds `OutputBufferLength`.
- [ ] `METHOD_NEITHER` handlers refuse above APC_LEVEL, probe, copy once, and never re-read user memory.
- [ ] No user-mode pointer is touched at DISPATCH_LEVEL or from a DPC, timer, or completion routine.
- [ ] All size arithmetic is checked for overflow, with an absolute cap on every client-provided count.
- [ ] Structs contain no pointers, `long`, `BOOL`, `size_t`, or `HANDLE`; sizes are asserted on both sides.
- [ ] Every completion sets both `Status` and `Information`; the routine returns a saved status, not a
      field read from the completed IRP.
- [ ] Nothing touches the IRP after `IoCompleteRequest`, and nothing touches a pending IRP after
      handing it to a queue.
- [ ] The device has an SDDL DACL; `FILE_ANY_ACCESS` codes are harmless; tests run as a standard user.
- [ ] No spinlock is held while completing an IRP; state copies happen under the lock, completion outside.
- [ ] Unload deletes the symbolic link first, has no pending IOCTLs, then deletes the device.
- [ ] Pool allocations carry a unique tag and are freed on every error path.

## Verification

Build, sign, and load on a test VM with test signing enabled:

```
msbuild IoctlState.vcxproj /p:Configuration=Debug /p:Platform=x64
signtool sign /fd SHA256 /a /s My /n "Contoso Test" IoctlState.sys
sc create IoctlState type= kernel binPath= C:\drivers\IoctlState.sys
sc start IoctlState
```

Enable verification before loading, so a length bug becomes a stop code instead of silent damage:

```
verifier /standard /driver IoctlState.sys
```

Run the client from an elevated prompt, then from a standard prompt. Expected elevated output:

```
protocol=1 state=88 capabilities=0x00000001
probe says READ_STATE needs 88 bytes
counter=0 label=""
```

After the write and reset calls, a second read shows the new label and a non-zero counter. From a
standard prompt the open must fail with `open failed: 5` because of the SDDL. Confirm the layout in the
debugger:

```
kd> !ioctldecode 0x8C40A00F
    DeviceType  : 8c40
    Access      : 2 (FILE_WRITE_ACCESS)
    Function    : 803
    Method      : 3 (METHOD_NEITHER)
```

Proof the pending path is clean: `!irpfind` filtered by the device address returns nothing after the
client exits, and `sc stop IoctlState` completes without 0xCE. Proof the lengths are right: with
Driver Verifier's special pool enabled, the negative tests (one-byte output, truncated input, version
99, NULL pointer) produce status codes, not a bugcheck.
