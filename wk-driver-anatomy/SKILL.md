---
name: wk-driver-anatomy
description: Build a complete WDM software driver end to end — DriverEntry, IoCreateDevice with a device extension, symbolic link, dispatch array, per-handle FsContext in create/close, buffered IRP_MJ_WRITE, buffered configuration IOCTLs, rundown-protected unload, and every unwind path, with a complete Driver.c and a shared protocol header.
---

# WDM driver anatomy: DriverEntry to a safe unload

A WDM software driver is a lifecycle, not a function. The object manager hands you a driver object,
you publish a device and a name, you service IRPs until the service is stopped, then you dismantle
everything while requests may still be running. Get one phase wrong and you leak a device object,
hang a stop, or bugcheck on the next close. This skill builds one complete driver — **KlogSink**, a
buffered record sink — with a device extension, a symbolic link, per-handle context, a buffered write
path, four configuration IOCTLs, and an unload that refuses to free memory it cannot prove unused.

The rule that prevents most disasters: **every object created in `DriverEntry` is destroyed on the
failure branch that follows it, in reverse creation order, and unload must prove that no request is
still executing.** A half-finished initialization that returns `STATUS_SUCCESS` is worse than
refusing to load.

## When to use

- You need a production-shaped WDM driver with create, close, write, IOCTL, and a working unload.
- You are adding an extension, a symbolic link, or a dispatch array to a legacy driver.
- You must audit a `DriverEntry` for unwinding bugs, a missing `IoDeleteSymbolicLink`, or a leak.
- You need a buffered write path whose length and structure validation is provably safe.
- You need unload that drains in-flight dispatch calls (`EX_RUNDOWN_REF`) and open handles.
- You are building the kernel half; the user-mode half is `wk-driver-client-protocol`.

Do not use for KMDF (`EvtDriverDeviceAdd`), PnP stacks with `IRP_MN_*` handling, filter drivers, or
IOCTL design on its own (`wk-ioctl-design`).

## Prerequisites and related skills

- **IRQL:** every dispatch routine here runs at `PASSIVE_LEVEL` because the only requestors are
  user-mode threads. The one elevated region is `KeAcquireSpinLock` around a 160-byte ring insert.
- **Headers:** `<ntddk.h>` plus the shared `Public.h`. Use `<ntifs.h>` instead when you need
  prototypes only it declares (`PsLookupThreadByThreadId`, `PsGetProcessId`).
- **WDK:** `ExAllocatePool2` needs WDK 2004 / Windows 10 2004 or later. On older targets use
  `ExAllocatePoolWithTag(NonPagedPool, size, tag)` and zero the memory yourself.
- **Deployment:** test signing and a disposable VM; an unload bug often needs a reboot, not a rebuild.
- Siblings: `wk-driver-client-protocol` (same protocol — read it before changing any IOCTL),
  `wk-device-objects`, `wk-irp-buffering`, `wk-irp-basics`, `wk-driver-build-deploy`.

## Mental model

Four things decide everything:

- The **driver object** is owned by the I/O manager and lives as long as the image. You fill
  `MajorFunction[]` and `DriverUnload`; you never free it.
- The **device object** comes from `IoCreateDevice` with a caller-sized **device extension**, a
  non-paged block that `IoDeleteDevice` frees for you. The extension is the only home for state.
- The **symbolic link** is the only name user mode can see. No link, no `CreateFileW`.
- The **dispatch array** maps an IRP major function to a routine; entries left at their default
  complete the request with `STATUS_INVALID_DEVICE_REQUEST`, never a crash.

Creation is asymmetric, and the order is the contract:

```
DriverEntry: unload pointer -> dispatch array -> IoCreateDevice(ext size)
             -> extension init -> pool -> device flags -> IoCreateSymbolicLink
             (any failure unwinds what already exists, newest first)
Unload:      IoDeleteSymbolicLink -> wait handles -> wait dispatch calls
             -> free ring -> IoDeleteDevice
```

Each handle becomes a `FILE_OBJECT` whose `FsContext` holds the driver's per-handle state; create
allocates it and close frees it, so there is exactly one place to look when it leaks.

### Request and lifecycle sequence

```
client            I/O manager                  KlogSink
  |------------------>| IRP_MJ_CREATE              |
  |                   |--------------------------->| rundown, ctx = pool,
  |<------------------|<---------------------------|   ++OpenHandles
  | WriteFile(record) | copy user -> SystemBuffer  |
  |------------------>| IRP_MJ_WRITE               |
  |                   |--------------------------->| length + struct checks,
  |<------------------|<---------------------------|   spinlock, insert, 160
  | CloseHandle       |                            |
  |------------------>| IRP_MJ_CLOSE               |
  |                   |--------------------------->| free ctx, --OpenHandles,
  |<------------------|<---------------------------|   set NoHandles
  | sc stop --> SCM --> DriverUnload                |
                      | IoDeleteSymbolicLink, wait NoHandles (30 s)
                      | ExWaitForRundownProtectionRelease
                      | free ring; IoDeleteDevice
```

