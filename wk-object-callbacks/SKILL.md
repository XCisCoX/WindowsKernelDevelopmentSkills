---
name: wk-object-callbacks
description: Use when a driver must observe or restrict handle opens and duplicates for process, thread, or desktop objects - ObRegisterCallbacks and ObUnRegisterCallbacks, OB_CALLBACK_REGISTRATION and OB_OPERATION_REGISTRATION setup, altitude and OB_FLT_REGISTRATION_VERSION, OB_PRE_OPERATION_INFORMATION DesiredAccess stripping, OB_POST_OPERATION_INFORMATION auditing, KernelHandle filtering, protected-PID lists, and unload ordering.
---

# Object callbacks: intercepting handle opens and duplicates

A driver registers a pre-operation and a post-operation callback for a few kernel object types, and from then on every handle open or duplicate for those types passes through the driver's code before and after the object manager finishes it. This is the supported mechanism behind process-protection, anti-cheat, and endpoint-security products, and the only sanctioned way to change the access mask a caller receives. The mental model that keeps the code correct: the pre-operation callback cannot fail an open, it can only shrink the access mask, so the caller always gets a handle and discovers the restriction later when the missing right is needed. The rule that prevents most disasters: unregister and drain the callbacks before tearing down any state they read, and strip only the rights the protection genuinely needs, because every removed right changes the behavior of unrelated software.

## When to use

- A handle opened or duplicated to a process must lose a specific right, such as `PROCESS_TERMINATE` or `PROCESS_VM_WRITE`.
- Every handle obtained to a protected process must be audited, counted, or attributed to its requester.
- Thread handles need the same treatment: `THREAD_SET_CONTEXT`, `THREAD_SUSPEND_RESUME`, `THREAD_TERMINATE`.
- A product must veto remote-thread creation and process-memory writes without a file-system filter or a notification-only design.
- Desktop object access must be filtered (Windows 10 and later), for example `DESKTOP_SWITCHDESKTOP`.
- A protected process can still be killed and the reason must be traced, or a handle arrives with fewer rights than requested.
- A driver already observes process creation and additionally needs handle-level control.

Do not use for: observing process creation and exit without touching handles (`wk-process-notifications`); filtering file-system operations, which object callbacks cannot do (`wk-minifilter-basics`); registry access control (`wk-registry-callbacks`); object naming and handle lifetime (`wk-object-manager`); the user-mode side of the IOCTL protocol (`wk-driver-client-protocol`).

## Prerequisites and related skills

- Kernel C, WDK 10.0.22621 or later, x64. `<ntddk.h>` supplies `ObRegisterCallbacks`, the `OB_*` structures, `PsProcessType`, `PsThreadType`, and `PsGetProcessId`.
- The image must be linked with **`/INTEGRITYCHECK`**. Without it `ObRegisterCallbacks` returns `STATUS_ACCESS_DENIED` (`0xC0000022`) even when the driver is properly signed.
- Registration, callbacks, and unregistration all run at **PASSIVE_LEVEL**, in the context of the thread that requested the handle, with no user-mode access.
- Registration happens in `DriverEntry`; unregistration is the first teardown action in unload.
- Related skills, and why: `wk-object-manager` for handle and reference semantics; `wk-process-notifications` for the cheaper process-lifetime alternative; `wk-synchronization` for push-lock versus `ERESOURCE` choice; `wk-driver-anatomy` for the device, symbol link, and dispatch skeleton; `wk-driver-client-protocol` for the client wrapper.

## Mental model

### Where the callbacks sit

The object manager creates a handle in two stages: it resolves the object and computes the requested access, then it runs the security check and installs the handle. The pre-operation callbacks run between those stages; the post-operation callbacks run after the handle exists.

```
user thread: OpenProcess(PROCESS_TERMINATE | SYNCHRONIZE, FALSE, pid)
   |
   v  NtOpenProcess, PreviousMode = UserMode
ObOpenObjectByName / ObpCreateHandle
   |
   +--> OB pre-operation callbacks, highest altitude first
   |      OB_PRE_OPERATION_INFORMATION
   |        Operation    = OB_OPERATION_HANDLE_CREATE
   |        Object       = EPROCESS of the target
   |        KernelHandle = 0
   |        Parameters->CreateHandleInformation
   |            OriginalDesiredAccess  (read-only: what the caller asked for)
   |            DesiredAccess          (read-write: what will be checked)
   |        CallContext                (write-only here, delivered to the post-op)
   |      legal return value: OB_PREOP_SUCCESS, nothing else
   |
   +--> access check against the object's security descriptor
   |      using the possibly reduced DesiredAccess
   |
   +--> OB post-operation callbacks
   |      OB_POST_OPERATION_INFORMATION
   |        ReturnStatus = final NTSTATUS, GrantedAccess = installed mask
   |
   v
handle installed in the caller's table, access mask possibly smaller
```

