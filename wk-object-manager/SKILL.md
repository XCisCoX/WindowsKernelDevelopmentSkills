---
name: wk-object-manager
description: Use when opening, creating, or closing Windows kernel objects from a driver - ZwOpenProcess and ZwOpenThread by client ID, ZwCreateFile and ZwOpenFile by NT path, IoCreateDeviceSecure with an SDDL descriptor, symbolic links, OBJECT_ATTRIBUTES flag values, OBJ_KERNEL_HANDLE, handle versus pointer semantics, ObReferenceObjectByHandle, access checks, previous mode, and leak-free cleanup paths.
---

# Windows object manager: names, handles, and references

Every kernel resource a driver touches - process, thread, file, event, key, device, section - is an object allocated in system space and owned by the object manager. User mode reaches those objects through handles, which are per-process table indices carrying an access mask; kernel code reaches them through referenced pointers, which stay valid until the reference is dropped. The contract has two halves that must be satisfied together: a successful open either produces a handle to close or a pointer to dereference, and every failure path unwinds exactly what succeeded before it. The rule that eliminates most leaks and use-after-frees: count acquisitions, not releases - if the number of successful opens does not equal the number of closes on every exit path, the driver is wrong even when it appears to work.

## When to use

- Opening a process or thread by ID (`ZwOpenProcess`, `ZwOpenThread`, `PsLookupProcessByProcessId`) with a minimal access mask.
- Opening a file, directory, or device by NT path from kernel mode (`ZwCreateFile`, `ZwOpenFile`, `IoGetDeviceObjectPointer`).
- Creating a named device object with an explicit security descriptor and a `\DosDevices` symbolic link.
- Converting a handle received from user mode into a referenced kernel pointer, and getting the access check right.
- Diagnosing `STATUS_OBJECT_NAME_NOT_FOUND`, `STATUS_OBJECT_NAME_COLLISION`, `STATUS_ACCESS_DENIED`, `STATUS_INVALID_HANDLE`, or `0xC0000022` returned to a client.
- Deciding between a name, an ID, a handle, and a direct pointer for a new interface.
- Auditing a driver for leaked references, leaked handles, or use of a user handle in kernel mode.

Do not use for: IRP construction and forwarding (`wk-irp-basics`); registry key semantics and value types (`wk-registry-programming`); stripping access in handle-open callbacks (`wk-object-callbacks`); full device object and stack lifecycle (`wk-device-objects`); enumerating all processes or handles system-wide (`wk-system-services`).

## Prerequisites and related skills

- Kernel-mode C, WDK 10.0.22621 or later, x64 unless stated; `<ntifs.h>` (which pulls in the WDM declarations) plus `<wdmsec.h>` when a security descriptor is supplied at device creation. Link `Wdmsec.lib` for `IoCreateDeviceSecure`.
- All open, create, and close calls in this skill run at **PASSIVE_LEVEL**. `ObDereferenceObject` also works at DISPATCH_LEVEL; nothing else here does.
- Related skills, and why: `wk-device-objects` for device flags, `DO_DEVICE_INITIALIZING` and stack setup; `wk-calling-drivers` for `IoGetDeviceObjectPointer` and IRPs aimed at another driver; `wk-system-services` for `ZwQuerySystemInformation` and handle enumeration; `wk-kernel-security` for SDDL hardening and IOCTL attack surface; `wk-kernel-debugging` for `!object`, `!handle`, `!trueref`.

## Mental model

### The namespace is a tree of named objects

