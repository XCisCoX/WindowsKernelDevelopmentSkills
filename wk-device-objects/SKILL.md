---
name: wk-device-objects
description: Use when creating, naming, securing, enumerating, or tearing down Windows device objects: DRIVER_OBJECT and DEVICE_OBJECT field layouts, IoCreateDevice and IoCreateDeviceSecure, DEVICE_TYPE values, FILE_* characteristics, DO_* flags including DO_DEVICE_INITIALIZING, the \Device versus \DosDevices versus \?? namespace, SDDL access control, device enumeration, and leak-free deletion.
---

# Device objects: the endpoints a client can actually open

A driver object is not something a client can talk to; the endpoints are device objects.
Each `DEVICE_OBJECT` owns a name in the object namespace, a chunk of non-paged extension
memory, a set of flags that decide buffering and security, and a reference count that the
I/O manager maintains while handles are open. The lifetime rules are strict: create the
device, finish initializing it, clear `DO_DEVICE_INITIALIZING`, publish a symbolic link
only when the device is ready, and on teardown remove the link, wait for handles to drain,
then delete the device. The rule that prevents most of the related crashes: **a device
object is deleted exactly once, after the last open handle is closed and after its
symbolic link can no longer produce new opens.**

## When to use

- You are creating a device object, a device extension, a symbolic link, or a device
  interface, and need the exact flags and call order.
- A client cannot open `\\.\Name`, or opens it without the access rights you expected.
- You must choose between `IoCreateDevice` and `IoCreateDeviceSecure`, or write an SDDL
  descriptor for a device.
- You are implementing `DriverUnload` and must delete devices without racing open handles.
- You suspect a device object leak: memory grows, names collide, or `!object \Device`
  keeps growing across load/unload cycles.
- You need to enumerate the devices a driver owns, in the kernel or from user mode.
- A driver bugchecks with 0xCE or 0x9F during unload or a power transition.

Do not use this skill for IRP handling after an open succeeds; see `wk-irp-basics` and
`wk-irp-buffering`. For the user-mode side of the path, see `wk-driver-client-protocol`.

## Prerequisites and related skills

- **Headers and libs:** `<ntddk.h>` declares the device APIs and the `FILE_DEVICE_*`
  constants. `IoCreateDeviceSecure` additionally needs `wdmsec.lib` and `<wdmsec.h>`.
  Device class GUIDs come from `<devguid.h>`.
- **IRQL:** `IoCreateDevice`, `IoCreateDeviceSecure`, `IoCreateSymbolicLink`,
  `IoDeleteSymbolicLink`, and `IoDeleteDevice` all require `PASSIVE_LEVEL`.
- **Context:** device creation normally happens in `DriverEntry` or `AddDevice`; creation
  in an IOCTL handler is legal at `PASSIVE_LEVEL` but must still clear
  `DO_DEVICE_INITIALIZING`.
- **Related skills:** `wk-driver-anatomy` (devices in a working driver),
  `wk-object-manager` (names, handles, `OBJECT_ATTRIBUTES`), `wk-object-manager` and
  `wk-driver-client-protocol` (opening the device from user mode), `wk-kernel-security`
  (device ACLs as an attack surface), `wk-filter-drivers-legacy` (attaching to a stack).

## Mental model

```text
   DRIVER_OBJECT (one per loaded image)
   +---------------------------------+
   | DeviceObject  ------------------+--> DEVICE_OBJECT A --NextDevice--> DEVICE_OBJECT B --NextDevice--> NULL
   | MajorFunction[IRP_MJ_*]         |          |                                |
   | DriverUnload / DriverStartIo    |          | DeviceExtension (non-paged)     | DeviceExtension
   +---------------------------------+          | Flags, Characteristics         |
                                                | StackSize, AlignmentRequirement|
                                                v                                v
                                        \Device\Sample0                   \Device\Sample1
                                                ^                                ^
                                                | target                         | target
                                        \DosDevices\Sample0              \DosDevices\Sample1
                                        ( == \??\Sample0 )               ( == \??\Sample1 )
                                                ^
                                                | CreateFileW(L"\\\\.\\Sample0")
                                        FILE_OBJECT  (one per open handle)
```

Four facts follow from that picture and drive every rule in this skill:

1. **The driver object is not an endpoint.** It is the registration record: the dispatch
   array, the unload routine, and the head of the device list.
2. **Device objects form a singly linked list, newest first.** `IoCreateDevice` inserts at
   the head, so `DriverObject->DeviceObject` is the most recently created device.
