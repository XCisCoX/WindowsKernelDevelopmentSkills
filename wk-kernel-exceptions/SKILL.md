---
name: wk-kernel-exceptions
description: Use when kernel code dereferences a user-mode pointer, when a driver bugchecks with 0x1E, 0x3B, 0x7E or 0x8E, or when SEH meets C++ objects. Covers __try/__except/__finally, filters, GetExceptionCode, catchable versus fatal faults, ProbeForRead/ProbeForWrite, safe copy and double-fetch prevention, C2712 and /EHa versus /EHsc, ExRaiseStatus, and a full IOCTL handler.
---

# Kernel exceptions: catch only what is catchable, and copy only once

A kernel fault is not an error you can shrug off: if no kernel frame handles it, the machine bugchecks, and the user whose buffer was bad never learns why. Structured exception handling exists in the kernel for exactly one job — surviving a fault on memory the driver does not own, which in practice means user-mode buffers whose lifetime the driver cannot control. The mental model that prevents most disasters is narrow: SEH can catch a fault on a **user** address, it can never catch a fault on a **kernel** address, and a probe is a range check rather than a lock. Copy user data once into memory you own, validate the copy, and never look at the user pointer again.

## When to use

- A dispatch routine dereferences `Irp->UserBuffer`, `Irp->AssociatedIrp.SystemBuffer`, or `IrpSp->Parameters.DeviceIoControl.Type3InputBuffer`.
- Triage of `0x1E`, `0x3B`, `0x7E`, or `0x8E` whose first argument is `0xC0000005`.
- Implementing or reviewing a `METHOD_NEITHER` IOCTL, a raw write handler, or any `RequestorMode == UserMode` path.
- Deciding whether a probe is enough, whether a length can change under you, or whether an output buffer can be read-only.
- A build fails with `C2712`, or someone proposes `/EHa` for a driver translation unit.
- Deciding between returning `NTSTATUS` and raising with `ExRaiseStatus` or `RtlRaiseStatus`.
- Do not use for: choosing an IRQL (`wk-irql`), locking (`wk-spinlocks`, `wk-synchronization`), buffer-method selection (`wk-irp-buffering`), or reading the crash afterwards (`wk-crash-dump-analysis`).

## Prerequisites and related skills

- Kernel C or C++ built with `<ntddk.h>`, MSVC from Visual Studio 2022 with a matching WDK 10; x64 or ARM64.
- You must know who owns the buffer: `Irp->RequestorMode`, the IOCTL transfer method, and whether the pointer came from an MDL.
- User-memory access requires `IRQL <= APC_LEVEL`; anything that can block or fault on a non-resident user buffer requires `PASSIVE_LEVEL`. At `DISPATCH_LEVEL` a user fault is a bugcheck, not an exception.
- Sibling skills: `wk-irp-buffering` (the three transfer methods and MDL lifetime), `wk-irql` (why high IRQL makes user access fatal), `wk-kernel-memory-pools` (the kernel copy target and its tags), `wk-driver-client-protocol` (the client side of the IOCTL), `wk-crash-dump-analysis` (what the resulting `0x1E`/`0x7E` dump tells you).

## Mental model

An exception is a synchronous trap raised by an instruction or by the memory manager on the faulting thread. The processor vectors through the IDT, the kernel builds an exception record on the **current kernel stack**, and `RtlDispatchException` walks the thread's SEH registration chain from the innermost frame outward. Each registration has a filter that returns one of three values; the first filter that returns `EXCEPTION_EXECUTE_HANDLER` wins, and control resumes after that `__except` block. If the chain is exhausted, the kernel does not return to user mode with an error: it calls `KeBugCheckEx`.

```text
  driver reads *(ULONG*)UserBuffer, backing memory not resident
        |
        v
  IDT vector 14 -> nt!KiPageFault
        |
        +-- address <= user limit and fault came from kernel mode
        |      -> MmAccessFault raises STATUS_ACCESS_VIOLATION
        |         -> SEH chain is searched: your __except may handle it
        |
        +-- address is a kernel address with an invalid PTE
               -> KeBugCheckEx(0x50 or 0xD1, arguments)   never reaches SEH

  exception record (stack of the faulting frame)
        |
        v
  RtlDispatchException walks registrations, innermost first
        |
        +-- filter = 1  EXCEPTION_EXECUTE_HANDLER  -> handler runs here
        +-- filter = 0  EXCEPTION_CONTINUE_SEARCH  -> next outer frame
        +-- filter = -1 EXCEPTION_CONTINUE_EXECUTION -> retry (do not use)
        +-- no frame left -> KeBugCheckEx(0x1E / 0x3B / 0x7E / 0x8E)
```