### The pre-operation callback cannot say no

`OB_PREOP_CALLBACK_STATUS` has exactly one value, `OB_PREOP_SUCCESS`; there is no deny. The only lever is `DesiredAccess`:

- Removing bits lowers what the caller can do. If the remaining mask passes the security check, the open succeeds.
- Removing every bit yields a valid handle with a **zero access mask**. The open still succeeds.
- Adding bits is ignored; the object manager never grants more than the caller requested.
- The denial surfaces later, at the caller's first API that needs the removed right: `TerminateProcess` returns `ERROR_ACCESS_DENIED`, `CreateRemoteThread` fails. Task Manager reports "Access is denied" when End task is clicked, not when the process is opened.

A design that expects `OpenProcess` itself to fail is wrong, and a test that asserts the open fails will fail.

### The post-operation callback can only observe

`OB_POST_OPERATION_INFORMATION` reports `ReturnStatus` and the `GrantedAccess` that was installed. Nothing there is writable and the callback returns `VOID`. Post-ops are for accounting and for detecting that a caller got rights through a path the pre-op did not cover, such as duplicating a handle opened before protection started.

### Handles opened earlier are already out

Protection applies from the moment of registration. A handle opened before it keeps full access. Protection does not stop a process from terminating itself through `ExitProcess` (no new handle is created), does not stop a kernel handle, and does not stop `DuplicateHandle` from a handle that already carries the right. A real design also answers "what about handles that pre-date registration": register before the protected process starts, or close the offending handles at registration time.

### Kernel handles are a separate population

`OB_PRE_OPERATION_INFORMATION::KernelHandle` is set when the handle goes into the system handle table (`OBJ_KERNEL_HANDLE`, or any kernel-mode open). Stripping rights there breaks the memory manager, dump writers, and debuggers. Every production protector returns `OB_PREOP_SUCCESS` immediately for kernel handles.

## Key APIs and structures

| API / structure | Purpose | IRQL / context | Notes |
|---|---|---|---|
| `ObRegisterCallbacks` | register one or more operation registrations | PASSIVE_LEVEL | returns an opaque `PVOID`; `/INTEGRITYCHECK` required |
| `ObUnRegisterCallbacks` | unregister | PASSIVE_LEVEL | returns `VOID`; waits for in-flight callbacks |
| `ObGetFilterVersion` | highest supported callback version | any | equals `OB_FLT_REGISTRATION_VERSION` (`0x0100`) on supported releases |
| `OB_CALLBACK_REGISTRATION` | version, count, altitude, context, array | PASSIVE_LEVEL | one per driver |
| `OB_OPERATION_REGISTRATION` | object type plus its two callbacks | PASSIVE_LEVEL | one per object type |
| `OB_PRE_OPERATION_INFORMATION` | pre-op view | PASSIVE_LEVEL | `Object`, `KernelHandle`, `Parameters`, `CallContext` |
| `OB_POST_OPERATION_INFORMATION` | post-op view | PASSIVE_LEVEL | `ReturnStatus`, `GrantedAccess`, `CallContext` |
| `PsProcessType`, `PsThreadType` | object type pointers | n/a | pass the exported pointer itself |
| `ExDesktopObjectType` | desktop object type | n/a | Windows 10 and later |

```c
typedef struct _OB_CALLBACK_REGISTRATION {
    USHORT Version;                    /* OB_FLT_REGISTRATION_VERSION == 0x0100 */
    USHORT OperationRegistrationCount;
    UNICODE_STRING Altitude;
    PVOID RegistrationContext;
    OB_OPERATION_REGISTRATION *OperationRegistration;
} OB_CALLBACK_REGISTRATION, *POB_CALLBACK_REGISTRATION;

typedef struct _OB_OPERATION_REGISTRATION {
    POBJECT_TYPE *ObjectType;          /* PsProcessType / PsThreadType / ExDesktopObjectType */
    OB_OPERATION Operations;           /* OB_OPERATION_HANDLE_CREATE 0x1, _DUPLICATE 0x2 */
    POB_PRE_OPERATION_CALLBACK PreOperation;    /* may be NULL only if Post is not */
    POB_POST_OPERATION_CALLBACK PostOperation;
} OB_OPERATION_REGISTRATION, *POB_OPERATION_REGISTRATION;

typedef struct _OB_PRE_OPERATION_INFORMATION {
    OB_OPERATION Operation;
    union { ULONG Flags; struct { ULONG KernelHandle : 1; ULONG Reserved : 31; }; };
    PVOID Object; POBJECT_TYPE ObjectType;
    PVOID CallContext;                 /* driver writes it, post-op reads it */
    POB_PRE_OPERATION_PARAMETERS Parameters;
} OB_PRE_OPERATION_INFORMATION, *POB_PRE_OPERATION_INFORMATION;
```