3. **The extension is part of the device allocation.** It is `DeviceExtensionSize` bytes
   of non-paged memory, zeroed by `IoCreateDevice`, valid until `IoDeleteDevice` completes.
4. **The symbolic link is a separate object with its own lifetime.** Deleting a device
   does not delete its link, and deleting a link does not delete the device. Both must be
   handled, in the right order.

### Namespace layers

| Path | Visible to | Meaning |
|---|---|---|
| `\Device\Name` | Kernel only | The device object's own name in the object namespace |
| `\DosDevices\Name` | Kernel creates links here | Legacy spelling that resolves to `\??\Name` |
| `\??\Name` | Native user mode and kernel | The per-process device map entry for a DOS name |
| `\GLOBAL??\Name` | Object manager | Where global DOS device names actually live |
| `\\.\Name` | Win32 `CreateFileW` | Win32 spelling that maps to `\??\Name` |

A client that can reach `\??\Name` reaches the device object only through the link; the
link's target is the `\Device\...` name. The link is what makes a Win32 path possible, and
the link's presence or absence is independent of the device object's security descriptor.

## Key APIs and structures

### `DRIVER_OBJECT` fields that matter

| Field | Type | Meaning and contract |
|---|---|---|
| `Type`, `Size` | `CSHORT` | Object type and structure size; set by the kernel |
| `DeviceObject` | `PDEVICE_OBJECT` | Head of the driver's device list; newest device first |
| `Flags` | `ULONG` | Driver flags; do not modify |
| `DriverStart`, `DriverSize` | `PVOID`, `ULONG` | Loaded image base and size; useful in logging |
| `DriverExtension` | `PDRIVER_EXTENSION` | Holds `AddDevice`, `DriverObject`, and `ServiceKeyName` |
| `DriverName` | `UNICODE_STRING` | Driver name as registered with the object manager |
| `HardwareDatabase` | `PUNICODE_STRING` | `\Registry\Machine\HARDWARE` |
| `FastIoDispatch` | `PFAST_IO_DISPATCH` | Optional fast I/O table; file-system drivers only |
| `DriverInit` | `PDRIVER_INITIALIZE` | The `DriverEntry` address recorded by the loader |
| `DriverStartIo` | `PDRIVER_STARTIO` | Optional `StartIo` routine for queued IRPs |
| `DriverUnload` | `PDRIVER_UNLOAD` | NULL means the driver cannot be stopped |
| `MajorFunction[]` | `PDRIVER_DISPATCH[IRP_MJ_MAXIMUM_FUNCTION + 1]` | Dispatch table; every entry defaults to `IopInvalidDeviceRequest` |

### `DEVICE_OBJECT` fields that matter

| Field | Type | Meaning and contract |
|---|---|---|
| `Type`, `Size` | `CSHORT`, `USHORT` | Object type (`IO_TYPE_DEVICE`) and structure size |
| `ReferenceCount` | `LONG` | Open handles plus explicit references; `IoDeleteDevice` waits for it to reach zero |
| `DriverObject` | `PDRIVER_OBJECT` | Owning driver; set at creation |
| `NextDevice` | `PDEVICE_OBJECT` | Next device in the driver's list, or NULL |
| `AttachedDevice` | `PDEVICE_OBJECT` | Device above this one in a stack; NULL for a standalone device |
| `CurrentIrp` | `PIRP` | IRP currently being processed, when the driver sets it |
| `Flags` | `ULONG` | `DO_*` flags; see the flag matrix |
| `Characteristics` | `ULONG` | `FILE_*` device characteristics; see the characteristics matrix |
| `DeviceType` | `DEVICE_TYPE` (`ULONG`) | Category reported to the I/O manager; see the device type matrix |
| `StackSize` | `CHAR` | Number of I/O stack locations; 1 for a standalone device, lower device + 1 for a filter |
| `AlignmentRequirement` | `ULONG` | Required buffer alignment for DMA (`0`, `1`, `3`, `7`, ...) |
| `DeviceExtension` | `PVOID` | Your non-paged context, `DeviceExtensionSize` bytes, zeroed at creation |
| `DeviceObjectExtension` | `PVOID` | I/O manager bookkeeping; opaque, never touch |
| `SecurityDescriptor` | `PSECURITY_DESCRIPTOR` | Descriptor applied by the I/O manager when `FILE_DEVICE_SECURE_OPEN` is set |
| `Vpb` | `PVPB` | Volume parameter block; file-system volumes only |
| `DeviceQueue` | `KDEVICE_QUEUE` | Built-in device queue used by `IoStartPacket` |
| `Dpc` | `KDPC` | Built-in DPC used by buffered I/O completion |
| `ActiveThreadCount` | `ULONG` | Threads currently blocked on this device |
| `SectorSize` | `USHORT` | Logical sector size advertised to the I/O manager |
| `Timer` | `PIO_TIMER` | Optional `IoTimer` context |