The object manager keeps a single hierarchy rooted at `\`. Only objects created with a name appear in it, so the tree is a directory service, not an inventory of every live object.

```
\  (root directory object)
+-- Device                          device objects
|   +-- ObjDemo                     <- created by this skill's example
|   +-- HarddiskVolume3
|   +-- NamedPipe
+-- DosDevices  --symlink--> \??    legacy alias of the DOS device map
+-- ??                              \??\C: -> \Device\HarddiskVolume3
|   +-- C:
+-- GLOBAL??                        machine-wide DOS device map
+-- SystemRoot  --symlink--> \Device\HarddiskVolume3\Windows
+-- BaseNamedObjects                session 0 events, mutexes, sections
+-- Sessions
|   +-- 1
|       +-- BaseNamedObjects        per-session named objects
|       +-- DosDevices              per-session drive letters
+-- AppContainerNamedObjects        package-scoped names
+-- KernelObjects                   \KernelObjects\LowMemoryCondition
+-- Callback                        \Callback\SetSystemTime
+-- Registry                        \Registry\Machine, \Registry\User
+-- FileSystem                      \FileSystem\Ntfs, \FileSystem\Filters
+-- Driver                          \Driver\Tcpip, \Driver\HTTP
+-- ObjectTypes                     one directory entry per object type
+-- Security                        \Security\SIDs, cached SID lookups
```

Rules that follow from the shape of the tree:

- A driver-created device named `\Device\ObjDemo` is reachable by kernel code immediately, but a user-mode `CreateFileW` cannot see it. User mode goes through the DOS device map: `\DosDevices\ObjDemo` (equivalently `\??\ObjDemo`) is a symbolic link that must point at the device.
- `\DosDevices` and `\??` resolve to the DOS device directory of the current session; `\GLOBAL??` is the machine-wide one. A name created in session 1 is invisible in session 0 unless it is placed under `\GLOBAL??` or `\BaseNamedObjects` from a system process.
- Names are case-insensitive for lookups only when `OBJ_CASE_INSENSITIVE` is supplied; without it, `\Device\objdemo` and `\Device\ObjDemo` are different names. Always set the flag for names that also arrive from user mode.
- Name length is bounded by `UNICODE_STRING::Length` (a `USHORT`, so 32767 characters at most) and by the object manager's per-component limits; a fixed 260-character buffer is a user-mode convention, not a kernel limit.
- Anyone with create rights in a directory can create a name there first and take it. That is name squatting: a driver that opens `\DosDevices\MyDevice` by name may be talking to an attacker's device. Create your own names in `\Device` (writable only by administrators and the kernel) and prefer device interfaces for discovery.

### Handles are table indices with an access mask

A handle is an index into a handle table, multiplied by four. The low two bits of a user-visible handle value are not part of the index: they carry handle attributes (inherit, protect-from-close) in some APIs, which is why handle values must be treated as opaque.

```
 user thread: CreateFileW("\\\\.\\ObjDemo")
      |
      v  NtCreateFile, PreviousMode = UserMode
 ObOpenObjectByName
      |
      +--> parse "\??\ObjDemo":
      |      \?? is a symlink to \GLOBAL??
      |      \GLOBAL??\ObjDemo -> symlink -> \Device\ObjDemo
      +--> object header: type = Device, PointerCount = N, HandleCount = M
      +--> SeAccessCheck(device SD, subject context, desired access, UserMode)
      +--> allocate entry in the *current process* handle table, index 7
      v
 returns HANDLE 0x1C  (7 * 4)

 kernel driver: ObReferenceObjectByHandle(0x1C, access, *IoDeviceObjectType, KernelMode)
      +--> AccessMode KernelMode: entry must be in the system handle table
      +--> compare GrantedAccess against DesiredAccess
      +--> ObjectHeader->PointerCount++
      v
 returns DEVICE_OBJECT body pointer

 kernel driver: ObDereferenceObject(ptr)   -> PointerCount--
