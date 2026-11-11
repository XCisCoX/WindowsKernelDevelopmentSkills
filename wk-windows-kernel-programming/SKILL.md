---
name: wk-windows-kernel-programming
description: Use when starting or planning any Windows kernel-mode driver task: choosing WDM, KMDF or a minifilter, setting up the toolchain and a test VM, writing DriverEntry and dispatch routines, adding IOCTLs, debugging a bugcheck, or finding the right specialized skill for I/O, synchronization, notifications, filtering, WFP, signing and hardening.
---

# Windows kernel programming: the working library

This is the entry point to a complete, task-oriented library for writing, debugging, and
shipping Windows kernel drivers. Each sibling skill is a self-contained field manual with
APIs, complete code, failure tables, and verification steps. Load this skill first when
the task is broad or when you do not yet know which specialized skill applies; load a
specialized skill directly when the task is already narrow.

The single most important rule: **kernel code has no safety net**. An unhandled
exception, a bad pointer, a leaked reference, or a wrong IRQL does not terminate a
process — it bugchecks the machine or silently corrupts state. Every design decision in
this library follows from that.

## When to use

- You are asked to create, modify, review, debug, sign, install, or explain a Windows
  kernel driver, filter driver, minifilter, WFP callout, or KMDF driver.
- You need to decide between WDM, KMDF, a file-system minifilter, a legacy filter, or a
  WFP callout before writing code.
- You have a bugcheck, a hang, a verifier report, a memory leak, or an IRQL violation to
  diagnose.
- You need a precise API contract (IRQL, ownership, buffer method, cleanup) rather than a
  tutorial.
- You are building a lab: Visual Studio + WDK, a test VM, kernel debugging over network
  or serial, test signing, and deployment.

Do not use this skill for user-mode Windows programming, .NET, or Linux kernel work.
For a single narrow question, jump straight to the specialized skill listed below.

## Non-negotiable rules

1. **Check every NTSTATUS.** `NT_SUCCESS(status)` before using any output. A failed
   allocation, open, or query must unwind everything acquired so far.
2. **Know your IRQL.** At `DISPATCH_LEVEL` (2) you may not page, block, wait on a
   dispatcher object, touch user memory, or call most file/registry APIs. If unsure,
   use the IRQL table in `wk-irql`.
3. **Own every resource explicitly.** Pool, objects, handles, device objects, IRPs,
   contexts, and references all need a named owner and a single release path. Driver
   unload does not clean up for you.
4. **Never trust user-mode input.** Probe or copy exactly once, validate lengths with
   overflow-safe arithmetic, and never dereference a user pointer at raised IRQL.
5. **Complete every IRP exactly once.** Either return the status directly or mark
   pending and complete later; never do both.
6. **Serialize shared state.** Lists, counters, and device state need a lock; the lock
   type is dictated by the highest IRQL that touches the state.
7. **Design for unload and surprise removal from day one.** Every wait, queue, thread,
   timer, and callback must be stoppable and drainable before `DriverUnload` returns.
8. **Build debug, test release, verify with Driver Verifier.** Debug builds hide races;
   release builds hide asserts; verifier finds both.
9. **Snapshot the VM before every load.** A bad driver can prevent boot; recovery is a
   snapshot revert or a WinRE offline registry edit.
10. **Prefer the narrowest mechanism.** Notify callbacks, minifilters, and WFP callouts
    are supported; SSDT/inline hooking is not, and modern Windows actively blocks it.

## Choose the technology

| Requirement | Use | Skill |
|---|---|---|
| Software device, full control of IRPs, legacy compatibility | WDM | `wk-driver-anatomy`, `wk-irp-basics` |
| New device driver, PnP/power handled by framework | KMDF | `wk-kmdf-basics` |
| Observe/block file operations, per-file state | Minifilter | `wk-minifilter-basics` |
| Observe/block network traffic, deep inspection | WFP callout | `wk-wfp-callouts` |
| Filter an existing device stack (keyboard, disk, USB) | Legacy filter | `wk-filter-drivers-legacy` |
| Watch process/thread/image events | Notify routines | `wk-process-notifications`, `wk-thread-image-notifications` |
| Watch handle opens/duplicates | Object callbacks | `wk-object-callbacks` |
| Watch/deny registry changes | Registry callbacks | `wk-registry-callbacks` |
| No kernel code is strictly required | User-mode service + ETW/WMI | Reconsider the design |