Two extension facts worth memorizing: the extension arrives **zeroed**, so explicit
`RtlZeroMemory` is redundant but harmless; and it is **non-paged**, so it is legal to touch
at `DISPATCH_LEVEL`, unlike any paged buffer you store a pointer to inside it.

### Creation, linking, and deletion APIs

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `IoCreateDevice(DriverObject, ExtensionSize, DeviceName, DeviceType, Characteristics, Exclusive, &DeviceObject)` | Allocate and initialize a device object | `PASSIVE_LEVEL` | Name may be NULL for an unnamed device; sets `DO_DEVICE_INITIALIZING`; `StackSize` starts at 1 |
| `IoCreateDeviceSecure(DriverObject, ExtensionSize, DeviceName, DeviceType, Characteristics, Exclusive, DefaultSDDLString, DeviceClassGuid, &DeviceObject)` | Same, with an SDDL-derived descriptor | `PASSIVE_LEVEL` | Requires `wdmsec.lib`; the class GUID identifies the device setup class |
| `IoDeleteDevice(DeviceObject)` | Remove the device from the list and drop its creation reference | `PASSIVE_LEVEL` | Unlinks from `DriverObject->DeviceObject`; the object is freed when the last reference goes |
| `IoCreateSymbolicLink(&LinkName, &DeviceName)` | Publish a DOS name for the device | `PASSIVE_LEVEL` | Does not verify that the target exists; duplicate name fails |
| `IoDeleteSymbolicLink(&LinkName)` | Remove the DOS name | `PASSIVE_LEVEL` | Best-effort; check the status and log it |
| `IoGetDeviceObjectPointer(&Name, DesiredAccess, &FileObject, &DeviceObject)` | Resolve a name to a device object with a reference | `PASSIVE_LEVEL` | Returns a referenced file object; dereference it, not the device |
| `IoEnumerateDeviceObjectList(DriverObject, List, ListBytes, &Count)` | Snapshot the driver's device list with references | `PASSIVE_LEVEL` | Dereference each returned device after use |
| `IoAttachDeviceToDeviceStack(SourceDevice, TargetDevice)` | Insert a filter device above a target | `PASSIVE_LEVEL` | Returns the device below; set `StackSize` and copy flags first |

### Device types

| Constant | Value | Typical use |
|---|---|---|
| `FILE_DEVICE_BEEP` | 0x00000001 | Simple notification device |
| `FILE_DEVICE_CONTROLLER` | 0x00000004 | Bus or DMA controller |
| `FILE_DEVICE_DISK` | 0x00000007 | Disk device |
| `FILE_DEVICE_FILE_SYSTEM` | 0x00000009 | File-system filter or recognizer |
| `FILE_DEVICE_NAMED_PIPE` | 0x00000011 | Named pipe |
| `FILE_DEVICE_NETWORK` | 0x00000012 | Network protocol |
| `FILE_DEVICE_NETWORK_FILE_SYSTEM` | 0x00000014 | Network redirector |
| `FILE_DEVICE_NULL` | 0x00000015 | Null device |
| `FILE_DEVICE_SERIAL_PORT` | 0x0000001B | Serial port |
| `FILE_DEVICE_UNKNOWN` | 0x00000022 | Software device with a custom protocol |
| `FILE_DEVICE_MODEM` | 0x0000002B | Modem |
| `FILE_DEVICE_MASS_STORAGE` | 0x0000002D | Mass-storage device |
| `FILE_DEVICE_ACPI` | 0x00000032 | ACPI |
| `FILE_DEVICE_VMBUS` | 0x0000003E | Virtual machine bus |

Vendor-defined types occupy 0x8000-0xFFFF. The type is reported to setup and to tools; it
does not restrict which IRPs the driver receives. Pick the closest existing type, and use
`FILE_DEVICE_UNKNOWN` for a protocol that fits nothing else.

### Device characteristics (`Characteristics` argument)