```

Handle facts that cause real bugs:

- **Handle 0 is never valid.** `CreateFileW` returns `INVALID_HANDLE_VALUE` (`(HANDLE)-1`) on failure, not `NULL`; `OpenProcess` returns `NULL`. Test the API's documented failure value, not "not zero".
- **Pseudo-handles are negative values**: `NtCurrentProcess()` is `(HANDLE)-1` and `NtCurrentThread()` is `(HANDLE)-2`; `ZwCurrentProcess()` and `ZwCurrentThread()` are the kernel-mode spellings of the same values. Passing a pseudo-handle to `ObReferenceObjectByHandle` with `KernelMode` works because the object manager special-cases them.
- **A closed handle value can be reused** by the next open in the same process. Two handles are the same object only if both are open at the same time and refer to the same object; comparing stored handle *values* for identity is a bug.
- **A handle is only meaningful in the table that owns it.** Using a process handle from another process fails with `STATUS_INVALID_HANDLE`, and a kernel handle passed down to user mode cannot be used there.
- **Process and thread IDs are handle values from the system handle table**, which is why they are always multiples of 4 and never zero. `UniqueProcess` in a `CLIENT_ID` is therefore a small integer, not a pointer.

### Kernel handles versus process handles

There are two handle tables that matter: the per-process table (`EPROCESS->ObjectTable`) and the system table used by the System process, which is where kernel handles live.

- A **process handle** is owned by one process. It becomes invalid when that process exits, its access is checked against that process's token, and a kernel driver that stores one keeps the target process's handle table alive - a leak with a security dimension.
- A **kernel handle** lives in the system table, is valid in every process context, is usable only from kernel mode, and dies only when closed. It survives the exit of whatever process happened to be current when it was created.

A handle created by a `Zw*` routine while `PreviousMode` is `KernelMode` is allocated in the system table. `OBJ_KERNEL_HANDLE` states that requirement explicitly, and is mandatory when the same code could run with a different previous mode, because the fallback is a handle in the user's table that the user can duplicate, use, or keep open after the driver thinks it is done.

**Never accept a handle from user mode and pass it to a `Zw*` routine as if it were a kernel handle.** Either resolve it with `ObReferenceObjectByHandle` using `AccessMode = UserMode` and work with the pointer, or duplicate it into a kernel handle with `ZwDuplicateObject` using `NtCurrentProcess()` as the source process.

### References, not handles, keep objects alive

Every open adds a reference. An object's body is freed only when its pointer count reaches zero, and `!object` shows both counters:

- **HandleCount** - number of open handles that refer to the object.
- **PointerCount** - handles plus explicit references taken by kernel code.
- `ObReferenceObjectByHandle` and `ObReferenceObjectByPointer` increment the pointer count; `ObDereferenceObject` decrements it.
- `!trueref` resolves the count when the header uses a packed or deferred representation and the raw numbers look implausible.

Two failure shapes:

- **Leaked reference** - the object is never freed; memory grows with no bugcheck. Detect by comparing pointer counts before and after a stress loop.
- **Extra dereference** - the object is freed while another component still holds it; the next access is a `0x50` fault on freed pool, or `0xC2 BAD_POOL_CALLER` if the pool block is reused. Driver Verifier with special pool turns this into an immediate, attributable bugcheck.

### Access is checked at open time, then replayed from the handle

An open performs one security check: the object's security descriptor is evaluated against the caller's subject context for the requested access mask. Afterwards, every operation through that handle is validated against the *granted* mask stored in the handle table, not against the descriptor. That is why `ObReferenceObjectByHandle` called with a larger `DesiredAccess` fails with `STATUS_ACCESS_DENIED` even though the handle is valid: the caller asked for more than the handle carries.

Useful masks:

| Object | Access mask | Value |
| --- | --- | --- |
| Process | `PROCESS_TERMINATE` / `PROCESS_CREATE_THREAD` | `0x0001` / `0x0002` |
| Process | `PROCESS_VM_READ` / `PROCESS_VM_WRITE` / `PROCESS_VM_OPERATION` | `0x0010` / `0x0020` / `0x0008` |
| Process | `PROCESS_QUERY_INFORMATION` / `PROCESS_QUERY_LIMITED_INFORMATION` | `0x0400` / `0x1000` |
| Process | `PROCESS_SET_INFORMATION` / `PROCESS_SUSPEND_RESUME` | `0x0200` / `0x0800` |
| Thread | `THREAD_TERMINATE` / `THREAD_SUSPEND_RESUME` | `0x0001` / `0x0002` |
| Thread | `THREAD_GET_CONTEXT` / `THREAD_SET_CONTEXT` | `0x0008` / `0x0010` |
| Thread | `THREAD_QUERY_INFORMATION` / `THREAD_QUERY_LIMITED_INFORMATION` | `0x0040` / `0x0800` |
| File | `FILE_READ_DATA` / `FILE_WRITE_DATA` / `FILE_READ_ATTRIBUTES` | `0x0001` / `0x0002` / `0x0080` |
| File | `FILE_READ_EA` / `FILE_WRITE_EA` / `DELETE` | `0x0008` / `0x0010` / `0x00010000` |
| Any | `READ_CONTROL` / `WRITE_DAC` / `WRITE_OWNER` / `SYNCHRONIZE` | `0x00020000` / `0x00040000` / `0x00080000` / `0x00100000` |

Ask for the least access that does the job. A driver that opens every process with `PROCESS_ALL_ACCESS` fails on protected processes, trips security products, and hands an attacker a far more useful primitive if the handle ever leaks.

### Previous mode decides who probes, and who checks

`Nt*` routines trust `PreviousMode`; `Zw*` wrappers set it to `KernelMode` for the duration of the call. Three practical consequences:

1. `Zw*` never probes its arguments. Passing a user-mode pointer to `ZwCreateFile` makes the kernel read the caller's address space as if it were kernel memory: a silent read of attacker-chosen data, or a `0x50`/`0x3B` bugcheck.
2. Handles created under `Zw*` with `PreviousMode == KernelMode` go to the system table, so they are kernel handles.
3. When the operation must run with the *caller's* security context - impersonation, or opening a handle on behalf of a user-mode request - either use `ObReferenceObjectByHandle` with `AccessMode = UserMode`, or set `OBJ_FORCE_ACCESS_CHECK` so the object manager performs the check even for a kernel-mode open.

Driver Verifier enforces the boundary: passing a user handle to `ObReferenceObjectByHandle` with `AccessMode = KernelMode` produces **bugcheck `0xC4` with subcode `0xF6`** on Windows 7 and later. Do not rely on the verifier to catch it on a customer's machine.

## Key APIs and structures

| API / structure | Purpose | IRQL | Notes |
| --- | --- | --- | --- |
| `InitializeObjectAttributes` | fill an `OBJECT_ATTRIBUTES` | any | macro; zeroes the structure first |
| `RTL_CONSTANT_OBJECT_ATTRIBUTES` | static attributes for a literal name | any | initializer form; name buffer must be static |
| `ZwOpenProcess` | handle to a process by `CLIENT_ID` | PASSIVE | `OBJ_KERNEL_HANDLE` for a kernel handle |
| `ZwOpenThread` | handle to a thread by `CLIENT_ID` | PASSIVE | `UniqueThread` is the TID |
| `ZwCreateFile` | create or open a file or device | PASSIVE | the general-purpose "open by name" call |
| `ZwOpenFile` | open an existing file | PASSIVE | no create disposition, no EA |
| `ZwQueryInformationFile` | file size, attributes, position | PASSIVE | needs an appropriate access mask |
| `ZwClose` | close any handle | PASSIVE | not for pseudo-handles |
| `ObReferenceObjectByHandle` | handle to referenced pointer, with access check | PASSIVE | pass `UserMode` for user handles |
| `ObReferenceObjectByPointer` | add a reference to a pointer | up to DISPATCH | no access check, no handle |
| `ObOpenObjectByPointer` | pointer to handle, with access check | PASSIVE | returns a handle in the current context |
| `ObDereferenceObject` | release a reference | up to DISPATCH | exactly once per reference |
| `ObGetObjectType` | `POBJECT_TYPE` of an object | any | compare against `*PsProcessType`, `*PsThreadType`, `*IoFileObjectType`, `*ExEventObjectType` |
| `ObReferenceObjectByName` | pointer from a full object path | PASSIVE | declared in `ntifs.h`; prefer typed helpers |
| `IoCreateDeviceSecure` | named device with an SDDL descriptor | PASSIVE | `Wdmsec.lib`, custom class GUID |
| `IoCreateSymbolicLink` | create `\DosDevices\Name` | PASSIVE | target is the `\Device` name |
| `IoDeleteSymbolicLink`, `IoDeleteDevice` | teardown | PASSIVE | link first, then device |

`OBJECT_ATTRIBUTES` field order (the structure is versioned by `Length`, so always initialize it with the macro rather than by hand): `Length`, `RootDirectory`, `ObjectName`, `Attributes`, `SecurityDescriptor`, `SecurityQualityOfService`.

Flags accepted in `Attributes`, with literal values:

| Flag | Value | Meaning and when to use it |
| --- | --- | --- |
| `OBJ_INHERIT` | `0x00000002` | handle is inheritable by child processes; meaningless for kernel handles |
| `OBJ_PERMANENT` | `0x00000010` | object survives until explicitly deleted; requires `SeCreatePermanentPrivilege` |
| `OBJ_EXCLUSIVE` | `0x00000020` | exclusive access object; rarely appropriate for devices |
| `OBJ_CASE_INSENSITIVE` | `0x00000040` | case-insensitive name lookup; set it for names that may come from user mode |
| `OBJ_OPENIF` | `0x00000080` | if the name exists, open it instead of failing with a collision |
| `OBJ_OPENLINK` | `0x00000100` | open the symbolic link itself, not its target |
| `OBJ_KERNEL_HANDLE` | `0x00000200` | allocate the handle in the system table; **required for driver-owned handles** |
| `OBJ_FORCE_ACCESS_CHECK` | `0x00000400` | run the access check even for a kernel-mode open |
| `OBJ_IGNORE_IMPERSONATED_DEVICEMAP` | `0x00000800` | resolve names against the system device map, not the impersonated user's |
| `OBJ_DONT_REPARSE` | `0x00001000` | fail instead of following a reparse point in the path |
| `OBJ_VALID_ATTRIBUTES` | `0x00001FF2` | mask of all valid bits; use it to validate untrusted input |

## Step-by-step: open anything without leaking

1. **Pick the addressing mechanism.** ID for process and thread (`ZwOpenProcess`, `ZwOpenThread`), NT path for files and devices (`ZwCreateFile`), direct pointer only when the caller already owns a reference. Never invent a pointer from a handle value.
2. **Build the name.** A kernel-mode `ZwCreateFile` path is an NT path: `\Device\HarddiskVolume3\tmp\a.txt`, `\SystemRoot\System32\ntoskrnl.exe`, `\??\C:\tmp\a.txt`, or `\Registry\Machine\Software\Vendor` for keys. A user-supplied `C:\tmp\a.txt` must be translated (typically by prepending `\??\`), and a user-supplied path must be validated for length and for reparse points (`OBJ_DONT_REPARSE`) before it reaches a privileged open.
3. **Fill `UNICODE_STRING` correctly.** `Length` is in bytes and excludes any terminator; `MaximumLength` must be at least `Length`. `ZwCreateFile` accepts a non-terminated buffer, so a `Length` that includes stray bytes is a path-truncation bug.
4. **Initialize `OBJECT_ATTRIBUTES` with the macro**, never field by field: `InitializeObjectAttributes(&oa, &name, OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE, NULL, NULL)`. Add `OBJ_FORCE_ACCESS_CHECK` when the open must be evaluated against the caller's token.
5. **Choose the minimal access mask** from the table above. Add `SYNCHRONIZE` when the handle will be used with synchronous I/O or waited on.
6. **Call and check.** `NT_SUCCESS(status)` is the only acceptable test. Return the status unchanged to the caller where possible: the I/O manager and `RtlNtStatusToDosError` already map it to a Win32 error.
7. **Convert or keep.** If the routine needs the object body, call `ObReferenceObjectByHandle(handle, access, *PsProcessType, KernelMode, &pointer, NULL)` and then close the handle immediately if it was only a means to the pointer. If the handle itself is the deliverable, record who closes it.
8. **Use the pointer.** Do not cache borrowed pointers (`PsGetCurrentProcess`) or dereference a pointer after releasing it. A referenced pointer is safe against handle closure, not against your own `ObDereferenceObject`.
9. **Unwind in reverse order** on every failure path, and free the last acquisition first: close handles, dereference objects, delete the symbolic link, then delete the device. A `goto` chain with one label per acquisition is the clearest encoding of this rule.
10. **Prove it.** After a stress loop, object counts must return to the baseline: `!object \Device\ObjDemo` shows `HandleCount 0` with the pre-test `PointerCount`, and `!handle 0 0` shows no handles of your type left in the test process.

## Complete example

The driver creates a secured named device plus a symbolic link, then serves an IOCTL that opens a process by ID and a file by NT path, reporting both results. Link with `Wdmsec.lib`; the SDDL grants full access to `LocalSystem` and `Builtin\Administrators` only, so a non-elevated client receives `STATUS_ACCESS_DENIED` at `CreateFileW` - which is the intended behaviour for a privileged control device.

```c
/* objmgr_demo.c - object manager recipes with complete cleanup */
#include <initguid.h>
#include <ntifs.h>
#include <wdmsec.h>