Both waits matter. `NoHandles` covers handles that outlive the service stop; the rundown covers
dispatch calls executing on another processor when unload starts. Skipping either produces `0xCE`
(`DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS`) or a fault in a dispatch routine that ran
after the image was unmapped.

**Unload policy is a decision, not a trick.** A legacy driver cannot veto an unload: once
`DriverUnload` returns, the SCM unloads the image. Choose up front: (1) no per-handle kernel state
and controlled clients; (2) per-handle state with unload blocking until the count reaches zero (this
example); (3) `DriverObject->DriverUnload = NULL`, which makes `sc stop` fail and forces a reboot to
upgrade. The third is legitimate where a stop race is unacceptable.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `IoCreateDevice` | device + extension | `PASSIVE_LEVEL` | extension is not zeroed for you |
| `IoCreateSymbolicLink` / `IoDeleteSymbolicLink` | publish / remove `\??\Name` | `PASSIVE_LEVEL` | `STATUS_OBJECT_NAME_COLLISION` if taken |
| `IoDeleteDevice` | free device + extension | `PASSIVE_LEVEL` | deferred while references exist |
| `IoGetCurrentIrpStackLocation` | your layer's parameters | any | never index `Tail.Overlay` by hand |
| `IoCompleteRequest` | finish the IRP | ≤ `DISPATCH_LEVEL` | IRP pointer is poisoned after this |
| `ExAllocatePool2` / `ExFreePoolWithTag` | ring buffer | ≤ `DISPATCH_LEVEL` | `POOL_FLAG_NON_PAGED`; tag must match |
| `ExInitializeRundownProtection` | arm the drain | `PASSIVE_LEVEL` | `EX_RUNDOWN_REF` in the extension |
| `ExAcquireRundownProtection` | may I touch the extension? | ≤ `DISPATCH_LEVEL` | `FALSE` once unload began |
| `ExWaitForRundownProtectionRelease` | wait for in-flight calls | `PASSIVE_LEVEL` | then `ExRundownCompleted` |
| `KeAcquireSpinLock` | protect the ring | any | raises to `DISPATCH_LEVEL`; save `KIRQL` |
| `KeWaitForSingleObject` | wait for handles to drain | `PASSIVE_LEVEL` | relative timeout is negative |
| `PsGetCurrentProcessId` | stamp the record | any | returns `HANDLE`; use `HandleToULong` |
| `KeQuerySystemTime` | stamp the record | any | 100 ns units since 1601-01-01 |

### The shared protocol

One contract in `Public.h`, duplicated verbatim in `wk-driver-client-protocol`; change it once and
rebuild both sides. Beyond the extension, the dispatch routines read only `FILE_OBJECT->FsContext`
and `IO_STACK_LOCATION->Parameters`.

| Element | Value |
|---|---|
| Device / link / client path | `\Device\KlogSink`, `\??\KlogSink`, `\\.\KlogSink` |
| Device type | `0x8A10` (`KLOG_DEVICE_TYPE`, vendor `0x8000`-`0xFFFF`) |
| Write payload | one `KLOG_RECORD`, exactly 160 bytes, `DO_BUFFERED_IO` |
| `SET_CONFIG` / `GET_CONFIG` | `0x8A10A004` / `0x8A106008` |
| `GET_STATS` / `RESET` | `0x8A10600C` / `0x8A10A010` |
| Structs / version | 160 / 24 / 32 bytes, `KLOG_PROTOCOL_VERSION` = 1 |

## Step-by-step

1. **Write `Public.h` first:** names, device type, four `CTL_CODE` macros, three structs inside
   `#pragma pack(push, 8)`; no pointers, no `size_t`. *Check:* `C_ASSERT(sizeof(KLOG_RECORD) == 160)`
   compiles.
2. **Create an empty WDM project, x64, `Win11` target;** this driver installs as a service (INF route:
   `wk-driver-build-deploy`). *Check:* `/W4` clean.
3. **Define `KLOG_DEVICE_EXTENSION`** with every field the dispatch routines need, plus the lock and
   both drain primitives; no globals. *Check:* nothing in it points at pageable memory.
4. **Fill `DriverEntry` in order:** unload pointer, dispatch array, `IoCreateDevice`, extension init,
   pool, flags, link. *Check:* every `NT_SUCCESS` test unwinds exactly what exists.