`OB_POST_OPERATION_INFORMATION` has the same first six members plus `NTSTATUS ReturnStatus` before `Parameters`. The pre-op parameter union holds `OB_PRE_CREATE_HANDLE_INFORMATION` and `OB_PRE_DUPLICATE_HANDLE_INFORMATION`; the post-op union holds `OB_POST_CREATE_HANDLE_INFORMATION` and `OB_POST_DUPLICATE_HANDLE_INFORMATION`, each with one read-only `ACCESS_MASK GrantedAccess`. Both pre-op arms begin with `ACCESS_MASK DesiredAccess; ACCESS_MASK OriginalDesiredAccess;` and the duplicate arm continues with `PVOID SourceProcess; PVOID TargetProcess;`, so shared code may touch the first two members but must still select the union arm by `Operation`.

| Object type | Registration pointer | Operations | Availability |
|---|---|---|---|
| Process | `PsProcessType` | open, duplicate | Windows Vista and later |
| Thread | `PsThreadType` | open, duplicate | Windows Vista and later |
| Desktop | `ExDesktopObjectType` | open, duplicate | Windows 10 and later |
| Token, section, file, key, event | none | — | not interceptable; use `wk-minifilter-basics` or `wk-registry-callbacks` |

| Right | Value | Effect of removal |
|---|---|---|
| `PROCESS_TERMINATE` | `0x0001` | `TerminateProcess` fails; the process keeps running |
| `PROCESS_CREATE_THREAD` | `0x0002` | `CreateRemoteThread` into the target fails |
| `PROCESS_VM_OPERATION` | `0x0008` | `VirtualAllocEx`/`VirtualProtectEx` fail |
| `PROCESS_VM_WRITE` | `0x0020` | `WriteProcessMemory` fails |
| `PROCESS_VM_READ` | `0x0010` | `ReadProcessMemory` fails; breaks most legitimate tools |
| `PROCESS_DUP_HANDLE` | `0x0040` | handle duplication and handle stealing fail |
| `THREAD_SET_CONTEXT` | `0x0010` | `SetThreadContext` fails |
| `THREAD_TERMINATE` | `0x0001` | `TerminateThread` fails |
| `SYNCHRONIZE` | `0x00100000` | **do not remove**: breaks waits and every monitoring tool |

## Step-by-step

1. Choose the smallest set of object types and operations; process create plus duplicate is the common minimum. Each added registration slows a hot path.
2. Define the deny mask as a named constant from documented rights. If a right is missing from the kernel headers, guard it with `#ifndef` and the documented value.
3. Initialize state before registration: push lock, PID table, counters, and the IO-drain event. Check the DbgPrint that initialization ran first.
4. Fill one `OB_OPERATION_REGISTRATION` per object type with both operations and both callbacks; fill `OB_CALLBACK_REGISTRATION` with `Version = OB_FLT_REGISTRATION_VERSION` and a unique altitude string.
5. Call `ObRegisterCallbacks`, log the status, and unload on failure. A protector that silently misses registration is a false sense of security.
6. Order pre-op checks cheapest first: `ObjectType`, `KernelHandle`, `Operation`, mask overlap, then the PID lookup under the lock. Every path returns `OB_PREOP_SUCCESS`.
7. Where the mask shrinks, store a marker in `Info->CallContext` that outlives the call (a module-level address, never a stack address).
8. In the post-op, return early on `!NT_SUCCESS(ReturnStatus)`, count by operation, and read `GrantedAccess` for logging only.
9. In unload: stop new I/O, `ObUnRegisterCallbacks`, delete the symbol link, delete the device, then free state. Check the handle is `NULL` before the device goes away.

## Complete example

The driver protects a set of process IDs: it removes termination, memory-write, and remote-thread rights from non-kernel handles opened to a protected process, counts the modified opens, and exposes add/remove/clear/query over IOCTLs. The PID list is persisted under the service key.

```c
/* ProtectorPublic.h - shared by driver and client */
#pragma once
#define PROTECTOR_DEVICE_TYPE   0x8330
#define PROTECTOR_SYMLINK_NAME  L"\\??\\KProtector"
#define PROTECTOR_WIN32_NAME    L"\\\\.\\KProtector"

#define IOCTL_PROTECTOR_ADD_PID \
    CTL_CODE(PROTECTOR_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_PROTECTOR_REMOVE_PID \
    CTL_CODE(PROTECTOR_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_PROTECTOR_CLEAR \
    CTL_CODE(PROTECTOR_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_PROTECTOR_QUERY \
    CTL_CODE(PROTECTOR_DEVICE_TYPE, 0x803, METHOD_BUFFERED, FILE_READ_DATA)
```