#define OMDEMO_DEVICE_NAME   L"\\Device\\ObjDemo"
#define OMDEMO_SYMLINK_NAME  L"\\DosDevices\\ObjDemo"
#define OMDEMO_MAX_PATH      260

/* Custom setup class GUID: never reuse a class GUID owned by another driver. */
DEFINE_GUID(GUID_DEVCLASS_OBJDEMO,
            0x2b7c9f10, 0x4a51, 0x4f0e,
            0x9b, 0x31, 0x6d, 0x2a, 0x8c, 0x77, 0x05, 0x11);

#define IOCTL_OMDEMO_QUERY \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _OMDEMO_REQUEST {
    ULONG ProcessId;                    /* 0 = skip the process probe      */
    ULONG PathChars;                    /* 0 = skip the file probe         */
    WCHAR FilePath[OMDEMO_MAX_PATH];    /* NT path, not null-terminated    */
} OMDEMO_REQUEST, *POMDEMO_REQUEST;

typedef struct _OMDEMO_REPLY {
    NTSTATUS ProcessStatus;
    NTSTATUS FileStatus;
    ULONG    ImageNameChars;
    WCHAR    ImageName[16];
    WCHAR    TypeName[16];
    ULONG    FileSizeLow;
    ULONG    FileSizeHigh;
} OMDEMO_REPLY, *POMDEMO_REPLY;