5. **Set `DO_BUFFERED_IO`, clear `DO_DEVICE_INITIALIZING`, before the link exists.** *Check:* a
   `CreateFileW` storm during `sc start` never returns `ERROR_NOT_READY`.
6. **Implement create/close as a pair:** create allocates the context into `FsContext` and increments
   the count; close clears, frees, decrements, signals `NoHandles` at zero. *Check:* 10,000 cycles with
   `!poolused` flat.
7. **Implement write:** exactly one record, validate the struct, copy to a local, stamp PID/TID/time,
   insert under the spinlock. *Check:* a zero-length write returns `STATUS_BUFFER_TOO_SMALL`.
8. **Implement the IOCTLs:** validate lengths before buffers; set `Information` to what you produced.
   *Check:* a 4-byte output buffer yields `STATUS_BUFFER_TOO_SMALL`.
9. **Implement unload: link → handles → rundown → ring → device,** refusing to free while any count is
   non-zero. *Check:* `sc stop` waits for an open handle, then refuses.
10. **Run verifier from the first load.** *Check:* no `0xC4`, `0xC1`, `0xC2`, `0x50` in three cycles.

## Complete example

Two files, both complete. `Public.h` is shared byte-for-byte with the user-mode client.

### Public.h

```c
/* Public.h - the contract between KlogSink and its clients.
   Include after <ntddk.h> in kernel code and after <windows.h> in user code. */
#pragma once

#define KLOG_PROTOCOL_VERSION  1
#define KLOG_DEVICE_TYPE       0x8A10                /* vendor 0x8000-0xFFFF */
#define KLOG_DEVICE_NAME       L"\\Device\\KlogSink"
#define KLOG_SYMLINK_NAME      L"\\??\\KlogSink"
#define KLOG_DOS_NAME          L"\\\\.\\KlogSink"    /* user mode only */

#define KLOG_TEXT_CHARS        64
#define KLOG_MAX_RECORDS       64
#define KLOG_MAX_SEVERITY      4
#define KLOG_CFG_DROP_ON_FULL      0x00000001
#define KLOG_CFG_OVERWRITE_OLDEST  0x00000002
#define KLOG_CFG_VALID_FLAGS       (KLOG_CFG_DROP_ON_FULL | KLOG_CFG_OVERWRITE_OLDEST)
#define KLOG_RECORD_URGENT         0x00000001
#define KLOG_RECORD_VALID_FLAGS    (KLOG_RECORD_URGENT)

#define KLOG_IOCTL_SET_CONFIG  CTL_CODE(KLOG_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_WRITE_DATA)
#define KLOG_IOCTL_GET_CONFIG  CTL_CODE(KLOG_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_READ_DATA)
#define KLOG_IOCTL_GET_STATS   CTL_CODE(KLOG_DEVICE_TYPE, 0x803, METHOD_BUFFERED, FILE_READ_DATA)
#define KLOG_IOCTL_RESET       CTL_CODE(KLOG_DEVICE_TYPE, 0x804, METHOD_BUFFERED, FILE_WRITE_DATA)

#pragma pack(push, 8)

typedef struct _KLOG_RECORD {
    ULONG    Version;                 /* KLOG_PROTOCOL_VERSION                 */
    ULONG    Size;                    /* sizeof(KLOG_RECORD), exactly 160      */
    ULONG    Severity;                /* 0..KLOG_MAX_SEVERITY                  */
    ULONG    Flags;                   /* KLOG_RECORD_*                         */
    ULONG    ProcessId;               /* overwritten by the driver             */
    ULONG    ThreadId;                /* overwritten by the driver             */
    LONGLONG Timestamp;               /* overwritten by the driver, 100 ns     */
    WCHAR    Text[KLOG_TEXT_CHARS];   /* NUL-terminated on the way in          */
} KLOG_RECORD, *PKLOG_RECORD;

typedef struct _KLOG_CONFIG {
    ULONG Version;                    /* KLOG_PROTOCOL_VERSION                 */
    ULONG Size;                       /* sizeof(KLOG_CONFIG), exactly 24       */
    ULONG Capacity;                   /* 1..KLOG_MAX_RECORDS                   */
    ULONG Flags;                      /* exactly one KLOG_CFG_* bit            */
    ULONG Reserved;                   /* must be 0                             */
    ULONG Reserved2;                  /* must be 0                             */
} KLOG_CONFIG, *PKLOG_CONFIG;

typedef struct _KLOG_STATS {
    ULONG Version;
    ULONG Size;                       /* sizeof(KLOG_STATS), exactly 32        */
    ULONG Accepted;                   /* records copied into the ring          */
    ULONG Dropped;                    /* records the policy refused            */
    ULONG Capacity;                   /* current slot count                    */
    ULONG Records;                    /* slots currently occupied              */
    ULONG OpenHandles;                /* live handles on this device           */
    ULONG MaxRecordBytes;             /* sizeof(KLOG_RECORD)                   */
} KLOG_STATS, *PKLOG_STATS;

#pragma pack(pop)
```