```c
/* Protector.c */
#include <ntddk.h>
#include "ProtectorPublic.h"

#define PROTECTOR_TAG          'troP'   /* pool tag if this driver starts allocating */
#define PROTECTOR_MAX_PIDS     256
#define PROTECTOR_ALTITUDE     L"32768.1842"

#ifndef PROCESS_TERMINATE
#define PROCESS_TERMINATE      0x0001
#endif
#ifndef PROCESS_CREATE_THREAD
#define PROCESS_CREATE_THREAD  0x0002
#endif
#ifndef PROCESS_VM_WRITE
#define PROCESS_VM_WRITE       0x0020
#endif
#define PROTECTOR_DENIED_MASK \
    (PROCESS_TERMINATE | PROCESS_CREATE_THREAD | PROCESS_VM_WRITE)

typedef struct _PROTECTOR_STATE {
    EX_PUSH_LOCK Lock;
    ULONG PidCount;                    /* Pids[0..PidCount-1] is compact */
    ULONG Pids[PROTECTOR_MAX_PIDS];
    PVOID RegistrationHandle;
    PDEVICE_OBJECT DeviceObject;
    BOOLEAN SymLinkCreated;
    WCHAR ServicePath[300];            /* NUL-terminated copy of the registry path */
    volatile LONG Stopping;
    volatile LONG IoInFlight;
    KEVENT IoDrained;
    volatile LONG StrippedOpens;
    volatile LONG StrippedDuplicates;
} PROTECTOR_STATE;

typedef struct _PROTECTOR_DEVICE_EXTENSION {
    PROTECTOR_STATE *State;
} PROTECTOR_DEVICE_EXTENSION, *PPROTECTOR_DEVICE_EXTENSION;

static PROTECTOR_STATE g_State;
static const UCHAR g_StripMarker = 0;  /* stable address used as CallContext */

/* Every *Locked helper requires g_State.Lock held. */
static BOOLEAN ProtectAddLocked(ULONG Pid)
{
    ULONG i;
    if (Pid == 0 || Pid == 4)          /* never protect Idle or System */
        return FALSE;
    for (i = 0; i < g_State.PidCount; i++)
        if (g_State.Pids[i] == Pid)
            return FALSE;
    if (g_State.PidCount >= PROTECTOR_MAX_PIDS)
        return FALSE;
    g_State.Pids[g_State.PidCount++] = Pid;
    return TRUE;
}

static BOOLEAN ProtectRemoveLocked(ULONG Pid)
{
    ULONG i;
    for (i = 0; i < g_State.PidCount; i++) {
        if (g_State.Pids[i] == Pid) {
            g_State.Pids[i] = g_State.Pids[--g_State.PidCount];
            return TRUE;
        }
    }
    return FALSE;
}

static BOOLEAN IsProtectedLocked(ULONG Pid)
{
    ULONG i;
    for (i = 0; i < g_State.PidCount; i++)
        if (g_State.Pids[i] == Pid)
            return TRUE;
    return FALSE;
}

static OB_PREOP_CALLBACK_STATUS
ProtectPreOperation(PVOID RegistrationContext, POB_PRE_OPERATION_INFORMATION Info)
{
    ACCESS_MASK *desired;
    ULONG pid;
    BOOLEAN prot;
    UNREFERENCED_PARAMETER(RegistrationContext);
    if (Info->ObjectType != *PsProcessType || Info->KernelHandle)
        return OB_PREOP_SUCCESS;
    if (Info->Operation == OB_OPERATION_HANDLE_CREATE)
        desired = &Info->Parameters->CreateHandleInformation.DesiredAccess;
    else if (Info->Operation == OB_OPERATION_HANDLE_DUPLICATE)
        desired = &Info->Parameters->DuplicateHandleInformation.DesiredAccess;
    else
        return OB_PREOP_SUCCESS;
    if ((*desired & PROTECTOR_DENIED_MASK) == 0)
        return OB_PREOP_SUCCESS;       /* nothing to take away: skip the lookup */
    pid = HandleToULong(PsGetProcessId((PEPROCESS)Info->Object));
    ExAcquirePushLockShared(&g_State.Lock);
    prot = IsProtectedLocked(pid);
    ExReleasePushLockShared(&g_State.Lock);
    if (!prot)
        return OB_PREOP_SUCCESS;
    *desired &= ~PROTECTOR_DENIED_MASK;
    Info->CallContext = (PVOID)&g_StripMarker;
    return OB_PREOP_SUCCESS;
}

static VOID
ProtectPostOperation(PVOID RegistrationContext, POB_POST_OPERATION_INFORMATION Info)
{
    UNREFERENCED_PARAMETER(RegistrationContext);
    if (Info->ObjectType != *PsProcessType || Info->KernelHandle)
        return;
    if (Info->CallContext != (PVOID)&g_StripMarker)
        return;                        /* pre-op did not modify this request */
    if (!NT_SUCCESS(Info->ReturnStatus))
        return;                        /* no handle was installed */
    if (Info->Operation == OB_OPERATION_HANDLE_CREATE)
        InterlockedIncrement(&g_State.StrippedOpens);
    else if (Info->Operation == OB_OPERATION_HANDLE_DUPLICATE)
        InterlockedIncrement(&g_State.StrippedDuplicates);
}

static OB_OPERATION_REGISTRATION g_Operations[] = {
    { PsProcessType,
      OB_OPERATION_HANDLE_CREATE | OB_OPERATION_HANDLE_DUPLICATE,
      ProtectPreOperation, ProtectPostOperation }
};

static NTSTATUS QueryPidsRoutine(PWSTR ValueName, ULONG ValueType, PVOID ValueData,
                                 ULONG ValueLength, PVOID Context, PVOID EntryContext)
{
    PUCHAR bytes = (PUCHAR)ValueData;
    ULONG count, i;
    UNREFERENCED_PARAMETER(ValueName); UNREFERENCED_PARAMETER(Context);
    UNREFERENCED_PARAMETER(EntryContext);
    if (ValueType != REG_BINARY || bytes == NULL || ValueLength < sizeof(ULONG))
        return STATUS_SUCCESS;
    count = ValueLength / sizeof(ULONG);
    ExAcquirePushLockExclusive(&g_State.Lock);
    for (i = 0; i < count; i++) {
        ULONG pid;
        RtlCopyMemory(&pid, bytes + (i * sizeof(ULONG)), sizeof(ULONG));
        ProtectAddLocked(pid);
    }
    ExReleasePushLockExclusive(&g_State.Lock);
    return STATUS_SUCCESS;
}

static VOID LoadProtectedPids(VOID)
{
    RTL_QUERY_REGISTRY_TABLE table[2];
    RtlZeroMemory(table, sizeof(table));
    table[0].QueryRoutine = QueryPidsRoutine;
    table[0].Name = L"ProtectedPids";
    table[0].DefaultType = REG_NONE;   /* absent value: callback is not called */
    if (!NT_SUCCESS(RtlQueryRegistryValues(RTL_REGISTRY_ABSOLUTE, g_State.ServicePath,
                                           table, NULL, NULL)))
        KdPrint(("KProtector: no saved PID list\n"));
}

static VOID SaveProtectedPids(VOID)
{
    ULONG buffer[PROTECTOR_MAX_PIDS];
    ULONG count;
    ExAcquirePushLockShared(&g_State.Lock);
    count = g_State.PidCount;
    RtlCopyMemory(buffer, g_State.Pids, count * sizeof(ULONG));
    ExReleasePushLockShared(&g_State.Lock);
    if (!NT_SUCCESS(RtlWriteRegistryValue(RTL_REGISTRY_ABSOLUTE, g_State.ServicePath,
                                          L"ProtectedPids", REG_BINARY, buffer,
                                          count * sizeof(ULONG))))
        KdPrint(("KProtector: failed to persist PID list\n"));
}

static VOID ProtectReleaseIo(PROTECTOR_STATE *State)
{
    if (InterlockedDecrement(&State->IoInFlight) == 0 && State->Stopping != 0)
        KeSetEvent(&State->IoDrained, IO_NO_INCREMENT, FALSE);
}

static NTSTATUS ProtectCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS ProtectDeviceControl(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PPROTECTOR_DEVICE_EXTENSION ext =
        (PPROTECTOR_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    PROTECTOR_STATE *state = ext->State;
    ULONG inputLength = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outputLength = sp->Parameters.DeviceIoControl.OutputBufferLength;
    ULONG code = sp->Parameters.DeviceIoControl.IoControlCode;
    PVOID buffer = Irp->AssociatedIrp.SystemBuffer;
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG information = 0, count, changed, i;
    ULONG *pids;
    InterlockedIncrement(&state->IoInFlight);
    if (state->Stopping != 0) {
        status = STATUS_DELETE_PENDING;
        goto Complete;
    }
    switch (code) {
    case IOCTL_PROTECTOR_ADD_PID:
    case IOCTL_PROTECTOR_REMOVE_PID:
        if (buffer == NULL || inputLength == 0 || (inputLength % sizeof(ULONG)) != 0) {
            status = STATUS_INVALID_BUFFER_SIZE;
            break;
        }
        pids = (ULONG *)buffer;
        count = inputLength / sizeof(ULONG);
        changed = 0;
        ExAcquirePushLockExclusive(&state->Lock);
        for (i = 0; i < count; i++) {
            if (code == IOCTL_PROTECTOR_ADD_PID) {
                if (ProtectAddLocked(pids[i]))
                    changed++;
            } else if (ProtectRemoveLocked(pids[i])) {
                changed++;
            }
        }
        ExReleasePushLockExclusive(&state->Lock);
        /* warning status: NT_SUCCESS is TRUE, partial count is in Information */
        status = (changed == count) ? STATUS_SUCCESS : STATUS_NOT_ALL_ASSIGNED;
        information = changed * sizeof(ULONG);
        break;
    case IOCTL_PROTECTOR_CLEAR:
        ExAcquirePushLockExclusive(&state->Lock);
        state->PidCount = 0;
        ExReleasePushLockExclusive(&state->Lock);
        status = STATUS_SUCCESS;
        break;
    case IOCTL_PROTECTOR_QUERY: {
        ULONG needed;
        if (buffer == NULL) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        ExAcquirePushLockShared(&state->Lock);
        needed = sizeof(ULONG) + (state->PidCount * sizeof(ULONG));
        if (outputLength < needed) {
            ExReleasePushLockShared(&state->Lock);
            status = STATUS_BUFFER_TOO_SMALL;
            information = needed;
            break;
        }
        ((ULONG *)buffer)[0] = state->PidCount;
        RtlCopyMemory((PUCHAR)buffer + sizeof(ULONG), state->Pids,
                      state->PidCount * sizeof(ULONG));
        ExReleasePushLockShared(&state->Lock);
        status = STATUS_SUCCESS;
        information = needed;
        break;
    }
    default:
        break;
    }

Complete:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    ProtectReleaseIo(state);
    return status;
}

static VOID ProtectorTeardown(BOOLEAN Persist)
{
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(PROTECTOR_SYMLINK_NAME);
    InterlockedExchange(&g_State.Stopping, 1);
    KeResetEvent(&g_State.IoDrained);
    if (InterlockedCompareExchange(&g_State.IoInFlight, 0, 0) != 0)
        KeWaitForSingleObject(&g_State.IoDrained, Executive, KernelMode, FALSE, NULL);
    if (g_State.RegistrationHandle != NULL) {
        ObUnRegisterCallbacks(g_State.RegistrationHandle);
        g_State.RegistrationHandle = NULL;
    }
    if (g_State.SymLinkCreated) {
        IoDeleteSymbolicLink(&symLink);
        g_State.SymLinkCreated = FALSE;
    }
    if (g_State.DeviceObject != NULL) {
        IoDeleteDevice(g_State.DeviceObject);
        g_State.DeviceObject = NULL;
    }
    if (Persist)
        SaveProtectedPids();
}

static VOID ProtectUnload(PDRIVER_OBJECT DriverObject)
{
    UNREFERENCED_PARAMETER(DriverObject);
    ProtectorTeardown(TRUE);
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(L"\\Device\\KProtector");
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(PROTECTOR_SYMLINK_NAME);
    UNICODE_STRING altitude;
    OB_CALLBACK_REGISTRATION registration;
    ExInitializePushLock(&g_State.Lock);
    KeInitializeEvent(&g_State.IoDrained, NotificationEvent, FALSE);
    if (RegistryPath == NULL || RegistryPath->Length == 0 ||
        RegistryPath->Length >= sizeof(g_State.ServicePath))
        return STATUS_INVALID_PARAMETER;
    RtlCopyMemory(g_State.ServicePath, RegistryPath->Buffer, RegistryPath->Length);
    g_State.ServicePath[RegistryPath->Length / sizeof(WCHAR)] = L'\0';
    LoadProtectedPids();
    RtlInitUnicodeString(&altitude, PROTECTOR_ALTITUDE);
    registration.Version = OB_FLT_REGISTRATION_VERSION;
    registration.OperationRegistrationCount = 1;
    registration.Altitude = altitude;
    registration.RegistrationContext = NULL;
    registration.OperationRegistration = g_Operations;
    status = ObRegisterCallbacks(&registration, &g_State.RegistrationHandle);
    if (!NT_SUCCESS(status)) {
        KdPrint(("KProtector: ObRegisterCallbacks failed (0x%08X)\n", status));
        ProtectorTeardown(FALSE);
        return status;
    }
    status = IoCreateDevice(DriverObject, sizeof(PROTECTOR_DEVICE_EXTENSION), &deviceName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN, FALSE,
                            &g_State.DeviceObject);
    if (!NT_SUCCESS(status)) {
        ProtectorTeardown(FALSE);
        return status;
    }
    ((PPROTECTOR_DEVICE_EXTENSION)g_State.DeviceObject->DeviceExtension)->State = &g_State;
    status = IoCreateSymbolicLink(&symLink, &deviceName);
    if (!NT_SUCCESS(status)) {
        ProtectorTeardown(FALSE);
        return status;
    }
    g_State.SymLinkCreated = TRUE;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = ProtectCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = ProtectCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = ProtectDeviceControl;
    DriverObject->DriverUnload = ProtectUnload;
    KdPrint(("KProtector: loaded with %lu protected PID(s)\n", g_State.PidCount));
    return STATUS_SUCCESS;
}
```