If the task can be solved in user mode, solve it in user mode. Kernel code is a
privilege-escalation surface and a support burden.

## Task router

| Task | Load, in order |
|---|---|
| Learn the platform | `wk-windows-architecture` → `wk-object-manager` |
| Set up tools and a VM | `wk-driver-dev-environment` → `wk-driver-build-deploy` |
| Write the first driver | `wk-hello-kernel-driver` → `wk-driver-anatomy` |
| Write kernel C safely | `wk-kernel-programming-rules` → `wk-kernel-strings` → `wk-kernel-memory-pools` |
| Create devices and names | `wk-device-objects` → `wk-object-manager` |
| Add an IOCTL interface | `wk-ioctl-design` → `wk-driver-client-protocol` |
| Handle read/write buffers | `wk-irp-buffering` → `wk-irp-basics` |
| Queue or cancel requests | `wk-irp-queues-cancel` |
| Synchronize | `wk-synchronization` → `wk-spinlocks` → `wk-irql` |
| Defer work | `wk-dpcs` → `wk-work-items` → `wk-kernel-threads` |
| Catch faults / probe user memory | `wk-kernel-exceptions` |
| Debug live | `wk-kernel-debugging` → `wk-windbg-fundamentals` |
| Analyze a crash | `wk-crash-dump-analysis` |
| Add logging | `wk-kernel-tracing` |
| Observe processes | `wk-process-notifications` → `wk-thread-image-notifications` |
| Protect processes | `wk-object-callbacks` |
| Store configuration | `wk-registry-programming` → `wk-registry-callbacks` |
| Time-based work | `wk-kernel-timers` |
| Lookup structures | `wk-generic-tables` → `wk-kernel-linked-lists` |
| Write a minifilter | `wk-minifilter-basics` → `wk-minifilter-operations` → `wk-file-names` |
| Per-file state in a filter | `wk-minifilter-contexts` |
| File I/O from a filter | `wk-minifilter-io` → `wk-file-names` |
| Talk to a filter from user mode | `wk-minifilter-communication` |
| Write a WFP callout | `wk-wfp-basics` → `wk-wfp-callouts` |
| Write a KMDF driver | `wk-kmdf-basics` → `wk-kmdf-ioctl` → `wk-kmdf-pnp-power` → `wk-kmdf-install` |
| Sign and deploy | `wk-driver-signing` → `wk-driver-build-deploy` |
| Find bugs pre-ship | `wk-driver-verifier` → `wk-driver-testing` |
| Harden / threat-model | `wk-kernel-security` → `wk-kernel-programming-rules` |

## Canonical driver skeleton

Every WDM driver follows this shape. Specialized skills add dispatch routines and
teardown, but the ordering and error handling below are invariant.