`#pragma pack(push, 8)` pins the layout even if a project later sets `/Zp4`, and the `C_ASSERT` lines
in `Driver.c` turn drift into a compile error instead of a silent shift that only the peer notices.
Every struct leads with `Version` and `Size` so a reader can reject a payload before reading past the
first eight bytes.

### Driver.c

```c
/* Driver.c - KlogSink: a buffered WDM software driver with per-handle context. */
#include <ntddk.h>
#include "Public.h"

#define KLOG_POOL_TAG   'golK'          /* 'K','l','o','g' in memory order */

C_ASSERT(sizeof(KLOG_RECORD) == 160);
C_ASSERT(sizeof(KLOG_CONFIG) == 24);
C_ASSERT(sizeof(KLOG_STATS) == 32);

typedef struct _KLOG_HANDLE_CONTEXT {
    ULONG ProcessId;
    ULONG Records;                      /* records accepted from this handle */
} KLOG_HANDLE_CONTEXT, *PKLOG_HANDLE_CONTEXT;

typedef struct _KLOG_DEVICE_EXTENSION {
    KSPIN_LOCK     Lock;
    KLOG_CONFIG    Config;
    PKLOG_RECORD   Ring;                /* KLOG_MAX_RECORDS slots, non-paged */
    ULONG          Head;                /* index of the oldest slot */
    ULONG          Count;               /* occupied slots */
    ULONG          Accepted;
    ULONG          Dropped;
    ULONG          Generations;
    volatile LONG  OpenHandles;
    KEVENT         NoHandles;           /* signaled while OpenHandles == 0 */
    EX_RUNDOWN_REF Rundown;
} KLOG_DEVICE_EXTENSION, *PKLOG_DEVICE_EXTENSION;

#ifdef __cplusplus
extern "C" {
#endif

DRIVER_INITIALIZE DriverEntry;
DRIVER_UNLOAD     KlogUnload;
DRIVER_DISPATCH   KlogCreate;
DRIVER_DISPATCH   KlogClose;
DRIVER_DISPATCH   KlogWrite;
DRIVER_DISPATCH   KlogDeviceControl;

#ifdef ALLOC_PRAGMA
#pragma alloc_text(INIT, DriverEntry)
#pragma alloc_text(PAGE, KlogCreate)
#pragma alloc_text(PAGE, KlogClose)
#pragma alloc_text(PAGE, KlogWrite)
#pragma alloc_text(PAGE, KlogDeviceControl)
#pragma alloc_text(PAGE, KlogUnload)
#endif

static NTSTATUS KlogComplete(PIRP Irp, PKLOG_DEVICE_EXTENSION Ext,
                             NTSTATUS Status, ULONG_PTR Info, BOOLEAN Rundown)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = Info;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    if (Rundown) {
        ExReleaseRundownProtection(&Ext->Rundown);
    }
    return Status;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING         devName = RTL_CONSTANT_STRING(KLOG_DEVICE_NAME);
    UNICODE_STRING         linkName = RTL_CONSTANT_STRING(KLOG_SYMLINK_NAME);
    PDEVICE_OBJECT         device = NULL;
    PKLOG_DEVICE_EXTENSION ext = NULL;
    NTSTATUS               status;

    UNREFERENCED_PARAMETER(RegistryPath);

    DriverObject->DriverUnload = KlogUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = KlogCreate;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = KlogClose;
    DriverObject->MajorFunction[IRP_MJ_WRITE] = KlogWrite;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = KlogDeviceControl;

    status = IoCreateDevice(DriverObject, sizeof(KLOG_DEVICE_EXTENSION), &devName,
                            KLOG_DEVICE_TYPE, FILE_DEVICE_SECURE_OPEN, FALSE, &device);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "KlogSink: dev 0x%08X\n", status);
        return status;
    }
    ext = (PKLOG_DEVICE_EXTENSION)device->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));
    KeInitializeSpinLock(&ext->Lock);
    KeInitializeEvent(&ext->NoHandles, NotificationEvent, TRUE);
    ExInitializeRundownProtection(&ext->Rundown);

    ext->Ring = (PKLOG_RECORD)ExAllocatePool2(POOL_FLAG_NON_PAGED,
        KLOG_MAX_RECORDS * sizeof(KLOG_RECORD), KLOG_POOL_TAG);
    if (ext->Ring == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Fail;
    }
    ext->Config.Version = KLOG_PROTOCOL_VERSION;
    ext->Config.Size = sizeof(KLOG_CONFIG);
    ext->Config.Capacity = 8;
    ext->Config.Flags = KLOG_CFG_DROP_ON_FULL;
    /* Flags are final before the name becomes visible to user mode. */
    device->Flags |= DO_BUFFERED_IO;
    device->Flags &= ~DO_DEVICE_INITIALIZING;

    status = IoCreateSymbolicLink(&linkName, &devName);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL, "KlogSink: link 0x%08X\n", status);
        goto Fail;
    }
    return STATUS_SUCCESS;

Fail:
    if (ext != NULL && ext->Ring != NULL) {
        ExFreePoolWithTag(ext->Ring, KLOG_POOL_TAG);
    }
    if (device != NULL) {
        IoDeleteDevice(device);         /* frees the extension as well */
    }
    return status;
}

NTSTATUS KlogCreate(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PKLOG_DEVICE_EXTENSION ext = (PKLOG_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION     sp = IoGetCurrentIrpStackLocation(Irp);
    PKLOG_HANDLE_CONTEXT   ctx;
    BOOLEAN                rundown;

    PAGED_CODE();
    rundown = ExAcquireRundownProtection(&ext->Rundown);
    if (!rundown) {
        return KlogComplete(Irp, ext, STATUS_DELETE_PENDING, 0, FALSE);
    }
    ctx = (PKLOG_HANDLE_CONTEXT)ExAllocatePool2(POOL_FLAG_NON_PAGED,
                                                sizeof(*ctx), KLOG_POOL_TAG);
    if (ctx == NULL) {
        return KlogComplete(Irp, ext, STATUS_INSUFFICIENT_RESOURCES, 0, rundown);
    }

    ctx->ProcessId = HandleToULong(PsGetCurrentProcessId());
    ctx->Records = 0;
    sp->FileObject->FsContext = ctx;    /* create always carries a FILE_OBJECT */

    if (InterlockedIncrement(&ext->OpenHandles) == 1) {
        KeClearEvent(&ext->NoHandles);
    }
    return KlogComplete(Irp, ext, STATUS_SUCCESS, 0, rundown);
}

NTSTATUS KlogClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PKLOG_DEVICE_EXTENSION ext = (PKLOG_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PFILE_OBJECT           file = IoGetCurrentIrpStackLocation(Irp)->FileObject;
    PKLOG_HANDLE_CONTEXT   ctx = (PKLOG_HANDLE_CONTEXT)file->FsContext;
    BOOLEAN                rundown;

    PAGED_CODE();

    file->FsContext = NULL;             /* a double free is now impossible */
    rundown = ExAcquireRundownProtection(&ext->Rundown);
    if (rundown && InterlockedDecrement(&ext->OpenHandles) == 0) {
        KeSetEvent(&ext->NoHandles, IO_NO_INCREMENT, FALSE);
    }
    if (ctx != NULL) {
        ExFreePoolWithTag(ctx, KLOG_POOL_TAG);
    }
    return KlogComplete(Irp, ext, STATUS_SUCCESS, 0, rundown);
}

NTSTATUS KlogWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PKLOG_DEVICE_EXTENSION ext = (PKLOG_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION     sp = IoGetCurrentIrpStackLocation(Irp);
    PKLOG_RECORD           src = (PKLOG_RECORD)Irp->AssociatedIrp.SystemBuffer;
    PKLOG_HANDLE_CONTEXT   ctx = (PKLOG_HANDLE_CONTEXT)sp->FileObject->FsContext;
    KLOG_RECORD            record;
    KIRQL                  oldIrql;
    BOOLEAN                rundown;
    ULONG                  i;

    PAGED_CODE();
    rundown = ExAcquireRundownProtection(&ext->Rundown);
    if (!rundown) {
        return KlogComplete(Irp, ext, STATUS_DELETE_PENDING, 0, FALSE);
    }

    if (sp->Parameters.Write.Length < sizeof(KLOG_RECORD)) {
        return KlogComplete(Irp, ext, STATUS_BUFFER_TOO_SMALL, 0, rundown);
    }
    if (sp->Parameters.Write.Length > sizeof(KLOG_RECORD)) {
        return KlogComplete(Irp, ext, STATUS_INVALID_BUFFER_SIZE, 0, rundown);
    }
    if (src == NULL) {
        return KlogComplete(Irp, ext, STATUS_INVALID_PARAMETER, 0, rundown);
    }
    RtlCopyMemory(&record, src, sizeof(record));

    for (i = 0; i < KLOG_TEXT_CHARS && record.Text[i] != L'\0'; ++i) {
    }
    if (record.Version != KLOG_PROTOCOL_VERSION) {
        return KlogComplete(Irp, ext, STATUS_REVISION_MISMATCH, 0, rundown);
    }
    if (record.Size != sizeof(KLOG_RECORD) || record.Severity > KLOG_MAX_SEVERITY ||
        (record.Flags & ~KLOG_RECORD_VALID_FLAGS) != 0 || i == KLOG_TEXT_CHARS) {
        return KlogComplete(Irp, ext, STATUS_INVALID_PARAMETER, 0, rundown);
    }
    /* Identity comes from the caller's token, never from the payload. */
    record.ProcessId = HandleToULong(PsGetCurrentProcessId());
    record.ThreadId = HandleToULong(PsGetCurrentThreadId());
    KeQuerySystemTime((PLARGE_INTEGER)&record.Timestamp);

    KeAcquireSpinLock(&ext->Lock, &oldIrql);
    if (ext->Count < ext->Config.Capacity) {
        ext->Ring[(ext->Head + ext->Count) % ext->Config.Capacity] = record;
        ext->Count++;
        ext->Accepted++;
        ctx->Records++;
    } else if ((ext->Config.Flags & KLOG_CFG_OVERWRITE_OLDEST) != 0) {
        ext->Ring[ext->Head] = record;
        ext->Head = (ext->Head + 1) % ext->Config.Capacity;
        ext->Accepted++;
        ext->Dropped++;
        ctx->Records++;
    } else {
        ext->Dropped++;
    }
    KeReleaseSpinLock(&ext->Lock, oldIrql);

    return KlogComplete(Irp, ext, STATUS_SUCCESS, sizeof(KLOG_RECORD), rundown);
}

NTSTATUS KlogDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PKLOG_DEVICE_EXTENSION ext = (PKLOG_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION     sp = IoGetCurrentIrpStackLocation(Irp);
    PVOID                  buffer = Irp->AssociatedIrp.SystemBuffer;
    ULONG                  inLen = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG                  outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    KIRQL                  oldIrql;
    BOOLEAN                rundown;
    NTSTATUS               status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR              info = 0;

    PAGED_CODE();
    rundown = ExAcquireRundownProtection(&ext->Rundown);
    if (!rundown) {
        return KlogComplete(Irp, ext, STATUS_DELETE_PENDING, 0, FALSE);
    }
    switch (sp->Parameters.DeviceIoControl.IoControlCode) {

    case KLOG_IOCTL_SET_CONFIG: {
        KLOG_CONFIG cfg;
        PKLOG_CONFIG in = (PKLOG_CONFIG)buffer;

        if (buffer == NULL || inLen < sizeof(cfg)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        RtlCopyMemory(&cfg, in, sizeof(cfg));   /* trailing bytes are ignored */
        if (cfg.Version != KLOG_PROTOCOL_VERSION) {
            status = STATUS_REVISION_MISMATCH;
            break;
        }
        if (cfg.Size != sizeof(cfg) || cfg.Capacity == 0 ||
            cfg.Capacity > KLOG_MAX_RECORDS ||
            (cfg.Flags & ~KLOG_CFG_VALID_FLAGS) != 0 ||
            (cfg.Flags & KLOG_CFG_VALID_FLAGS) == KLOG_CFG_VALID_FLAGS ||
            cfg.Reserved != 0 || cfg.Reserved2 != 0) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        KeAcquireSpinLock(&ext->Lock, &oldIrql);
        ext->Config = cfg;
        ext->Generations++;
        ext->Head = 0;
        ext->Count = 0;                     /* slot mapping depends on Capacity */
        KeReleaseSpinLock(&ext->Lock, oldIrql);
        status = STATUS_SUCCESS;
        break;
    }

    case KLOG_IOCTL_GET_CONFIG: {
        PKLOG_CONFIG out = (PKLOG_CONFIG)buffer;

        if (outLen < sizeof(*out)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        KeAcquireSpinLock(&ext->Lock, &oldIrql);
        *out = ext->Config;
        KeReleaseSpinLock(&ext->Lock, oldIrql);
        info = sizeof(*out);
        status = STATUS_SUCCESS;
        break;
    }

    case KLOG_IOCTL_GET_STATS: {
        PKLOG_STATS out = (PKLOG_STATS)buffer;

        if (outLen < sizeof(*out)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        KeAcquireSpinLock(&ext->Lock, &oldIrql);
        out->Version = KLOG_PROTOCOL_VERSION;
        out->Size = sizeof(*out);
        out->Accepted = ext->Accepted;
        out->Dropped = ext->Dropped;
        out->Capacity = ext->Config.Capacity;
        out->Records = ext->Count;
        out->OpenHandles = (ULONG)ext->OpenHandles;
        out->MaxRecordBytes = sizeof(KLOG_RECORD);
        KeReleaseSpinLock(&ext->Lock, oldIrql);
        info = sizeof(*out);
        status = STATUS_SUCCESS;
        break;
    }

    case KLOG_IOCTL_RESET:
        KeAcquireSpinLock(&ext->Lock, &oldIrql);
        ext->Head = 0;
        ext->Count = 0;
        ext->Accepted = 0;
        ext->Dropped = 0;
        KeReleaseSpinLock(&ext->Lock, oldIrql);
        status = STATUS_SUCCESS;
        break;

    default:
        status = STATUS_INVALID_DEVICE_REQUEST;
        break;
    }

    return KlogComplete(Irp, ext, status, info, rundown);
}

VOID KlogUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING         linkName = RTL_CONSTANT_STRING(KLOG_SYMLINK_NAME);
    PDEVICE_OBJECT         device = DriverObject->DeviceObject;
    PKLOG_DEVICE_EXTENSION ext;
    LARGE_INTEGER          timeout;

    PAGED_CODE();

    IoDeleteSymbolicLink(&linkName);    /* no new opens after this line */
    if (device == NULL) {
        return;
    }
    ext = (PKLOG_DEVICE_EXTENSION)device->DeviceExtension;

    timeout.QuadPart = -300000000LL;    /* 30 s, relative */
    KeWaitForSingleObject(&ext->NoHandles, Executive, KernelMode, FALSE, &timeout);

    if (InterlockedCompareExchange(&ext->OpenHandles, 0, 0) != 0) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "KlogSink: %ld handles, unload refused\n", ext->OpenHandles);
        return;                         /* freeing now would be worse */
    }
    ExWaitForRundownProtectionRelease(&ext->Rundown);
    if (InterlockedCompareExchange(&ext->OpenHandles, 0, 0) != 0) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "KlogSink: create raced unload\n");
        return;                         /* a late create owns a context */
    }
    ExRundownCompleted(&ext->Rundown);

    if (ext->Ring != NULL) {
        ExFreePoolWithTag(ext->Ring, KLOG_POOL_TAG);
        ext->Ring = NULL;
    }
    IoDeleteDevice(device);
}

#ifdef __cplusplus
}
#endif
```