DRIVER_DISPATCH OmDemoCreateClose;
DRIVER_DISPATCH OmDemoDeviceControl;

/* Copy a bounded wide string without depending on ntstrsafe. */
static VOID OmDemoCopyString(_Out_writes_(DestChars) PWSTR Dest,
                             _In_ ULONG DestChars,
                             _In_ PCWSTR Source)
{
    ULONG i = 0;

    while ((i + 1) < DestChars && Source[i] != L'\0') {
        Dest[i] = Source[i];
        i++;
    }
    Dest[i] = L'\0';
}

static PCWSTR OmDemoTypeName(_In_ PVOID Object)
{
    POBJECT_TYPE type = ObGetObjectType(Object);

    if (type == *PsProcessType)    { return L"Process"; }
    if (type == *PsThreadType)     { return L"Thread"; }
    if (type == *IoFileObjectType) { return L"File"; }
    return L"Other";
}

/*
 * Open a process by ID. The handle is a kernel handle, used only to obtain a
 * referenced PEPROCESS, then closed. Caller must ObDereferenceObject(*Process).
 */
static NTSTATUS OmDemoOpenProcessById(_In_ ULONG ProcessId,
                                      _In_ ACCESS_MASK Access,
                                      _Outptr_ PEPROCESS *Process)
{
    CLIENT_ID clientId;
    OBJECT_ATTRIBUTES attributes;
    HANDLE handle = NULL;
    NTSTATUS status;

    *Process = NULL;
    clientId.UniqueProcess = (HANDLE)(ULONG_PTR)ProcessId;
    clientId.UniqueThread = NULL;

    InitializeObjectAttributes(&attributes, NULL, OBJ_KERNEL_HANDLE, NULL, NULL);

    status = ZwOpenProcess(&handle, Access, &attributes, &clientId);
    if (!NT_SUCCESS(status)) {
        return status;                       /* nothing acquired */
    }

    status = ObReferenceObjectByHandle(handle, Access, *PsProcessType,
                                       KernelMode, (PVOID *)Process, NULL);
    ZwClose(handle);                          /* handle is not needed now */
    if (!NT_SUCCESS(status)) {
        *Process = NULL;
        return status;
    }
    return STATUS_SUCCESS;
}

/*
 * Open an existing file by NT path. The handle is a kernel handle; the caller
 * owns it and must ZwClose it.
 */