| Constant | Value | Meaning |
|---|---|---|
| `FILE_REMOVABLE_MEDIA` | 0x00000001 | Media can be changed |
| `FILE_READ_ONLY_DEVICE` | 0x00000002 | Device cannot be written |
| `FILE_FLOPPY_DISKETTE` | 0x00000004 | Floppy-class device |
| `FILE_WRITE_ONCE_MEDIA` | 0x00000008 | Write-once media |
| `FILE_REMOTE_DEVICE` | 0x00000010 | Device is remote |
| `FILE_DEVICE_IS_MOUNTED` | 0x00000020 | Volume is mounted |
| `FILE_VIRTUAL_VOLUME` | 0x00000040 | Virtual volume |
| `FILE_AUTOGENERATED_DEVICE_NAME` | 0x00000080 | The I/O manager generates the name; do not pass a name |
| `FILE_DEVICE_SECURE_OPEN` | 0x00000100 | Apply the device descriptor to every open, not just the first |

For a software device, `FILE_DEVICE_SECURE_OPEN` is the one that matters: without it the
device object's descriptor is not consulted for each open, and any caller that can reach
the name can open a handle.

### Device flags (`DEVICE_OBJECT.Flags`)

| Flag | Value | Meaning and rules |
|---|---|---|
| `DO_BUFFERED_IO` | 0x00000004 | I/O manager copies data through a system buffer; use this or direct I/O, not both |
| `DO_EXCLUSIVE` | 0x00000008 | Only one open at a time; later opens fail with `STATUS_ACCESS_DENIED` |
| `DO_DIRECT_IO` | 0x00000010 | I/O manager locks pages into an MDL; mutually exclusive with buffered I/O |
| `DO_MAP_IO_BUFFER` | 0x00000020 | Legacy mapping behavior; not for new drivers |
| `DO_DEVICE_INITIALIZING` | 0x00000080 | Set by `IoCreateDevice`; while set the device cannot be opened. Cleared automatically after `DriverEntry` returns, and **must be cleared by hand** for devices created later |
| `DO_POWER_PAGABLE` | 0x00000200 | Driver handles power IRPs at `PASSIVE_LEVEL`; never set it if the driver touches pageable code in power paths |
| `DO_POWER_INRUSH` | 0x00000400 | Device needs inrush current at power-on; hardware drivers only |

### Security notes

- **Prefer `IoCreateDeviceSecure` for anything user mode can open.** It takes a default
  SDDL string and a device setup class GUID, and stores the resulting descriptor on the
  device object where the I/O manager applies it.
- **A minimal descriptor** for an administrative interface is
  `L"D:P(A;;GA;;;SY)(A;;GA;;;BA)"`: a protected DACL granting generic-all to LocalSystem
  and to Builtin Administrators, and nothing to anyone else.
- **Adding interactive users** means adding a specific ACE, for example
  `(A;;GRGW;;;IU)` for authenticated users with read/write access, rather than opening the
  device to `WD` (Everyone).
- **`FILE_DEVICE_SECURE_OPEN` is not optional for named devices.** Set it in the
  `Characteristics` argument at creation; assigning `DeviceObject->SecurityDescriptor`
  afterwards is not equivalent and the open path may already have cached what it needs.
- **The name is a security boundary too.** A link in `\DosDevices` is visible to every
  session unless you create it per-session, so treat the name as public and the descriptor
  as the control.
- **Never trust the client because it opened the device.** Access checks happen at open
  time; every IOCTL still validates lengths, pointers, and values.

## Step-by-step

1. **Pick the type, characteristics, and buffering model before writing code.** Device type
   from the closest match, `FILE_DEVICE_SECURE_OPEN` always, and exactly one of
   `DO_BUFFERED_IO` or `DO_DIRECT_IO`. Check: the client and the driver agree on the method
   encoded in each `CTL_CODE`.
2. **Define the extension structure.** Put all per-device state there, never in globals.
   Check: no pointer in the extension refers to paged memory that a raised-IRQL path uses.
3. **Set `MajorFunction[]` and `DriverUnload` before creating the first device.** Check: a
   client cannot open a device whose dispatch array is still default.
4. **Call `IoCreateDevice` with a unique, fully qualified name.** Check: the status is
   `STATUS_SUCCESS`; a collision means another instance owns the name.
5. **Initialize the extension completely, then set buffering flags, `StackSize`, and
   `AlignmentRequirement`**, and clear `DO_DEVICE_INITIALIZING` for every device created
   after `DriverEntry`. Check: `StackSize` is 1 standalone and lower device + 1 for a
   filter, and a client open succeeds.