### Block by block

- **`C_ASSERT`** pins the wire format: a size change must break the build, not a transfer.
- **`KLOG_HANDLE_CONTEXT`** is what `FsContext` points at — allocated once in create, freed once in
  close, counting the records that handle contributed.
- **`KLOG_DEVICE_EXTENSION`** holds all state; `Ring` is separate so ring lifetime is not tied to
  device lifetime, which would hide leaks.
- **`KlogComplete`** is the single exit: publishes both `IoStatus` fields, completes once, and
  releases the rundown after completion.
- **`#pragma alloc_text(PAGE, ...)`** is safe because every request arrives at `PASSIVE_LEVEL`; the
  spinlock raises IRQL only after the code is resident. If a kernel component could send an IRP at
  `DISPATCH_LEVEL`, delete these pragmas.
- **`DriverEntry`** is acquisitions with one `Fail:` label freeing the ring, then the device (which
  frees the extension); `INIT` code is discarded after load.
- **`KlogCreate`** acquires rundown before touching anything; **`KlogClose`** nulls `FsContext`
  before freeing, so a second close cannot double-free.
- **`KlogWrite`** uses `AssociatedIrp.SystemBuffer`, never `UserBuffer`, which still holds the user
  address and is unsafe to dereference.
- **`KlogDeviceControl`** checks lengths before buffers and fills the system buffer under the lock,
  so no partial update escapes.