```c
/* ProtectClient.c - add, remove, clear, query */
#include <windows.h>
#include <stdio.h>
#include "ProtectorPublic.h"

static BOOL SendIoctl(HANDLE Device, DWORD Code, LPVOID In, DWORD InBytes,
                      LPVOID Out, DWORD OutBytes, DWORD *Returned)
{
    if (!DeviceIoControl(Device, Code, In, InBytes, Out, OutBytes, Returned, NULL)) {
        printf("DeviceIoControl 0x%08lX failed: %lu\n", Code, GetLastError());
        return FALSE;
    }
    return TRUE;
}

int wmain(int argc, wchar_t **argv)
{
    HANDLE device;
    DWORD pids[64];
    DWORD count = 0, returned = 0, i;
    BYTE buffer[sizeof(DWORD) + (256 * sizeof(DWORD))];
    int arg;

    if (argc < 2) {
        printf("usage: protect add|remove <pid>... | clear | query\n");
        return 1;
    }
    device = CreateFileW(PROTECTOR_WIN32_NAME, GENERIC_READ | GENERIC_WRITE,
                         FILE_SHARE_READ | FILE_SHARE_WRITE, NULL, OPEN_EXISTING, 0, NULL);
    if (device == INVALID_HANDLE_VALUE) {
        printf("CreateFileW failed: %lu (driver started?)\n", GetLastError());
        return 1;
    }

    if (_wcsicmp(argv[1], L"add") == 0 || _wcsicmp(argv[1], L"remove") == 0) {
        for (arg = 2; arg < argc && count < 64; arg++)
            pids[count++] = (DWORD)_wtoi(argv[arg]);
        if (count == 0 || !SendIoctl(device,
                _wcsicmp(argv[1], L"add") == 0 ? IOCTL_PROTECTOR_ADD_PID
                                               : IOCTL_PROTECTOR_REMOVE_PID,
                pids, count * sizeof(DWORD), NULL, 0, &returned))
            goto Fail;
        printf("applied %lu of %lu PID(s)\n", returned / sizeof(DWORD), count);
    } else if (_wcsicmp(argv[1], L"clear") == 0) {
        if (!SendIoctl(device, IOCTL_PROTECTOR_CLEAR, NULL, 0, NULL, 0, &returned))
            goto Fail;
    } else if (_wcsicmp(argv[1], L"query") == 0) {
        if (!SendIoctl(device, IOCTL_PROTECTOR_QUERY, NULL, 0, buffer, sizeof(buffer),
                       &returned))
            goto Fail;
        printf("protected PIDs (%lu):\n", *(DWORD *)buffer);
        for (i = 0; i < *(DWORD *)buffer; i++)
            printf("  %lu\n", ((DWORD *)(buffer + sizeof(DWORD)))[i]);
    } else {
        printf("unknown command\n");
        goto Fail;
    }

    CloseHandle(device);
    return 0;
Fail:
    CloseHandle(device);
    return 1;
}
```