static NTSTATUS OmDemoOpenFileByNtPath(_In_ PCUNICODE_STRING NtPath,
                                       _In_ ACCESS_MASK Access,
                                       _Out_ PHANDLE FileHandle)
{
    OBJECT_ATTRIBUTES attributes;
    IO_STATUS_BLOCK iosb;
    NTSTATUS status;

    *FileHandle = NULL;
    InitializeObjectAttributes(&attributes, (PUNICODE_STRING)NtPath,
                               OBJ_KERNEL_HANDLE | OBJ_CASE_INSENSITIVE,
                               NULL, NULL);

    status = ZwCreateFile(FileHandle,
                          Access | SYNCHRONIZE,
                          &attributes,
                          &iosb,
                          NULL,                  /* AllocationSize        */
                          FILE_ATTRIBUTE_NORMAL,
                          FILE_SHARE_READ | FILE_SHARE_WRITE | FILE_SHARE_DELETE,
                          FILE_OPEN,
                          FILE_NON_DIRECTORY_FILE | FILE_SYNCHRONOUS_IO_NONALERT,
                          NULL,                  /* EaBuffer              */
                          0);                    /* EaLength              */
    if (!NT_SUCCESS(status)) {
        *FileHandle = NULL;
    }
    return status;
}

static NTSTATUS OmDemoQueryFileSize(_In_ HANDLE FileHandle, _Out_ PULONGLONG Size)
{
    IO_STATUS_BLOCK iosb;
    FILE_STANDARD_INFORMATION info;
    NTSTATUS status;

    status = ZwQueryInformationFile(FileHandle, &iosb, &info, sizeof(info),
                                    FileStandardInformation);
    if (NT_SUCCESS(status)) {
        *Size = (ULONGLONG)info.EndOfFile.QuadPart;
    }
    return status;
}