- **`KlogUnload`** is link, handles, rundown, ring, device; each wait can refuse and free nothing.

## Variations and advanced use

- **Direct I/O for large payloads.** Swap `DO_BUFFERED_IO` for `DO_DIRECT_IO` and read
  `Irp->MdlAddress` with `MmGetSystemAddressForMdlSafe(irp->MdlAddress, NormalPagePriority |
  MdlMappingNoExecute)` after checking `MmGetMdlByteCount`; no copy, but the MDL must live until
  completion.
- **Inverted call.** Queue pending `METHOD_OUT_DIRECT` IOCTLs and complete one per event to push
  records instead of accepting them. Cancellation becomes mandatory: a client that exits leaves a
  pending IRP that unload must drain.
- **Device interface instead of a link.** `IoRegisterDeviceInterface` + `IoSetDeviceInterfaceState`
  remove the DOS namespace but need a PnP-managed device, which a service-created one is not.
- **Several device objects.** `DriverObject->DeviceObject` is a list through `NextDevice`; unload must
  walk and delete every entry, not just the head.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `sc start` fails 577 / 1275 | unsigned image, wrong architecture, test signing off | `signtool verify /pa /v KlogSink.sys`; enable test signing, reboot |
| `CreateFileW` returns 2 | driver not started, no `\??\KlogSink` | `sc query klgsink`; WinObj under `\??\`; check the `DriverEntry` trace |
| `CreateFileW` returns 5 | device security descriptor excludes the caller | test elevated; real policy needs an explicit DACL |
| write returns 122 | client sent fewer than 160 bytes | print `nNumberOfBytesToWrite` beside `sizeof(KLOG_RECORD)` |
| write returns 87 | `Severity` > 4, bad `Size`, unterminated `Text`, unknown record flag | dump the struct fields before sending |
| write returns 1306 | `Version` mismatch between client and driver | rebuild both from one `Public.h` |
| IOCTL returns 1 | function code not in the switch | compare the number with the four macros; a missing `break` hits `default` |
| IOCTL returns 122 on a GET | output buffer smaller than the struct | pass `sizeof(KLOG_CONFIG)` or `sizeof(KLOG_STATS)` |
| IOCTL returns 5 | handle lacked the access bits in `CTL_CODE` | open with `GENERIC_READ \| GENERIC_WRITE` |
| `sc stop` hangs, then 1053 | clients hold handles; unload sits in the 30 s wait | close clients; `!devobj \Device\KlogSink` shows the reference count |
| `0xCE` | a queued IRP, timer, or work item outlived unload | keep completion synchronous or drain before `IoDeleteDevice` |
| `0xD1` / `0x7E` after a new feature | pageable routine entered at `DISPATCH_LEVEL` | `!analyze -v` for the faulting address, compare with the `alloc_text` list |
| `0x50` in the write path | dereferencing `UserBuffer` instead of `SystemBuffer` | switch to `AssociatedIrp.SystemBuffer` |
| `0xC2` on close | `FsContext` freed twice | null `FsContext` before freeing; `!pool` the address |
| device present after unload, or reload fails with a name collision | a failure path skipped `IoDeleteDevice`, or the previous unload refused | `!devobj`; audit every `goto Fail`; reboot if the device is stale |

## Safety, IRQL, and cleanup checklist

- [ ] Every `NTSTATUS` is tested with `NT_SUCCESS`; no return value is discarded.
- [ ] `DO_BUFFERED_IO` set and `DO_DEVICE_INITIALIZING` cleared **before** the link exists.
- [ ] Device state lives in the extension, not in globals or statics.
- [ ] `KlogCreate` is the only allocator of `KLOG_HANDLE_CONTEXT`, `KlogClose` the only freer, and it
      nulls `FsContext` first.
- [ ] The spinlock covers the ring and every counter; nothing pageable is touched while it is held.
- [ ] `AssociatedIrp.SystemBuffer`, not `UserBuffer`; lengths validated before the buffer is touched.
- [ ] `Information` equals the bytes transferred, and the IRP is never dereferenced after completion.
- [ ] Unload order is link, handles, rundown, ring, device, with a refusal branch that frees nothing.
- [ ] The pool tag is unique, and verifier ran for three load/use/unload cycles.

## Verification

```text
msbuild KlogSink.vcxproj /p:Configuration=Release /p:Platform=x64
sc create klgsink type= kernel binPath= C:\Test\KlogSink.sys
sc start klgsink                -> STATE : 4 RUNNING
WinObj (elevated)               -> \Device\KlogSink exists, \??\KlogSink links to it
kd> !drvobj \Driver\klgsink 2   -> Device object list: \Device\KlogSink
kd> !devobj \Device\KlogSink    -> RefCount: 1, Flags: 0x100 (DO_BUFFERED_IO)
sc stop klgsink                 -> STATE : 1 STOPPED
kd> !devobj \Device\KlogSink    -> No matching devices found
verifier /standard /driver KlogSink.sys
kd> !verifier 3 klgsink         -> no Pool, IRQL, or Handles violations
kd> !poolused 4 Klog           -> no blocks after the driver stops
```

`RefCount: 1` with no clients is the driver's own reference; anything higher means a leaked
`FILE_OBJECT` or IRP. A device that survives `sc stop` was refused, and the `DbgPrintEx` line in
DebugView names the handle count.