## Variations and advanced use

- **Add thread objects.** Append a second `OB_OPERATION_REGISTRATION` with `PsThreadType` and a pre-op that strips `THREAD_SET_CONTEXT | THREAD_IMPERSONATE | THREAD_SET_THREAD_TOKEN`; keep `OperationRegistrationCount` equal to the array length or registration fails.
- **Handle PID reuse.** A PID is recycled, so protecting 4321 protects whatever process gets that ID next. Store `PsGetProcessCreateTimeQuadPart` beside each PID and compare both, or add PIDs from a `PsSetCreateProcessNotifyRoutineEx` policy on image name.
- **Avoid `PEPROCESS` in the list.** A referenced `PEPROCESS` cannot be recycled, but the reference keeps a terminated process object alive and leaks kernel memory on a bad path. PID plus creation time is the durable key.
- **`ERESOURCE` instead of a push lock.** Large, rarely updated sets benefit from writer preference and shared waiters; a push lock spins but is cheaper. Neither may be held across a call that blocks.
- **Cost.** The pre-op runs for every process and thread handle open system-wide, including thousands per second during builds and service starts. An exclusive lock, a paged allocation, or a `DbgPrint` in that path turns process creation into a bottleneck; time 100000 `OpenProcess` calls with the driver started and stopped to see the delta.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `ObRegisterCallbacks` returns `0xC0000022` at load | image not linked `/INTEGRITYCHECK`, or unsigned with test signing off | add the linker flag, re-sign, `sc start KProtector` |
| Registration fails with `STATUS_FLT_INSTANCE_ALTITUDE_COLLISION` | another loaded driver uses the same altitude | change the fractional digits, log the value used |
| Registration returns `0xC000000D` | `Version` wrong, count zero or larger than the array, both callbacks `NULL` | print every field before the call |
| Task Manager cannot end the task but can open the process | expected: the pre-op shrank the mask and the open still succeeded | `handle -p <pid>` shows the mask without `0x0001` |
| The protected process still dies from its own window | `ExitProcess` uses the process's own handle | protect processes without a UI, or use another mechanism |
| Stop fails with `ERROR_SERVICE_CANNOT_ACCEPT_CTRL` / `1051` family | a client still holds the device open, or `Stopping` was never set | close clients first, check the drain path |
| Bugcheck `0x9F` around sleep or shutdown | a callback waited (event, file I/O) while a power transition held the caller | remove every wait and I/O from callbacks; `!analyze -v`, `!stacks 2` |
| Bugcheck `0x139` after unload | callback ran after state was freed, or a push lock was used recursively | verify unregistration precedes the free; `!analyze -v` |
| Protected processes never exit, `!objsize` keeps growing | the callback references objects and leaks the reference | remove `ObReferenceObject` from the callback |
| Unload hangs in `ObUnRegisterCallbacks` | the lock the callback takes is held by the unloader | release the lock before unregistering |
| Process creation slows system-wide | exclusive lock, pool allocation, or logging on the hot path | take the shared lock only after the mask check |
| `DeviceIoControl` reports success but fewer PIDs applied | `STATUS_NOT_ALL_ASSIGNED` is a warning, `NT_SUCCESS` is TRUE | read `lpBytesReturned` |