7. **Create the symbolic link only after the device is fully initialized.** Link name in
   `\DosDevices`, target name in `\Device`. Check: `IoCreateSymbolicLink` returns success;
   on failure, delete the device and fail the load.
8. **Unwind every failure in reverse order.** A failed second device must delete the first
   device and its link, because `DriverUnload` is not called when `DriverEntry` fails.
9. **On unload: delete every link first, then every device.** Check: no link survives to
   route a new open to a device that is being deleted.
10. **Verify with the debugger and with a client** after each load/unload cycle. Check:
    `!object \Device` and `!object \GLOBAL??` return to their pre-load contents, and the
    device count in `!drvobj` is zero after unload.

## Complete example

A driver that creates two devices, publishes two links, keeps per-device state in the
extension, and tears everything down through one helper used by both the failure path and
unload.

```c
/* DevObjDemo.c - two devices, two links, one teardown path. */
#include <ntddk.h>

#define DEVOJBJ_TAG         'jbVD'      /* reads as "DVbj" in a pool dump */
#define DEVOJBJ_COUNT       2
#define DEVOJBJ_DEVICE_TYPE 0x8335

#define DEVOJBJ_IOCTL_INDEX \
    CTL_CODE(DEVOJBJ_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

static const PWSTR g_DeviceNames[DEVOJBJ_COUNT] = {
    L"\\Device\\DevObjDemo0",
    L"\\Device\\DevObjDemo1"
};

static const PWSTR g_LinkNames[DEVOJBJ_COUNT] = {
    L"\\DosDevices\\DevObjDemo0",
    L"\\DosDevices\\DevObjDemo1"
};

typedef struct _DEVOJBJ_CONTEXT {
    ULONG Index;          /* which entry of g_DeviceNames this device is */
    ULONG OpenCount;      /* handles currently open */
} DEVOJBJ_CONTEXT, *PDEVOJBJ_CONTEXT;

DRIVER_INITIALIZE DriverEntry;
DRIVER_UNLOAD DevObjUnload;
DRIVER_DISPATCH DevObjCreateClose;
DRIVER_DISPATCH DevObjDeviceControl;

#ifdef ALLOC_PRAGMA
#pragma alloc_text(INIT, DriverEntry)
#pragma alloc_text(PAGE, DevObjUnload)
#endif

static VOID
DevObjTearDown(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT device;

    /* IoDeleteDevice unlinks the device, so re-reading the head terminates. */
    while ((device = DriverObject->DeviceObject) != NULL) {
        PDEVOJBJ_CONTEXT context = (PDEVOJBJ_CONTEXT)device->DeviceExtension;

        if (context->Index < DEVOJBJ_COUNT) {
            UNICODE_STRING linkName;
            NTSTATUS status;

            RtlInitUnicodeString(&linkName, g_LinkNames[context->Index]);
            status = IoDeleteSymbolicLink(&linkName);
            if (!NT_SUCCESS(status)) {
                DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_WARNING_LEVEL,
                           "DevObjDemo: link removal failed 0x%08X\n", status);
            }
        }
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "DevObjDemo: deleting device %u\n", context->Index);
        IoDeleteDevice(device);
    }
}

static NTSTATUS
DevObjCreateOne(_In_ PDRIVER_OBJECT DriverObject, _In_ ULONG Index)
{
    PDEVICE_OBJECT device = NULL;
    PDEVOJBJ_CONTEXT context;
    UNICODE_STRING deviceName;
    UNICODE_STRING linkName;
    NTSTATUS status;

    RtlInitUnicodeString(&deviceName, g_DeviceNames[Index]);

    status = IoCreateDevice(DriverObject,
                            sizeof(DEVOJBJ_CONTEXT),
                            &deviceName,
                            FILE_DEVICE_UNKNOWN,
                            FILE_DEVICE_SECURE_OPEN,
                            FALSE,
                            &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    context = (PDEVOJBJ_CONTEXT)device->DeviceExtension;
    RtlZeroMemory(context, sizeof(*context));
    context->Index = Index;

    /* Buffered I/O: the smallest change to make a byte-oriented protocol work. */
    device->Flags |= DO_BUFFERED_IO;
    device->StackSize = 1;               /* standalone device, no lower driver */
    device->AlignmentRequirement = 0;

    RtlInitUnicodeString(&linkName, g_LinkNames[Index]);
    status = IoCreateSymbolicLink(&linkName, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);          /* link failed: device must not survive */
        return status;
    }

    /* Redundant in DriverEntry, mandatory for devices created later. */
    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}

static NTSTATUS
DevObjCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    PDEVOJBJ_CONTEXT context = (PDEVOJBJ_CONTEXT)DeviceObject->DeviceExtension;

    if (stack->MajorFunction == IRP_MJ_CREATE) {
        InterlockedIncrement((LONG*)&context->OpenCount);
    } else {
        InterlockedDecrement((LONG*)&context->OpenCount);
    }

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS
DevObjDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    PDEVOJBJ_CONTEXT context = (PDEVOJBJ_CONTEXT)DeviceObject->DeviceExtension;
    NTSTATUS status;

    Irp->IoStatus.Information = 0;

    if (stack->Parameters.DeviceIoControl.IoControlCode != DEVOJBJ_IOCTL_INDEX) {
        status = STATUS_INVALID_DEVICE_REQUEST;
    } else if (stack->Parameters.DeviceIoControl.OutputBufferLength < sizeof(ULONG)) {
        status = STATUS_BUFFER_TOO_SMALL;
    } else if (Irp->AssociatedIrp.SystemBuffer == NULL) {
        status = STATUS_INVALID_PARAMETER;
    } else {
        *(ULONG*)Irp->AssociatedIrp.SystemBuffer = context->Index;
        Irp->IoStatus.Information = sizeof(ULONG);
        status = STATUS_SUCCESS;
    }

    Irp->IoStatus.Status = status;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static VOID
DevObjUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PAGED_CODE();

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "DevObjDemo: unload\n");
    DevObjTearDown(DriverObject);
}

NTSTATUS
DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status = STATUS_SUCCESS;
    ULONG index;

    UNREFERENCED_PARAMETER(RegistryPath);

    /* Dispatch array and unload exist before any device or link is published. */
    DriverObject->MajorFunction[IRP_MJ_CREATE] = DevObjCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = DevObjCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = DevObjDeviceControl;
    DriverObject->DriverUnload = DevObjUnload;

    for (index = 0; index < DEVOJBJ_COUNT; index++) {
        status = DevObjCreateOne(DriverObject, index);
        if (!NT_SUCCESS(status)) {
            /* DriverUnload is not called for a failed load: unwind here. */
            DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                       "DevObjDemo: device %u failed (0x%08X)\n", index, status);
            DevObjTearDown(DriverObject);
            return status;
        }
    }

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "DevObjDemo: %u devices created\n", DEVOJBJ_COUNT);
    return STATUS_SUCCESS;
}
```