Three consequences are worth memorising.

1. **Scope is a stack frame, not a process.** An `__except` covers the call tree below it, so a fault inside a helper you called is caught by your block — unless the helper installs its own handler first. It cannot cover another thread, a DPC running on another processor, or a fault that happens after your function returned.
2. **The dispatcher only offers the exception to kernel frames.** When the fault occurred in kernel mode, user-mode handlers in the faulting process are not consulted. That is why an unguarded user pointer takes down the machine instead of returning an error to the caller.
3. **Unwinding is not C++ unwinding.** `__finally` runs, but destructors of C++ objects in the abandoned frames do not. With `/EHsc` the compiler cannot even combine `__try` with objects that need unwinding, which is error `C2712`.

What the hardware and the memory manager will and will not turn into a catchable exception:

| Situation | Catchable | Why |
|---|---|---|
| Fault on a user address from kernel mode at `IRQL <= APC_LEVEL` | **yes** | `MmAccessFault` raises `STATUS_ACCESS_VIOLATION` for user addresses |
| Same access at `DISPATCH_LEVEL` | **no** | The memory manager cannot block to resolve it; bugcheck `0xA`/`0xD1` |
| Fault on a kernel address (freed pool, corrupt pointer, bad MDL) | **no** | Treated as a kernel defect: `0x50`, `0xD1`, `0x1E` |
| Misaligned access flagged by the processor | yes | `STATUS_DATATYPE_MISALIGNMENT`, but only when alignment checking is active |
| Paging error while reading a user buffer backed by a network or removable file | yes | `STATUS_IN_PAGE_ERROR`; the data may be truncated, so retry or fail |
| Kernel stack exhaustion | **no** | The guard fault cannot be handled on an exhausted stack: `0x7F` |
| Exception in an ISR, DPC, or timer callback | **no** | There is no user context to blame; the kernel bugchecks |
| Exception on another thread | **no** | SEH is per-thread and per-stack |
| Hardware machine check or NMI | **no** | `0x9C`, `0x80`; not exception-dispatch paths |
| `KeBugCheckEx` called deliberately | **never** | A bugcheck is not an exception |
| Integer divide by zero or overflow in your own arithmetic | yes | But it is a logic bug: fix the code, do not paper over it |
| `STATUS_BREAKPOINT` / `STATUS_SINGLE_STEP` | avoid | Owned by the debugger; catching it breaks debugging |
| A status you raised yourself with `ExRaiseStatus` inside the same `__try` | yes | Handled like any other exception |

## Key APIs and structures

| Construct | Purpose | Contract and notes |
|---|---|---|
| `__try` / `__except` | Guard a region and choose a handler | Filter expression is evaluated in the faulting context, before unwinding |
| `__try` / `__finally` | Guarantee cleanup | Runs on normal exit, `return`, `break`, `goto`, `__leave`, and on exception |
| `__leave` | Jump to the end of the `__try` body | Cheaper than `goto`; still runs the matching `__finally` |
| `GetExceptionCode()` | Exception code | Legal in the filter expression and in the handler block |
| `GetExceptionInformation()` | `LPEXCEPTION_POINTERS` (`ExceptionRecord`, `ContextRecord`) | **Filter only.** The pointers die with the filter frame, so copy what you need into locals there |
| `EXCEPTION_EXECUTE_HANDLER` | `1` | Handle here; execution continues after the block |
| `EXCEPTION_CONTINUE_SEARCH` | `0` | Pass to the next outer frame |
| `EXCEPTION_CONTINUE_EXECUTION` | `-1` | Retry the faulting instruction; **unsupported and unsafe in the kernel** |
| `ProbeForRead(Address, Length, Alignment)` | Validate a user read range | `VOID`; raises `STATUS_ACCESS_VIOLATION`; range and alignment only |
| `ProbeForWrite(Address, Length, Alignment)` | Validate a user write range | Same; does not make a read-only mapping writable, the write still faults |
| `RtlCopyMemory(Dst, Src, Len)` | Byte copy | Must sit inside `__try` when `Src` is user memory |
| `ExRaiseStatus(Status)` | Raise `Status` as an exception | Never returns; must be a failure status; noncontinuable |
| `RtlRaiseStatus(Status)` | Lower-level raise | Same effect; used inside runtime-library style code |
| `MmProbeAndLockPages(Mdl, AccessMode, Operation)` | Lock a user range into an MDL | Raises on a bad range; on raise the pages are **not** locked |
| `MmGetSystemAddressForMdlSafe(Mdl, Priority)` | Map a locked MDL | Check for `NULL`; pair with `MmUnlockPages` |
| `KeBugCheckEx(Code, P1..P4)` | Deliberate crash | Not catchable; the last resort for detected corruption |