## Safety, IRQL, and cleanup checklist

- Register at PASSIVE_LEVEL in `DriverEntry`; unregister at PASSIVE_LEVEL before the device is deleted and before state is freed.
- `ObUnRegisterCallbacks` waits for running callbacks. Never call it while holding a lock a callback acquires, and never from inside a callback.
- Callbacks must not wait, must not do file or registry I/O, and must not re-enter `ObRegisterCallbacks`/`ObUnRegisterCallbacks`.
- Filter `KernelHandle` first; stripping kernel handles breaks the memory manager and dump writers.
- Keep callbacks and their state non-paged: the state is loader-allocated and touched on every handle open.
- `CallContext` must outlive the operation: module-level storage only.
- Strip the minimum; removing `SYNCHRONIZE` or `PROCESS_QUERY_LIMITED_INFORMATION` breaks monitoring tools users rely on.
- Drain in-flight IOCTLs with a counter plus `KEVENT`; a dispatch after the extension is gone is a `0x50` or `0x139`.
- Delete the symbol link before the device object, set the registration handle to `NULL` after unregistering, and free every allocation on the reverse path.
- The device object may outlive unload while a client holds a handle; never free the device extension yourself.
- Persist state before freeing it and tolerate a missing key on first boot.

## Verification

1. `msbuild Protector.vcxproj /p:Configuration=Release /p:Platform=x64` with `/INTEGRITYCHECK` in linker options; `dumpbin /headers Protector.sys | findstr /i integrity` confirms the flag.
2. `signtool sign /fd SHA256 /a Protector.sys`, then `sc create KProtector type= kernel binPath= C:\Drivers\Protector.sys` and `sc start KProtector`. DebugView shows `KProtector: loaded with 0 protected PID(s)`. A `577`/`1275` start failure is signing, not callbacks.
3. `start notepad`, `protect add <pid>`, then Task Manager, End task. Expected "Access is denied" and notepad still running; `protect query` lists the PID.
4. From Process Explorer, open a handle to the protected process: the open succeeds, `TerminateProcess` fails with `ERROR_ACCESS_DENIED (5)`.
5. Run `handle -p <pid>` and watch the access mask of newly created handles: `0x0001` missing, `SYNCHRONIZE` present.
6. `reg query HKLM\SYSTEM\CurrentControlSet\Services\KProtector /v ProtectedPids`, stop and start the driver, `protect query` shows the same PIDs.
7. Close the client and `sc stop KProtector`; expected success, no further callback activity, and a fresh start with no altitude collision.
8. `!drvobj KProtector 2` lists the device object and unload routine; after stop, `lm` no longer shows the module and `!object \Device\KProtector` is gone.