```c
#include <ntddk.h>

#define DEVICE_NAME     L"\\Device\\Example"
#define SYMLINK_NAME    L"\\DosDevices\\Example"
#define POOL_TAG        'pmxE'

typedef struct _DEVICE_CONTEXT {
    LIST_ENTRY   PendingList;
    KSPIN_LOCK   Lock;
    ULONG        PendingCount;
} DEVICE_CONTEXT, *PDEVICE_CONTEXT;

DRIVER_UNLOAD ExampleUnload;
DRIVER_DISPATCH ExampleCreateClose;

static VOID ExampleUnload(PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT dev = DriverObject->DeviceObject;

    if (dev != NULL) {
        UNICODE_STRING link;
        RtlInitUnicodeString(&link, SYMLINK_NAME);
        IoDeleteSymbolicLink(&link);
        IoDeleteDevice(dev);
    }
}

static NTSTATUS ExampleCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    PDEVICE_OBJECT device = NULL;
    UNICODE_STRING devName, linkName;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, DEVICE_NAME);
    status = IoCreateDevice(DriverObject, sizeof(DEVICE_CONTEXT), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    PDEVICE_CONTEXT ctx = (PDEVICE_CONTEXT)device->DeviceExtension;
    InitializeListHead(&ctx->PendingList);
    KeInitializeSpinLock(&ctx->Lock);

    RtlInitUnicodeString(&linkName, SYMLINK_NAME);
    status = IoCreateSymbolicLink(&linkName, &devName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = ExampleCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]  = ExampleCreateClose;
    DriverObject->DriverUnload = ExampleUnload;

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

Build with the WDK (`msbuild`), sign with a test certificate, install with `sc create`
or `pnputil`, start, exercise from a user-mode client, then stop and uninstall. The
exact commands and expected errors are in `wk-driver-build-deploy`.

## The development loop

1. **Design the interface first.** Device name, IOCTL codes, buffer method, input/output
   structs, version field, access mask, and the exact failure statuses.
2. **Write the skeleton.** DriverEntry, device + symlink, create/close, unload. Build,
   install, and open it from user mode before adding features.
3. **Add one dispatch routine at a time.** After each, build, deploy, run the client,
   and check DebugView/ETW plus `!drvobj`.
4. **Add synchronization and cancellation before stress.** Pending IRPs without cancel
   support are the most common source of unload hangs.
5. **Enable Driver Verifier early** (`verifier /standard /driver your.sys`) with special
   pool and forced IRQL checking, then run the stress client.
6. **Test teardown.** Disable/enable the device, stop the service, unload the driver, and
   unload while I/O is in flight (it must fail cleanly, not hang).
7. **Analyze every bugcheck** with `!analyze -v` and the stop-code table in
   `wk-crash-dump-analysis`; fix the class of bug, not the symptom.
8. **Only then** optimize with `wk-kernel-performance`.

## Cross-cutting invariants

| Invariant | Consequence of violation | Skill |
|---|---|---|
| No pageable code/data at IRQL ≥ DISPATCH | 0xA / 0xD1 bugcheck | `wk-irql` |
| No blocking wait at IRQL ≥ DISPATCH | 0x1E / hang | `wk-synchronization` |
| Every IRP completed once | 0x8E / 0x20 / hang | `wk-irp-basics` |
| Pending requires `IoMarkIrpPending` + `STATUS_PENDING` | lost completion / 0x9F | `wk-irp-queues-cancel` |
| References balanced | use-after-free / 0x50 / 0x139 | `wk-object-manager` |
| Pool freed with the same tag and once | 0xC2 / 0x19 | `wk-kernel-memory-pools` |
| Device removed only after handles close | 0xCE / 0x9F | `wk-device-objects` |
| Callbacks unregistered before unload returns | 0x9F / 0x109 | `wk-process-notifications` |
| User pointers probed and copied once | 0x50 / privilege escalation | `wk-kernel-exceptions` |
| No C++ exceptions/RTTI/STL | compile or runtime failure | `wk-kernel-programming-rules` |

## Failure modes and diagnosis

| Symptom | First checks | Skill |
|---|---|---|
| Driver will not load (577/1275/52F) | signature, test signing, architecture, INF/catalog | `wk-driver-signing` |
| Device shows code 28/39/10 | INF hardware ID, service key, KMDF version, `setupapi.dev.log` | `wk-kmdf-install` |
| Immediate bugcheck on load | unload path, DriverEntry unwind, section alignment, imports | `wk-crash-dump-analysis` |
| 0xA / 0xD1 at random times | paged code at DISPATCH, user pointer at high IRQL, lock misuse | `wk-irql`, `wk-spinlocks` |
| 0x9F on unload | pending IRP, thread, timer, work item, callback still active | `wk-irp-queues-cancel`, `wk-work-items` |
| 0xCE on unload | device object still referenced/open | `wk-device-objects` |
| 0xC2 / 0x19 | double free, wrong pool tag, pool overflow | `wk-kernel-memory-pools` |
| 0x50 / 0x139 | use-after-free, stale object reference, race | `wk-object-manager`, `wk-kernel-exceptions` |
| Hang, no bugcheck | deadlock, blocked unload, DPC/IRQL inversion | `wk-crash-dump-analysis`, `wk-synchronization` |
| IOCTL returns garbage | wrong buffer method, size mismatch, missing probe | `wk-ioctl-design`, `wk-irp-buffering` |
| Filter does not see operations | altitude, instance attachment, layer/weight | `wk-minifilter-basics`, `wk-wfp-basics` |
| Verifier bugcheck 0xC4 | read the subcode and parameters, then `!verifier 0x80` | `wk-driver-verifier` |
| Machine boot-loops after install | Safe Mode, last known good, offline registry, VM snapshot | `wk-driver-build-deploy` |

## Learning paths

**First driver (2-3 days).** `wk-windows-architecture` → `wk-driver-dev-environment` →
`wk-hello-kernel-driver` → `wk-kernel-programming-rules` → `wk-device-objects` →
`wk-driver-anatomy` → `wk-windbg-fundamentals`.

**I/O and production quality (1 week).** `wk-irp-basics` → `wk-irp-buffering` →
`wk-ioctl-design` → `wk-driver-client-protocol` → `wk-irp-queues-cancel` →
`wk-synchronization` → `wk-kernel-debugging` → `wk-driver-verifier`.

**Security product (2 weeks).** `wk-process-notifications` →
`wk-thread-image-notifications` → `wk-object-callbacks` → `wk-registry-callbacks` →
`wk-minifilter-basics` → `wk-minifilter-operations` → `wk-minifilter-contexts` →
`wk-minifilter-communication` → `wk-kernel-security`.

**Networking filter.** `wk-wfp-basics` → `wk-wfp-user-mode` → `wk-wfp-callouts` →
`wk-kernel-debugging` → `wk-driver-verifier`.

**Modern device driver.** `wk-kmdf-basics` → `wk-kmdf-ioctl` → `wk-kmdf-pnp-power` →
`wk-kmdf-install` → `wk-driver-signing`.

## Environment facts to establish before coding

- Windows build and architecture of the target (client vs server, x64 vs ARM64).
- WDK/SDK versions installed and whether the project targets Windows 10 or 11.
- Test VM with kernel debugging configured (network or serial), snapshots enabled.
- Test signing state (`bcdedit /enum {current}` → `testsigning`, `nointegritychecks`).
- Symbol path (`_NT_SYMBOL_PATH`) and a local symbol cache.
- Whether Secure Boot, HVCI, or a vulnerable-driver blocklist is active; they change
  what can load and what hooking techniques fail.
- The exact hardware ID or device interface GUID the driver binds to.

## Glossary

| Term | Meaning |
|---|---|
| IRP | I/O request packet; the unit of I/O work |
| IRQL | Interrupt request level; execution priority/masking level |
| DPC | Deferred procedure call; runs at DISPATCH after an ISR |
| APC | Asynchronous procedure call; runs in a thread context |
| MDL | Memory descriptor list; describes locked physical pages |
| PDO/FDO/Filter DO | Physical, functional, filter device objects in a stack |
| Altitude | Minifilter ordering value assigned by Microsoft |
| WDF/KMDF | Windows Driver Framework / Kernel-Mode Driver Framework |
| WFP | Windows Filtering Platform |
| HVCI | Hypervisor-protected Code Integrity |
| Test signing | Boot mode allowing test-signed drivers to load |

## Verification of your own work

Before declaring a driver task complete:

- Builds clean at `/W4` (or `/WX`) for the target architecture, Debug and Release.
- Loads and unloads 100 times without a leak (`!poolused`, handle counts stable).
- Passes the stress client under Driver Verifier with special pool and forced IRQL.
- Survives disable/enable, surprise removal (if PnP), and unload with I/O in flight.
- Has no unhandled IRP, no pending IRP left at unload, and no callback registered after
  unload.
- Logs enough (ETW/WPP) to diagnose a field failure without a debugger.
- Has a documented install, uninstall, and recovery procedure (including Safe Mode).