Exception codes a driver actually sees:

| Code | Name | Usual cause |
|---|---|---|
| `0xC0000005` | `STATUS_ACCESS_VIOLATION` | Buffer freed, never committed, or above the user limit |
| `0x80000002` | `STATUS_DATATYPE_MISALIGNMENT` | Pointer alignment does not match the access width |
| `0xC0000006` | `STATUS_IN_PAGE_ERROR` | Paging read failed; buffer content is not trustworthy |
| `0xC00000FD` | `STATUS_STACK_OVERFLOW` | Kernel stack exhausted; not recoverable in practice |
| `0xC0000094` | `STATUS_INTEGER_DIVIDE_BY_ZERO` | Unvalidated divisor, often from user data |

## Step-by-step

1. **Classify the buffer before writing any access.** Read `Irp->RequestorMode`. If it is `KernelMode`, the caller is another kernel component: the pointer is a kernel pointer, probing it is wrong, and a fault is not catchable. If it is `UserMode`, continue.
2. **Classify the transfer method.** `METHOD_BUFFERED` gives you a private non-paged copy in `Irp->AssociatedIrp.SystemBuffer` — no probe, no SEH, no double fetch. `METHOD_IN_DIRECT`/`METHOD_OUT_DIRECT` give you a locked MDL the I/O manager already validated — use `MmGetSystemAddressForMdlSafe`, again no probe. Only `METHOD_NEITHER`, or a driver that deliberately takes a raw pointer, needs the rest of this procedure. Check: is the method really `NEITHER`?
3. **Check the IRQL.** `KeGetCurrentIrql() <= APC_LEVEL` is required to touch user memory at all; use `PASSIVE_LEVEL` if the access can block. Return `STATUS_INVALID_DEVICE_STATE` rather than faulting. Check: no `__try` anywhere in a routine that can run at `DISPATCH_LEVEL`.
4. **Validate lengths as integers, not as pointers.** Check the input length against `sizeof(Header)`, cap it at a documented maximum, and reject `OutputBufferLength < sizeof(Output)` with `STATUS_BUFFER_TOO_SMALL`. Check: every subtraction is guarded so it cannot underflow.
5. **Copy the whole input once into a kernel buffer** sized from the captured length. Do the probe and the copy in the same `__try`, so that a buffer that disappears between the two is still caught. Check: the destination is non-paged pool you own, allocated with a tag.
6. **Validate the kernel copy, never the user buffer.** Magic, version, and every length field are read from your copy. All subsequent work uses the copy. Check: grep the routine for a second read of the user pointer.
7. **Write output through a second guarded copy.** Fill a local output structure, `ProbeForWrite`, copy, and map a fault to `STATUS_ACCESS_VIOLATION` with `Information = 0`. Check: a read-only output buffer must produce an error, not a crash.
8. **Unwind exactly once.** Free the pool, dereference objects, set `Irp->IoStatus`, and call `IoCompleteRequest` on every path, including both exception handlers. Check: one completion, one free, no path that returns without completing.

## Complete example

A driver that accepts a `METHOD_NEITHER` request, copies the caller's buffer once, validates the copy, and returns a checksum. The header is shared with the client.