### Enumerating a driver's device objects

The kernel snapshot below takes references, so every returned device must be dereferenced.
It also shows how to detect the leak that a missing `IoDeleteDevice` produces.

```c
#include <ntintsafe.h>

NTSTATUS
DevObjEnumerate(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT* list = NULL;
    ULONG needed = 0;
    ULONG count = 0;
    ULONG index;
    SIZE_T bytes = 0;
    NTSTATUS status;

    status = IoEnumerateDeviceObjectList(DriverObject, NULL, 0, &needed);
    if (status != STATUS_BUFFER_TOO_SMALL && !NT_SUCCESS(status)) {
        return status;
    }
    if (needed == 0) {
        return STATUS_SUCCESS;
    }
    status = RtlSizeTMult(needed, sizeof(PDEVICE_OBJECT), &bytes);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    list = (PDEVICE_OBJECT*)ExAllocatePool2(POOL_FLAG_PAGED, bytes, DEVOJBJ_TAG);
    if (list == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    status = IoEnumerateDeviceObjectList(DriverObject, list, (ULONG)bytes, &count);
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(list, DEVOJBJ_TAG);
        return status;
    }

    for (index = 0; index < count; index++) {
        PDEVICE_OBJECT device = list[index];

        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "DevObjEnumerate: %p type=0x%X flags=0x%X refs=%ld next=%p\n",
                   device, device->DeviceType, device->Flags,
                   device->ReferenceCount, device->NextDevice);

        ObDereferenceObject(device);     /* the list owns one reference per entry */
    }

    ExFreePoolWithTag(list, DEVOJBJ_TAG);
    return STATUS_SUCCESS;
}
```

From user mode the equivalent inventory comes from `QueryDosDeviceW(NULL, buffer, size)`,
which returns a multi-string of every name in the caller's `\??` directory; filter the
entries by your driver's prefix. From WinDbg, `!object \Device` and `!object \GLOBAL??`
list the same objects without writing any code.