static NTSTATUS OmDemoCreateDevice(_In_ PDRIVER_OBJECT DriverObject,
                                   _Outptr_ PDEVICE_OBJECT *DeviceObject)
{
    UNICODE_STRING deviceName;
    UNICODE_STRING symlinkName;
    UNICODE_STRING sddl = RTL_CONSTANT_STRING(L"D:P(A;;GA;;;SY)(A;;GA;;;BA)");
    NTSTATUS status;

    *DeviceObject = NULL;
    RtlInitUnicodeString(&deviceName, OMDEMO_DEVICE_NAME);
    RtlInitUnicodeString(&symlinkName, OMDEMO_SYMLINK_NAME);

    status = IoCreateDeviceSecure(DriverObject,
                                  0,                 /* extension size */
                                  &deviceName,
                                  FILE_DEVICE_UNKNOWN,
                                  FILE_DEVICE_SECURE_OPEN,
                                  FALSE,             /* not exclusive */
                                  &sddl,
                                  &GUID_DEVCLASS_OBJDEMO,
                                  DeviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    status = IoCreateSymbolicLink(&symlinkName, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(*DeviceObject);           /* unwind what succeeded */
        *DeviceObject = NULL;
        return status;
    }
    return STATUS_SUCCESS;
}

NTSTATUS OmDemoCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

NTSTATUS OmDemoDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    OMDEMO_REQUEST local;
    POMDEMO_REPLY reply;
    PEPROCESS process = NULL;
    HANDLE fileHandle = NULL;
    UNICODE_STRING path;
    ULONGLONG fileSize = 0;
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR information = 0;

    UNREFERENCED_PARAMETER(DeviceObject);

    if (stack->Parameters.DeviceIoControl.IoControlCode != IOCTL_OMDEMO_QUERY) {
        goto Complete;
    }
    if (stack->Parameters.DeviceIoControl.InputBufferLength < sizeof(OMDEMO_REQUEST) ||
        stack->Parameters.DeviceIoControl.OutputBufferLength < sizeof(OMDEMO_REPLY)) {
        status = STATUS_BUFFER_TOO_SMALL;
        goto Complete;
    }

    /*
     * METHOD_BUFFERED: SystemBuffer is input and output at the same address.
     * Copy the request out before the reply overwrites it.
     */
    RtlCopyMemory(&local, Irp->AssociatedIrp.SystemBuffer, sizeof(local));
    reply = (POMDEMO_REPLY)Irp->AssociatedIrp.SystemBuffer;
    RtlZeroMemory(reply, sizeof(*reply));

    if (local.ProcessId != 0) {
        reply->ProcessStatus =
            OmDemoOpenProcessById(local.ProcessId,
                                  PROCESS_QUERY_LIMITED_INFORMATION,
                                  &process);
        if (NT_SUCCESS(reply->ProcessStatus)) {
            PCHAR image = PsGetProcessImageFileName(process);
            ULONG i = 0;

            while (image[i] != '\0' && i < (RTL_NUMBER_OF(reply->ImageName) - 1)) {
                reply->ImageName[i] = (WCHAR)image[i];
                i++;
            }
            reply->ImageName[i] = L'\0';
            reply->ImageNameChars = i;
            OmDemoCopyString(reply->TypeName, RTL_NUMBER_OF(reply->TypeName),
                             OmDemoTypeName(process));
        }
    }

    if (local.PathChars != 0) {
        if (local.PathChars > OMDEMO_MAX_PATH) {
            reply->FileStatus = STATUS_INVALID_PARAMETER;
        } else {
            path.Buffer = local.FilePath;
            path.Length = (USHORT)(local.PathChars * sizeof(WCHAR));
            path.MaximumLength = path.Length;

            reply->FileStatus = OmDemoOpenFileByNtPath(&path, FILE_READ_ATTRIBUTES,
                                                      &fileHandle);
            if (NT_SUCCESS(reply->FileStatus)) {
                reply->FileStatus = OmDemoQueryFileSize(fileHandle, &fileSize);
                reply->FileSizeLow = (ULONG)(fileSize & 0xFFFFFFFF);
                reply->FileSizeHigh = (ULONG)(fileSize >> 32);
                ZwClose(fileHandle);
                fileHandle = NULL;
            }
        }
    }

    status = STATUS_SUCCESS;
    information = sizeof(OMDEMO_REPLY);

Complete:
    if (process != NULL) {
        ObDereferenceObject(process);
    }
    if (fileHandle != NULL) {
        ZwClose(fileHandle);
    }

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static VOID OmDemoUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING symlinkName;
    NTSTATUS status;

    RtlInitUnicodeString(&symlinkName, OMDEMO_SYMLINK_NAME);
    status = IoDeleteSymbolicLink(&symlinkName);
    if (!NT_SUCCESS(status) && status != STATUS_OBJECT_NAME_NOT_FOUND) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "ObjDemo: IoDeleteSymbolicLink failed 0x%08X\n", status);
    }

    if (DriverObject->DeviceObject != NULL) {
        IoDeleteDevice(DriverObject->DeviceObject);
    }
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    PDEVICE_OBJECT deviceObject = NULL;

    UNREFERENCED_PARAMETER(RegistryPath);

    status = OmDemoCreateDevice(DriverObject, &deviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE]         = OmDemoCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]          = OmDemoCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = OmDemoDeviceControl;
    DriverObject->DriverUnload                         = OmDemoUnload;

    return STATUS_SUCCESS;
}
```

The cleanup contract, stated once so it can be reviewed: `OmDemoOpenProcessById` acquires exactly two things (handle, reference) and releases the handle on both paths; the reference moves to the caller and is released at `Complete`. `OmDemoDeviceControl` acquires a process reference, a file handle and possibly nothing else, and the single `Complete` label releases whichever ones are non-NULL. `OmDemoCreateDevice` deletes the device if the symbolic link fails. `OmDemoUnload` deletes the link before the device, because a link that outlives its target is a dangling name that returns `STATUS_OBJECT_NAME_NOT_FOUND` to every client.

## Variations and advanced use

- **Opening a device instead of a file.** `IoGetDeviceObjectPointer(&name, FILE_READ_DATA, &fileObject, &deviceObject)` returns a referenced device object plus its file object; it internally opens the device by name and adds the reference, so the cleanup is `ObDereferenceObject(fileObject)` (releasing the file object releases the device reference). It is shorter and less error-prone than a hand-built `ZwCreateFile` for this case.
- **Resolving a name you do not have a type for.** `ObReferenceObjectByName` with an explicit `*IoDeviceObjectType` works, but the returned pointer is the device object of the top of the stack only if you then call `IoGetAttachedDeviceReference`. Prefer the typed helper; the untyped call gives no compile-time protection and its declaration lives in `ntifs.h` rather than in the documented WDM surface.
- **Handing a kernel object to user mode.** `ObOpenObjectByPointer` converts a referenced pointer into a handle in the current process's table; combine it with `OBJ_KERNEL_HANDLE` when the handle is meant for kernel use only, and omit the flag only when returning the handle to a client through an IOCTL output buffer. That returned handle is a process handle and dies with the client.
- **Directory-relative opens.** Set `RootDirectory` to an open handle for a directory (a file object for a file-system directory, or an object manager directory handle) and pass a relative name. This avoids re-parsing long absolute paths and is how a driver can constrain an open to a subtree it already validated.
- **Names that must survive reboots.** `\Device` names disappear with the driver. Persistent discovery belongs in device interfaces (`IoRegisterDeviceInterface`) or in `\Registry`, not in a symbolic link that someone can pre-create.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
| --- | --- | --- |
| `STATUS_OBJECT_NAME_NOT_FOUND` (`0xC0000034`) | device created under a different name, link not created, or link deleted before the device | `!object \Device\ObjDemo`, `!object \??\ObjDemo`; check the `IoCreateSymbolicLink` return value |
| `STATUS_OBJECT_NAME_COLLISION` (`0xC0000035`) | a previous load did not delete the name, or another driver owns it | `!object \Device\<name>`; delete the stale link, or add `OBJ_OPENIF` if sharing is intended |
| `STATUS_OBJECT_PATH_NOT_FOUND` (`0xC000003A`) | intermediate directory in the NT path does not exist | verify the volume name with `!object \Device`; `\??\C:\path` resolves through `\GLOBAL??` |
| `STATUS_ACCESS_DENIED` (`0xC0000022`) at open, client sees `ERROR_ACCESS_DENIED` | SDDL excludes the caller, or the requested mask exceeds the granted mask | inspect the device SD (`!object` then `!sd`); open with the least mask |
| `STATUS_INVALID_HANDLE` (`0xC0000008`) | user handle passed where a kernel handle is required, handle closed twice, or wrong handle table | `!handle <value> f` in the owning process; pass `UserMode` to `ObReferenceObjectByHandle` for user handles |
| `STATUS_OBJECT_TYPE_MISMATCH` (`0xC0000024`) | `ObjectType` argument does not match the handle's object | use `*PsProcessType` / `*PsThreadType` / `*IoFileObjectType` correctly |
| `STATUS_SHARING_VIOLATION` (`0xC0000043`) | file already open without the share mode you requested | widen `ShareAccess`, or open with `FILE_READ_ATTRIBUTES` only |
| `STATUS_PROCESS_IS_TERMINATING` (`0xC000010A`) | target process exited between lookup and open | treat as a normal race; re-enumerate rather than retrying the same ID |
| `STATUS_INVALID_CID` (`0xC000000B`) | `CLIENT_ID` fields swapped, or the ID is not a real process/thread | remember IDs are multiples of 4; verify with `!process <pid> 0` |
| `STATUS_PRIVILEGE_NOT_HELD` (`0xC0000061`) | `OBJ_PERMANENT`, or opening a protected process with a strong mask | drop `OBJ_PERMANENT`; use `PROCESS_QUERY_LIMITED_INFORMATION` |
| Bugcheck `0x50 PAGE_FAULT_IN_NONPAGED_AREA` touching a freed object body | object dereferenced one time too many, then used | `!analyze -v`, `!pool <addr>`; look for a missing ownership rule, enable special pool |
| Bugcheck `0xC2 BAD_POOL_CALLER` on a pool block behind an object | double free triggered by an extra `ObDereferenceObject` | `!analyze -v` gives the caller; count references per acquisition |
| Bugcheck `0xC4` subcode `0xF6` under Driver Verifier | user handle used with `AccessMode = KernelMode` | pass `UserMode`, or duplicate into a kernel handle first |
| Bugcheck `0xCE` after unload, with a stale handle or callback | handle or object outlived the driver image | close every handle in `DriverUnload`; check with `!handle 0 0` before unload |
| Handle count grows in `handle.exe -a <name>` or `!handle 0 0` | a failure path returns without `ZwClose` | audit `goto` labels; every acquisition needs one release |

## Safety, IRQL, and cleanup checklist

- Every handle created by the driver carries `OBJ_KERNEL_HANDLE`, and every one is closed with `ZwClose` at PASSIVE_LEVEL before unload.
- Every `ObReferenceObjectByHandle` / `ObReferenceObjectByPointer` / `PsLookup*` call has exactly one matching `ObDereferenceObject` on every path.
- Handles received from user mode are resolved with `AccessMode = UserMode`; they are never passed to a `Zw*` routine and never stored in the device extension.
- Access masks are minimal and specific; `PROCESS_ALL_ACCESS` does not appear in shipping code.
- The device has an explicit SDDL descriptor created through `IoCreateDeviceSecure` with a private class GUID, and `FILE_DEVICE_SECURE_OPEN` is set so the DACL is enforced on every open.
- Symlink names are deleted before the device object, and the device is deleted only after its last handle closes.
- `UNICODE_STRING` lengths are in bytes, exclude the terminator, and are validated against the buffer before any call.
- Names built from user input are bounded, checked for reparse points, and never resolved with `OBJ_FORCE_ACCESS_CHECK` missing when the caller's identity matters.
- No `Nt*` routine is called directly; the `Zw*` wrapper is what keeps `PreviousMode` consistent.
- Driver Verifier runs with handle tracing (`verifier /standard` plus `!htrace`) during stress, not only at first load.

## Verification

Install and start, then exercise the IOCTL and watch the counts:

```bat
sc create ObjDemo type= kernel binPath= C:\drv\objmgr_demo.sys
sc start ObjDemo
```

In the debugger, before and after a stress loop that issues hundreds of IOCTLs:

```
lkd> !object \Device\ObjDemo
Object: ffffa70b`2d10b610  Type: (ffffa70b`2a0c3d80) Device
    HandleCount: 0  PointerCount: 3
lkd> !object \??\ObjDemo
SymbolicLink: \??\ObjDemo -> \Device\ObjDemo
lkd> !handle 0 0
```

`HandleCount` must return to its pre-test value and `PointerCount` must not drift upward; a rising pointer count with a stable handle count is a leaked `ObReferenceObjectByHandle` reference. In the test process, `handle.exe -a ObjDemo` (Sysinternals) must show nothing after the client exits. A successful open of a protected path is confirmed by the client receiving `STATUS_SUCCESS`, while the same client run unelevated must receive `ERROR_ACCESS_DENIED` at `CreateFileW`, proving the SDDL is applied rather than ignored.

For the file path, run the IOCTL with `PathChars` set to `\SystemRoot\System32\ntoskrnl.exe` and expect `FileStatus == STATUS_SUCCESS` with a plausible size; the same request with `\??\C:\no-such-file` must return `STATUS_OBJECT_NAME_NOT_FOUND` (`0xC0000034`) and no handle leak. Enabling `verifier /standard /driver objmgr_demo.sys` and repeating both cases must produce no `0xC4` stop, which is the practical proof that no user handle crossed into kernel mode and no handle was closed twice.