```c
/* safecopy.h - include after <ntddk.h> (driver) or <windows.h> (client). */
#pragma once

#define SAFECOPY_DEVICE_NAME  L"\\Device\\SafeCopy"
#define SAFECOPY_SYMLINK_NAME L"\\DosDevices\\SafeCopy"
#define SAFECOPY_WIN32_NAME   L"\\\\.\\SafeCopy"

#define SAFECOPY_MAGIC       0x53434659u   /* 'YFCS' in memory order */
#define SAFECOPY_VERSION     1u
#define SAFECOPY_MAX_INPUT   (64u * 1024u)

typedef struct _SAFECOPY_INPUT {
    ULONG Magic;
    ULONG Version;
    ULONG PayloadLength;
    ULONG Reserved;
} SAFECOPY_INPUT;

typedef struct _SAFECOPY_OUTPUT {
    ULONG Checksum;
    ULONG BytesAccepted;
    ULONG Version;
    ULONG Reserved;
} SAFECOPY_OUTPUT;

#define IOCTL_SAFECOPY_CHECKSUM \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_NEITHER, FILE_ANY_ACCESS)
```

```c
/* safecopy.c - the driver, non-paged throughout so any dispatch context works. */
#include <ntddk.h>
#include "safecopy.h"

#define SAFECOPY_POOL_TAG 'ycaS'   /* appears as "Sacy" in !poolused */

typedef struct _SAFECOPY_EXTENSION {
    volatile LONG Requests;
    volatile LONG Faults;
} SAFECOPY_EXTENSION, *PSAFECOPY_EXTENSION;

static
NTSTATUS
SafeCopyFromUser(
    _Out_writes_bytes_(Length) PVOID KernelBuffer,
    _In_ const VOID *UserBuffer,
    _In_ SIZE_T Length,
    _In_ ULONG Alignment)
{
    if (Length == 0) {
        return STATUS_SUCCESS;
    }
    if (UserBuffer == NULL || KernelBuffer == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    __try {
        ProbeForRead(UserBuffer, Length, Alignment);
        RtlCopyMemory(KernelBuffer, UserBuffer, Length);
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        /* GetExceptionCode is legal here; the record is still live. */
        return GetExceptionCode();
    }

    return STATUS_SUCCESS;
}

static
NTSTATUS
SafeCopyToUser(
    _In_ const VOID *KernelBuffer,
    _Out_writes_bytes_(Length) PVOID UserBuffer,
    _In_ SIZE_T Length,
    _In_ ULONG Alignment)
{
    if (Length == 0) {
        return STATUS_SUCCESS;
    }
    if (UserBuffer == NULL || KernelBuffer == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    __try {
        ProbeForWrite(UserBuffer, Length, Alignment);
        RtlCopyMemory(UserBuffer, KernelBuffer, Length);
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        return GetExceptionCode();
    }

    return STATUS_SUCCESS;
}

static
NTSTATUS
SafeCopyCreateClose(
    _In_ PDEVICE_OBJECT DeviceObject,
    _Inout_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static
NTSTATUS
SafeCopyControl(
    _In_ PDEVICE_OBJECT DeviceObject,
    _Inout_ PIRP Irp)
{
    PSAFECOPY_EXTENSION ext = (PSAFECOPY_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    ULONG code = stack->Parameters.DeviceIoControl.IoControlCode;
    ULONG inLength = stack->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outLength = stack->Parameters.DeviceIoControl.OutputBufferLength;
    PVOID userIn = stack->Parameters.DeviceIoControl.Type3InputBuffer;
    PVOID userOut = Irp->UserBuffer;
    BOOLEAN fromUser = (Irp->RequestorMode == UserMode);
    PSAFECOPY_INPUT input = NULL;
    PUCHAR payload;
    SAFECOPY_OUTPUT output;
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG info = 0;
    ULONG index;

    if (code != IOCTL_SAFECOPY_CHECKSUM) {
        goto done;
    }

    if (userIn == NULL || inLength < sizeof(SAFECOPY_INPUT) ||
        inLength > SAFECOPY_MAX_INPUT) {
        status = STATUS_INVALID_PARAMETER;
        goto done;
    }
    if (outLength < sizeof(SAFECOPY_OUTPUT) || userOut == NULL) {
        status = STATUS_BUFFER_TOO_SMALL;
        goto done;
    }

    /* ExAllocatePool2 zeroes the buffer unless POOL_FLAG_UNINITIALIZED. */
    input = (PSAFECOPY_INPUT)ExAllocatePool2(POOL_FLAG_NON_PAGED,
                                             inLength, SAFECOPY_POOL_TAG);
    if (input == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto done;
    }

    if (fromUser) {
        status = SafeCopyFromUser(input, userIn, inLength, sizeof(ULONG));
        if (!NT_SUCCESS(status)) {
            InterlockedIncrement(&ext->Faults);
            goto done;
        }
    } else {
        /* A kernel requestor owns its pointer; probing it would be wrong. */
        RtlCopyMemory(input, userIn, inLength);
    }

    /* From here on the user buffer is never touched again: the kernel copy is
     * the only source of truth, so a racing writer cannot change a length. */
    if (input->Magic != SAFECOPY_MAGIC || input->Version != SAFECOPY_VERSION) {
        status = STATUS_INVALID_PARAMETER;
        goto done;
    }
    if (input->PayloadLength > inLength - sizeof(SAFECOPY_INPUT)) {
        status = STATUS_INVALID_PARAMETER;
        goto done;
    }

    RtlZeroMemory(&output, sizeof(output));
    output.Version = SAFECOPY_VERSION;
    output.BytesAccepted = input->PayloadLength;
    output.Checksum = 0;
    payload = (PUCHAR)input + sizeof(SAFECOPY_INPUT);
    for (index = 0; index < input->PayloadLength; index++) {
        output.Checksum = (output.Checksum << 5) - output.Checksum
                        + payload[index];
    }

    if (fromUser) {
        status = SafeCopyToUser(&output, userOut, sizeof(output),
                                sizeof(ULONG));
        if (!NT_SUCCESS(status)) {
            InterlockedIncrement(&ext->Faults);
            goto done;
        }
    } else {
        RtlCopyMemory(userOut, &output, sizeof(output));
    }

    InterlockedIncrement(&ext->Requests);
    info = sizeof(output);
    status = STATUS_SUCCESS;

done:
    if (input != NULL) {
        ExFreePoolWithTag(input, SAFECOPY_POOL_TAG);
    }
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = info;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static
VOID
SafeCopyUnload(
    _In_ PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING link;

    RtlInitUnicodeString(&link, SAFECOPY_SYMLINK_NAME);
    IoDeleteSymbolicLink(&link);
    IoDeleteDevice(DriverObject->DeviceObject);
}

NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT device = NULL;
    PSAFECOPY_EXTENSION ext;
    UNICODE_STRING name;
    UNICODE_STRING link;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&name, SAFECOPY_DEVICE_NAME);
    status = IoCreateDevice(DriverObject, sizeof(SAFECOPY_EXTENSION), &name,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PSAFECOPY_EXTENSION)device->DeviceExtension;
    ext->Requests = 0;
    ext->Faults = 0;

    RtlInitUnicodeString(&link, SAFECOPY_SYMLINK_NAME);
    status = IoCreateSymbolicLink(&link, &name);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = SafeCopyCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = SafeCopyCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = SafeCopyControl;
    DriverObject->DriverUnload = SafeCopyUnload;

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

```c
/* client.c - exercises the boundary, including the paths that must fail. */
#include <windows.h>
#include <stdio.h>
#include "safecopy.h"