## Variations and advanced use

- **Secure creation with an explicit descriptor.** Replace the `IoCreateDevice` call with
  the following; `wdmsec.lib` must be in the link line and a class GUID must be supplied.

  ```c
  UNICODE_STRING sddl;

  RtlInitUnicodeString(&sddl, L"D:P(A;;GA;;;SY)(A;;GA;;;BA)");
  status = IoCreateDeviceSecure(DriverObject,
                                sizeof(DEVOJBJ_CONTEXT),
                                &deviceName,
                                FILE_DEVICE_UNKNOWN,
                                FILE_DEVICE_SECURE_OPEN,
                                FALSE,
                                &sddl,
                                &GUID_DEVCLASS_UNKNOWN,
                                &device);
  ```

- **Unnamed devices and device interfaces.** Pass `NULL` for the name and let
  `IoRegisterDeviceInterface` publish a GUID-based interface instead of a DOS link. This is
  the correct model for PnP devices: no name to squat on, and clients enumerate by class.

- **Attaching a filter device.** Create with the same `DeviceType`, copy `Flags` and
  `Characteristics` from the target, set `StackSize = TargetDevice->StackSize + 1` and
  `AlignmentRequirement = TargetDevice->AlignmentRequirement` before
  `IoAttachDeviceToDeviceStack`. See `wk-filter-drivers-legacy`.
- **Rundown before deletion.** If opens can race with unload, guard the extension with
  `EX_RUNDOWN_REF` (`ExInitializeRundownProtection`, `ExAcquireRundownProtection`,
  `ExReleaseRundownProtection`, `ExWaitForRundownProtectionRelease`) and delete the link
  before starting the rundown, so no new open can arrive.

## Failure modes and diagnosis

| Symptom | Likely cause | First command or fix |
|---|---|---|
| `IoCreateDevice` returns `STATUS_OBJECT_NAME_COLLISION` (0xC0000035) | Another device object already owns that `\Device\` name | `!object \Device\<Name>`; choose a unique name or delete the stale device |
| `IoCreateSymbolicLink` returns `STATUS_OBJECT_NAME_COLLISION` | A stale link survived a crashed or unloaded instance | `!object \GLOBAL??`; delete the link, then recreate it |
| Client open fails with `STATUS_OBJECT_NAME_NOT_FOUND` (0xC0000034) | Link missing, wrong spelling, or target device deleted | `!object \GLOBAL??`, `!devobj`; verify `\DosDevices\Name` and `\Device\Name` |
| Client open fails with `STATUS_ACCESS_DENIED` (0xC0000022) | Descriptor denies the caller, or `DO_EXCLUSIVE` is set and a handle is already open | Inspect the descriptor via `!devobj`; check the SDDL; check for an existing handle |
| Client open succeeds but every IOCTL returns `STATUS_INVALID_DEVICE_REQUEST` | `MajorFunction[IRP_MJ_DEVICE_CONTROL]` never set | Set the dispatch entry before creating devices |
| Open succeeds, device cannot be opened a second time | `DO_EXCLUSIVE` set at creation | Create with `Exclusive = FALSE`, or accept single-open semantics |
| Bugcheck 0xCE `DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | Pending IRP or thread still references the driver at unload | `!irpfind`, `!stacks 2`; drain before deleting devices |
| Bugcheck 0x9F `DRIVER_POWER_STATE_FAILURE` | Device holds an IRP across a power transition, typically a PnP device | `!irp`, `!devstack`, `!poaction` |
| Bugcheck 0x50 `PAGE_FAULT_IN_NONPAGED_AREA` | Extension used after `IoDeleteDevice`, or a stale pointer stored in the extension | Special pool; `!pool <addr>` on the extension address |
| Device object leak | `IoDeleteDevice` skipped on an error path, or unload never ran | Count `!object \Device` entries before and after a load/unload cycle |
| Symbolic-link leak | Device deleted but the link was left behind | `!object \GLOBAL??`; add `IoDeleteSymbolicLink` to every teardown path |
| Open works from an elevated client but not from a standard user | Descriptor grants only `SY` and `BA` | Add a narrower ACE for the intended principal; do not open the device to `WD` |
| `!drvobj` shows a device after unload returned | A second device list entry was never deleted | Walk `DriverObject->DeviceObject` until NULL, not just the first device |

## Safety, IRQL, and cleanup checklist