static void run_case(HANDLE device, PUCHAR buffer, DWORD inLength,
                     PUCHAR out, DWORD outLength, const char *label)
{
    DWORD returned = 0;
    BOOL ok = DeviceIoControl(device, IOCTL_SAFECOPY_CHECKSUM,
                              buffer, inLength, out, outLength,
                              &returned, NULL);
    printf("%-22s ok=%d err=%lu returned=%lu\n",
           label, ok, ok ? 0UL : GetLastError(), returned);
}

int main(void)
{
    HANDLE device;
    UCHAR input[sizeof(SAFECOPY_INPUT) + 16];
    SAFECOPY_INPUT *header = (SAFECOPY_INPUT *)input;
    SAFECOPY_OUTPUT output;
    UCHAR tiny[2];
    int i;

    device = CreateFileW(SAFECOPY_WIN32_NAME, GENERIC_READ | GENERIC_WRITE,
                         0, NULL, OPEN_EXISTING, 0, NULL);
    if (device == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }

    header->Magic = SAFECOPY_MAGIC;
    header->Version = SAFECOPY_VERSION;
    header->PayloadLength = 16;
    header->Reserved = 0;
    for (i = 0; i < 16; i++) {
        input[sizeof(SAFECOPY_INPUT) + i] = (UCHAR)i;
    }

    run_case(device, input, sizeof(input), (PUCHAR)&output, sizeof(output),
             "valid");
    run_case(device, input, sizeof(input), tiny, sizeof(tiny),
             "output too small");
    run_case(device, input, 4, (PUCHAR)&output, sizeof(output),
             "input too small");

    header->PayloadLength = 4096;   /* longer than the buffer that follows */
    run_case(device, input, sizeof(input), (PUCHAR)&output, sizeof(output),
             "payload lies");
    header->PayloadLength = 16;

    CloseHandle(device);
    return 0;
}
```

Expected client output on a healthy driver, in order: `valid` prints `ok=1` and a `returned` of 16; `output too small` prints `ok=0 err=122`; `input too small` prints `ok=0 err=87`; `payload lies` prints `ok=0 err=87`. A crash instead of `err=122` means the output probe is missing.

## Variations and advanced use

### `__try`/`__finally` for cleanup, and where RAII replaces it

`__finally` is the only construct that runs when an exception abandons a frame, so the allocation-plus-lock pattern belongs in it:

```c
/* Mutex is initialized by the caller; the guarded region owns the release. */
static
NTSTATUS
ReadOneChunk(
    _Inout_ FAST_MUTEX *Mutex,
    _In_ const VOID *UserBuffer,
    _In_ ULONG Length)
{
    NTSTATUS status = STATUS_SUCCESS;
    PVOID buffer = NULL;
    BOOLEAN held = FALSE;

    if (Length > 4096) {
        return STATUS_INVALID_PARAMETER;
    }
    buffer = ExAllocatePool2(POOL_FLAG_NON_PAGED, 4096, SAFECOPY_POOL_TAG);
    if (buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    ExAcquireFastMutex(Mutex);          /* raises to APC_LEVEL */
    __try {
        held = TRUE;
        if (Length == 0) {
            status = STATUS_INVALID_PARAMETER;
            __leave;                    /* jump to the end, run __finally */
        }
        status = SafeCopyFromUser(buffer, UserBuffer, Length, sizeof(ULONG));
    }
    __finally {
        if (held) {
            ExReleaseFastMutex(Mutex);
        }
        ExFreePoolWithTag(buffer, SAFECOPY_POOL_TAG);
    }
    return status;
}
```

RAII is cleaner for the normal path but is **not** a substitute for `__finally` across a possible raise, because destructors do not run during SEH unwinding:

```cpp
template <typename T>
class kernel_ptr {
public:
    explicit kernel_ptr(T *p = nullptr) noexcept : m_p(p) {}
    ~kernel_ptr() { if (m_p != nullptr) { ExFreePoolWithTag(m_p, SAFECOPY_POOL_TAG); } }
    kernel_ptr(const kernel_ptr &) = delete;
    kernel_ptr &operator=(const kernel_ptr &) = delete;
    T *get() const noexcept { return m_p; }
    T *operator->() const noexcept { return m_p; }
    explicit operator bool() const noexcept { return m_p != nullptr; }
private:
    T *m_p;
};

class object_ref {
public:
    explicit object_ref(PVOID p = nullptr) noexcept : m_p(p) {}
    ~object_ref() { if (m_p != nullptr) { ObDereferenceObject(m_p); } }
    object_ref(const object_ref &) = delete;
    object_ref &operator=(const object_ref &) = delete;
    PVOID get() const noexcept { return m_p; }
private:
    PVOID m_p;
};
```

Use `object_ref` for anything returned by `PsLookupThreadByThreadId`, `ObReferenceObjectByHandle`, or `IoGetDeviceObjectPointer`; every early return then releases the reference. Keep the SEH itself in a small C helper that owns no objects.

### `C2712`, `/EHsc`, and `/EHa`

| Property | `/EHsc` | `/EHa` |
|---|---|---|
| C++ `try`/`catch` for `throw` | yes | yes |
| `catch (...)` catches an access violation | no | **yes** |
| `__try` allowed alongside unwinding objects | no — `C2712` | yes |
| Code size, inlining, optimization | better | worse |
| Kernel default recommendation | yes | only for a file that truly needs both |

`C2712: Cannot use __try in functions that require object unwinding` means the compiler must emit both SEH frame registrations and C++ unwind tables for one function and cannot. Fix it structurally: move the guarded copy into a file-scope C function that uses only plain data, and let the C++ caller keep its objects. Compiling the translation unit with `/EHa` also silences it, at the cost of letting `catch (...)` swallow faults — which in the kernel hides exactly the corruption you need to see. Never let a C++ exception cross a kernel frame boundary; there is no runtime to unwind it.

### Raising instead of returning

```c
static NTSTATUS ValidateRequest(_In_ ULONG Length)
{
    if (Length > SAFECOPY_MAX_INPUT) {
        ExRaiseStatus(STATUS_INVALID_PARAMETER);   /* does not return */
    }
    return STATUS_SUCCESS;
}
```

Both `ExRaiseStatus` and `RtlRaiseStatus` build an exception record whose code is the status you passed, mark it noncontinuable, and transfer control to the nearest matching handler. `GetExceptionCode()` in the catcher sees the same value. A filter that needs the faulting address reads `GetExceptionInformation()->ExceptionRecord->ExceptionInformation[1]`, and must do so inside the filter expression, never in the handler body. Use a raise only where the caller is already inside a `__try`: a raise that nobody catches becomes `0x1E`, `0x3B`, `0x7E`, or `0x8E` depending on the thread, and the machine dies. Never pass a success status, and never raise from a routine that can run at `DISPATCH_LEVEL` unless the handler is also safe there — unwinding does not lower IRQL.

### Locking an MDL when you must have direct access

When a `METHOD_NEITHER` request needs repeated access rather than one copy, map it once:

```c
/* Map a raw user range for repeated access. On success the caller owns the
 * MDL and must eventually call MmUnlockPages then IoFreeMdl. */
static
NTSTATUS
MapUserBuffer(
    _In_ PVOID UserBuffer,
    _In_ ULONG Length,
    _Outptr_ PVOID *Mapped,
    _Out_ PMDL *MdlOut)
{
    PMDL mdl;
    BOOLEAN locked = FALSE;
    NTSTATUS status = STATUS_SUCCESS;
    PVOID mapped;

    *Mapped = NULL;
    *MdlOut = NULL;

    if (UserBuffer == NULL || Length == 0) {
        return STATUS_INVALID_PARAMETER;
    }
    mdl = IoAllocateMdl(UserBuffer, Length, FALSE, FALSE, NULL);
    if (mdl == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    __try {
        MmProbeAndLockPages(mdl, UserMode, IoReadAccess);
        locked = TRUE;
    }
    __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
    }

    if (!locked) {
        /* The lock raised, so nothing was locked: free the MDL and stop. */
        IoFreeMdl(mdl);
        return status;
    }

    mapped = MmGetSystemAddressForMdlSafe(mdl, NormalPagePriority);
    if (mapped == NULL) {
        MmUnlockPages(mdl);
        IoFreeMdl(mdl);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    *Mapped = mapped;
    *MdlOut = mdl;
    return STATUS_SUCCESS;
}
```

The `locked` flag is the whole point: `MmProbeAndLockPages` raises before locking anything, so the failure path frees the MDL and must not unlock it. `MmUnlockPages` on an unlocked MDL is a bugcheck, not an error return.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `0x1E` with arg1 `0xC0000005`, driver frame near the top | Raw user pointer dereferenced outside any `__try` | `!analyze -v`, `kv`; wrap the access and copy once |
| `0x7E` / `0x8E` with `0xC0000005` at `PASSIVE_LEVEL` | Same fault reached from a system thread, so the process could not be blamed | `!analyze -v`, then `!thread`; find the buffer's origin |
| `0xD1` arg2 = `2`, arg1 a user address | User buffer touched from a DPC or at `DISPATCH_LEVEL` | `!irp`; defer the access to `PASSIVE_LEVEL` via a work item |
| `0x50` or `0xD1` with a kernel address | Freed pool, stale object, or corrupt pointer — **not** catchable | `!pool <addr>`, special pool; fix ownership |
| `0x1E` raised right after `ExRaiseStatus` | The raise had no handler in the current stack | Add the `__try`, or return the status instead |
| `error C2712` at build | `__try` in a function with objects that need unwinding under `/EHsc` | Move SEH into a plain C helper, or compile that file `/EHa` |
| Probe passes, copy still faults | Probe is a range check, not a lock; the memory changed | Keep probe and copy in one `__try` |
| Checksum or length differs between two runs with the same input | Double fetch: the user value was read twice | Copy once, validate the copy, never re-read the pointer |
| Client gets error 998 (`ERROR_NOACCESS`) instead of a status | Write to a read-only output mapping was caught | Report it, do not retry in a loop |
| A fault is "handled" but memory is corrupt minutes later | A real bug was swallowed by a broad handler | Narrow the filter; never catch `EXCEPTION_EXECUTE_HANDLER` around code that should not fault |
| `__finally` did not free something | Early `return` placed outside the guarded region, or cleanup duplicated on the success path | One guarded region, one free, `__leave` for early exits |

## Safety, IRQL, and cleanup checklist

- [ ] Every user pointer is reached through a single documented path, and `RequestorMode` is checked before probing.
- [ ] `ProbeForRead`/`ProbeForWrite` and the copy that follows are inside the **same** `__try`.
- [ ] The user buffer is copied exactly once; all validation reads your kernel copy.
- [ ] Alignment passed to the probe matches the widest field you will read.
- [ ] `METHOD_BUFFERED` and `METHOD_*_DIRECT` paths do no probing, because the I/O manager already did it.
- [ ] No user-memory access at `DISPATCH_LEVEL`, in a DPC, in an ISR, in a completion routine, or in a timer callback.
- [ ] Every `__finally` is straight-line code with no `return`, no raise, and no blocking call.
- [ ] Pool allocated before the `__try` is freed after it, on both the success and the exception path, exactly once.
- [ ] The IRP is completed once per request, with `Information` = 0 on a copy failure.
- [ ] `MmProbeAndLockPages` failures free the MDL and skip `MmUnlockPages`.
- [ ] The filter is as narrow as the fault classes you actually expect; a blanket `EXCEPTION_EXECUTE_HANDLER` around complex code hides corruption.

## Verification

1. Build with `/W4`: `msbuild safecopy.vcxproj /p:Configuration=Debug /p:Platform=x64`, sign with the test certificate, install and start (`wk-driver-build-deploy`).
2. Run the client. Expected: `valid ok=1 returned=16`, `output too small ok=0 err=122`, `input too small ok=0 err=87`, `payload lies ok=0 err=87`. Any other result means a length check or a probe is misplaced.
3. Add a hostile case to the client: pass a read-only output buffer with `VirtualProtect(out, sizeof(out), PAGE_READONLY, &old)`. Expected: `ok=0 err=998`, no bugcheck — the write fault was caught and converted.
4. Enable special pool so a freed buffer faults deterministically instead of silently: `verifier /flags 0x1 /driver safecopy.sys`, reboot, rerun the client. Expected: no `0xC1`/`0xD6`.
5. Break in the debugger (`wk-kernel-debugging`) on `bp safecopy!SafeCopyControl`, then `!irp @rdx` to confirm the request is `METHOD_NEITHER` and inspect `Type3InputBuffer`. Step to `SafeCopyFromUser` and confirm `ProbeForRead` runs before `RtlCopyMemory`.
6. Prove the copy-once property: run a second thread that rewrites `PayloadLength` in the input buffer in a tight loop while 100000 IOCTLs run. The checksum must always match the payload length that was copied; with a double-fetch design this test yields a fault or a mismatched checksum.
7. Check for leaks: `!poolused 2` before and after the loop. The `Sacy` tag must return to its baseline count and byte total; a growing count is an unwind path that skips the free.