- [ ] The device name is unique and fully qualified under `\Device`; the link name is under
      `\DosDevices` and points at it.
- [ ] `FILE_DEVICE_SECURE_OPEN` is set on every named device, and a descriptor is supplied
      through `IoCreateDeviceSecure` unless the default is deliberately acceptable.
- [ ] Exactly one buffering model is selected (`DO_BUFFERED_IO` or `DO_DIRECT_IO`), and the
      client's `CTL_CODE` methods match it.
- [ ] `MajorFunction[]` entries and `DriverUnload` are installed before the first link
      exists.
- [ ] The extension is fully initialized and `DO_DEVICE_INITIALIZING` is cleared before the
      link is published, and it holds no pointer whose lifetime is shorter than the device.
- [ ] `StackSize` and `AlignmentRequirement` are set explicitly for filters; the
      standalone case keeps `StackSize = 1`.
- [ ] Every failure path unwinds in reverse: delete link, delete device, and never rely on
      `DriverUnload` being called for a failed `DriverEntry`.
- [ ] Unload deletes all links first, then all devices, by walking
      `DriverObject->DeviceObject` until it is NULL.
- [ ] No device is deleted while an IRP it owns is still in flight; pending work is drained
      or cancelled first.
- [ ] The device count and link count return to the pre-load baseline after unload,
      verified with `!object`, not by inspection.
- [ ] Access checks at open time are not treated as validation of IOCTL input.

## Verification

```text
:: Build both configurations, then install and start
msbuild DevObjDemo.vcxproj /p:Configuration=Debug /p:Platform=x64 /warnaserror
msbuild DevObjDemo.vcxproj /p:Configuration=Release /p:Platform=x64 /warnaserror
sc create DevObjDemo type= kernel binPath= C:\drivers\DevObjDemo.sys
sc start DevObjDemo
```

Kernel debugger, after the driver loads (addresses are illustrative):

```text
kd> !drvobj DevObjDemo 2
Driver object (ffffa20b`1c2d3e40) is for:
 \Driver\DevObjDemo
DriverUnload:  fffff805`12341120 DevObjDemo!DevObjUnload
Dispatch array: MajorFunction[0..31]
   IRP_MJ_CREATE
   IRP_MJ_CLOSE
   IRP_MJ_DEVICE_CONTROL

kd> !devobj ffffa20b`1c2d3e40
Device object (ffffa20b1c2d3e40) is for:
 \Driver\DevObjDemo DriverObject ffffa20b1c2d3e00
Current Irp 00000000 RefCount 0 Type 00000022 Flags 00000084
DeviceExtension ffffa20b1c2d3e88 DeviceType 00000022
Characteristics 00000100

kd> !object \GLOBAL??
Directory object: ffffa20b1c2d4000  Name: GLOBAL??
  DevObjDemo0 -> SymbolicLink
  DevObjDemo1 -> SymbolicLink

kd> !object \Device\DevObjDemo0
Object: ffffa20b1c2d3e40  Type: (ffffa20b1c2d3f00) Device
```

The `Flags` value `0x84` in that listing is `DO_DEVICE_INITIALIZING` (0x80) plus
`DO_BUFFERED_IO` (0x04); if a device created after `DriverEntry` still shows `0x80` set,
opens will fail until the driver clears it. `Characteristics 0x100` confirms
`FILE_DEVICE_SECURE_OPEN`.

Functional checks:

1. **Open and query.** A client opens `\\.\DevObjDemo0` with read/write access and sends
   `DEVOJBJ_IOCTL_INDEX`; it must receive `0` for device 0 and `1` for device 1.
2. **Second open of an exclusive device.** Not applicable here (`Exclusive = FALSE`), but
   the same test with `TRUE` must return `STATUS_ACCESS_DENIED` on the second handle.
3. **Unload with a handle open.** Close the client first; then `sc stop DevObjDemo` must
   succeed, and `!object \GLOBAL??` must no longer list either link.
4. **Leak cycle.** Run 100 load/unload cycles and compare device counts:

   ```text
   kd> !object \Device
   ```

   The count of `DevObjDemo` objects must be zero after each unload. Any residual entry
   means a device or link survived, and the address gives the object to inspect with
   `!devobj`.

5. **Failure path.** Temporarily run two copies of the driver (or pre-create the link) so
   the second `IoCreateSymbolicLink` fails; `sc start` must fail with the returned status,
   and the first device must be gone from `!object \Device` after the failed load.
