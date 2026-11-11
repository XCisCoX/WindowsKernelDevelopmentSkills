---
name: wk-windows-architecture
description: Use when reasoning about the execution environment a Windows driver runs in - EPROCESS/ETHREAD and PEB/TEB, user versus kernel address space, virtual and physical page-states, kernel stack limits, system calls and previous mode, the executive subsystem map, IRQL as an execution constraint, and the place a driver occupies in a device stack. Supplies the context-classification procedure to run before writing any kernel routine.
---

# Windows architecture for driver authors

Every line of a driver executes inside a specific context: a processor, an IRQL, a thread, and a process. Which APIs are legal, which pointers are valid, and which bugcheck follows a mistake all derive from those four facts rather than from the source line. The model worth carrying: processes own resources, threads execute, the kernel half of the address space is global while the user half is process-relative, and IRQL - not thread priority - is what forbids paging, blocking and waiting. The discipline that prevents most avoidable crashes is to state a routine's IRQL and process context before writing it, and to classify every pointer as kernel, paged, user or invalid. When a crash contradicts the model, the model is wrong and the crash is the evidence.

## When to use

- Deciding whether a routine may call `Zw*`, touch a user buffer, allocate paged pool, or wait on a dispatcher object.
- Explaining a `0x50`, `0xA`, `0xD1`, `0x7F` or `0x133` crash by mapping it back to an address-space or context error.
- Choosing between inline work, a work item, a system thread, a DPC, or completion in an ISR.
- Estimating stack budget, or deciding that a structure must not be a local variable.
- Reading `!process`, `!thread`, `!vm`, `!irql` output and mapping fields to kernel structures.
- Judging what a user handle exposes and what the object manager already validated (deeper coverage in `wk-object-manager`).
- Understanding why a driver must call `Zw*` and never `Nt*` for caller-supplied data.
- Locating a driver within a device stack and predicting which IRQL its dispatch routines see.

Do not use for: per-API IRQL limits and the raise/lower mechanics (`wk-irql`); pool selection and tagging (`wk-kernel-memory-pools`); object and handle lifecycle (`wk-object-manager`); device object and stack construction (`wk-device-objects`); thread creation and attach/detach code (`wk-kernel-threads`).

## Prerequisites and related skills

- Kernel-mode C, WDK 10.0.22621 or later, x64 target unless a row says otherwise; `<ntddk.h>` or `<wdm.h>`.
- Ability to load an unsigned test driver on a disposable VM; a kernel debugger attached if a bugcheck is under investigation.
- All examples assume the reader can already build, sign for test, install and start a driver; if not, `wk-driver-dev-environment` and `wk-driver-build-deploy` cover the toolchain first.
- Related skills, and why: `wk-irql` for the legality matrix per level; `wk-object-manager` for handles and names; `wk-kernel-memory-pools` for allocation choices; `wk-kernel-threads` for `PsCreateSystemThread` and `KeStackAttachProcess`; `wk-kernel-debugging` for the debugger commands referenced here.

## Mental model

### Process owns, thread executes

A process is a management object: it holds an executable image, a private address space (its own page-tables), a primary access token, a handle table, and one or more threads. Its kernel representation is `EPROCESS`, a system-space structure whose first member is the `KPROCESS` (`EPROCESS->Pcb`) that the scheduler actually uses: `DirectoryTableBase` (the CR3 value), affinity mask, priority, and process state. Driver code never dereferences `EPROCESS` fields directly; it calls accessors such as `PsGetProcessId`, `PsGetProcessImageFileName`, `PsGetThreadProcess`, and `PsGetCurrentProcess`. Those return borrowed pointers with no reference taken, so they must be consumed immediately, not cached across a wait.

A thread is the runnable entity: `ETHREAD`, whose first member is the `KTHREAD`. The `KTHREAD` carries the fields a driver indirectly cares about most: `PreviousMode` (see below), the kernel stack pointers (`InitialStack`, `StackLimit`), the APC state, the wait IRQL, and the trap frame of the last user-to-kernel transition. Two extra structures live in the process's user-space half: the `PEB` (image base, loader lists, environment, heap list) and one `TEB` per thread (stack limits, TLS array, last error). They are readable from kernel mode only in the right process context, only at PASSIVE_LEVEL, and only inside `__try`.

Identity is the trap. A process ID is unique only while the process object is alive; the kernel reuses the numeric ID afterwards. A driver that stores a `ULONG` PID and later reopens it may act on a completely different process - including a protected one - which is both a correctness and a security defect. Store a referenced `PEPROCESS` instead, and compare `PsGetProcessId` when a human-readable identity is needed.

### One user half per process, one kernel half for the machine

x64 Windows splits its 256 TB of virtual address space into two 128 TB halves. The user half is per-process and changes on every context switch; the kernel half is identical in every process, which makes kernel addresses absolute and user addresses relative to a process.

```
0x00000000`00000000  +-----------------------------------+
                     | user address space, 128 TB        |  process-relative:
                     | image, heaps, thread stacks,      |  each process sees a
                     | PEB/TEB, mapped DLLs, sections    |  different mapping here
0x00007FFF`FFFFFFFF  +-----------------------------------+
                     | non-canonical hole                |  bits 63:48 do not
                     |                                   |  sign-extend bit 47
0xFFFF8000`00000000  +-----------------------------------+
                     | kernel address space, 128 TB      |  absolute: identical
                     | ntoskrnl, hal, drivers, PTEs,     |  in every process and
                     | non-paged pool, paged pool,       |  inaccessible from user
                     | system cache, kernel stacks,      |  mode
                     | session space                     |
0xFFFFFFFF`FFFFFFFF  +-----------------------------------+
```

Address sizes that matter:

| Platform | User half | Kernel half | Note |
| --- | --- | --- | --- |
| x64, Windows 8.1/Server 2012 R2 and later | 128 TB (`0x7FFF_FFFF_FFFF`) | 128 TB | 48-bit virtual addresses |
| x64, Windows 8/Server 2012 and earlier | 8 TB | 8 TB | 44-bit virtual addresses |
| x86, default | 2 GB | 2 GB | `LARGEADDRESSAWARE` plus `bcdedit /set increaseuserva` lifts user to 3 GB |
| x86 process on x64 (WOW64) | 2 GB, or 4 GB with `LARGEADDRESSAWARE` | shared 64-bit kernel half | a 32-bit driver cannot exist in this process |

The CPU translates through four-level page-tables (PML4, PDPT, PD, PT) with 4 KB pages, 2 MB large pages (one PDE, no PT) and 1 GB huge pages on Windows 10/Server 2016 and later when a 1 GB contiguous allocation exists. Two consequences for drivers: a large mapping can fail purely because physical memory is fragmented, and **large pages are always non-pageable and read/write only**.

Kernel code that receives an address from user mode must never treat it as a kernel pointer. A kernel address has bits 63:48 set; a user address does not. That test is necessary but not sufficient - a user can pass any value - so the authoritative check is `Irp->RequestorMode == UserMode` combined with `ProbeForRead`/`ProbeForWrite` inside `__try`, or better, letting the I/O manager copy through a buffered or direct IRP.

`MmIsAddressValid` answers a different question than most callers think: it reports whether the address is mapped **at this instant**, without validating the whole range, the protection, or the caller's right to touch it, and another thread can unmap the range immediately afterwards. It is a diagnostic, never a security check.

### Page-states: commitment is virtual, residency is physical

Virtual memory has three region states, and physical memory has a separate life cycle. Both matter.

- **Free** - not allocated; any access raises an access violation.
- **Reserved** - the address range is reserved so later allocations cannot take it, but nothing is committed; the CPU still faults on access. Thread stacks use this to keep a contiguous range without committing all of it.
- **Committed** - backed by RAM or by a file (pagefile, image, section); charge is made against the commit limit.

Each 4 KB physical page-frame sits on one of the page-lists: **active/valid** (mapped in at least one working set), **standby** (still holds valid data but was removed from every working set - a fault on it is cheap because no I/O is needed), **modified** (dirty, must be written before reuse), **modified-no-write**, **free** (contains stale data, not zeroed), **zeroed**, **ROM**, and **bad**. The memory manager's working sets decide which pages stay resident; pages that leave a working set go to standby and its PTE becomes a transition PTE. File-backed and section-backed pages use prototype PTEs so that a fault can be resolved from the section rather than from a private copy.

Practical reading for a driver author: the distinction between non-paged and paged pool is exactly the distinction between "always resident" and "resident until the memory manager decides otherwise". **A paged-pool access at DISPATCH_LEVEL or above works until the memory manager trims it, which is why the resulting bugcheck appears hours into a test run and never on the developer's machine.** Allocate with `ExAllocatePool2(POOL_FLAG_NON_PAGED, size, tag)` for anything touched above PASSIVE_LEVEL, and remember that pool is not zeroed unless `POOL_FLAG_ZERO_ALLOCATE` is added.

### The kernel stack is small and does not grow on demand

Every thread has a kernel stack that is resident whenever the thread is running or ready. Defaults are **12 KB on x86 and 24 KB on x64**, ending in a guard page; the user-mode stack is a different object: 1 MB reserved by default, committed lazily, and growable on demand when the guard region is touched. The kernel stack has no such growth path in normal execution.

Consequences that show up as real bugs:

- A single frame with a 4 KB or 16 KB local array consumes a quarter or half of the whole stack. Send large structures through pool memory.
- Deep recursion in a parser or a filesystem walk can exhaust the stack; the resulting double fault cannot be handled on the exhausted stack, so the system bugchecks `0x7F UNEXPECTED_KERNEL_MODE_TRAP` with parameter 1 equal to `8` (double fault).
- A DPC does not run on the interrupted thread's stack; the processor switches to a per-processor DPC stack. Locals in a DPC still consume that smaller stack, so the same rule applies.
- `KeExpandKernelStackAndCallout` (and its `Ex` variant) can run a callout on a larger stack for known-deep paths. It is expensive, PASSIVE_LEVEL only, and must not be used as a substitute for fixing recursion.
- Stack memory is never a place for data that outlives the call: queueing a pointer to a local into a DPC or work item is a guaranteed use-after-free.

### Crossing the boundary: system calls and previous mode

User code cannot open a file or allocate memory; it asks the kernel. The transition is a single instruction and a table lookup.

```
user mode      kernel32!CreateFileW
                    |
               ntdll!NtCreateFile          eax = service number, r10 = args
                    |  syscall (x64) / sysenter (x86)
====================|=====================================================
kernel mode         v
               KiSystemCall64
                    |  load kernel stack from the TSS, build KTRAP_FRAME,
                    |  save the user stack pointer, set
                    |  KTHREAD.PreviousMode = UserMode
                    v
               KiSystemServiceStart -> SSDT[service number]
                    |
                    v
               nt!NtCreateFile
                    |  ProbeForRead/ProbeForWrite because PreviousMode is
                    |  UserMode; handles resolve against the caller's table
                    v
               STATUS_* in eax, return to user mode
```

`PreviousMode` is a per-thread `KPROCESSOR_MODE` value that tells every `Nt*` routine how to interpret the pointers and handles it was given. Two rules follow.

1. **A driver calls `Zw*`, not `Nt*`.** The `Zw` entry points are thin wrappers that set `PreviousMode` to `KernelMode` for the duration of the call. Without that, a driver invoked from a user thread would have its kernel buffers probed as if they were user addresses, and its kernel handles validated against the calling process's handle table.
2. **Never pass a user-mode pointer to a `Zw*` routine.** With `PreviousMode == KernelMode` nothing probes, so the kernel dereferences the raw user address as kernel memory: either an immediate `0x50`/`0x3B`, or a silent write into whatever the current process maps at that address - a privilege-escalation primitive.

`ExGetPreviousMode()` and `Irp->RequestorMode` expose the same fact. Use them to decide whether probing is required; do not use them as a substitute for it.

System service numbers are build-specific. Nothing in a driver should depend on an SSDT index; that fragility is one reason dispatch-table patching is both unreliable and blocked by PatchGuard (`wk-driver-hooking` covers the defensive alternatives).

### The executive map

`ntoskrnl.exe` contains the executive (upper layer) and the kernel (lower layer). `hal.dll` abstracts interrupt controllers, timers, DMA and bus access. `win32k.sys` implements the window manager and GDI, which is why session space exists. Drivers link against `ntoskrnl` exports and, for I/O, `hal` exports.

| Subsystem | Responsibility | Driver-visible surface | Typical IRQL / context |
| --- | --- | --- | --- |
| Object manager | Namespace, handles, object types, reference counts, security on open | `ObReferenceObjectByHandle`, `ObOpenObjectByPointer`, `ZwOpen*`, `ZwClose`, `ObDereferenceObject` | PASSIVE for name lookup; dereference works to DISPATCH |
| Memory manager | Address space, PTEs, working sets, pool, MDLs, sections | `ExAllocatePool2`, `MmProbeAndLockPages`, `MmGetSystemAddressForMdlSafe`, `MmMapLockedPagesSpecifyCache` | non-paged pool and MDL mapping to DISPATCH; paged pool and locks at PASSIVE |
| I/O manager | IRPs, device stacks, file objects, completion | `IoCallDriver`, `IoCompleteRequest`, `IoCreateDevice`, `IoAllocateIrp`, `IoBuildDeviceIoControlRequest` | dispatch IRQL depends on caller; PASSIVE to DISPATCH |
| PnP manager | Device enumeration, start/stop/remove, resources | `AddDevice`, `IRP_MJ_PNP`, `IoRegisterDeviceInterface` | PASSIVE for PnP IRPs |
| Power manager | Power IRPs, idle detection, wake | `IRP_MJ_POWER`, `PoCallDriver`, `PoRegisterPowerSettingCallback` | PASSIVE for power IRPs |
| Configuration manager | Registry hives, keys, values, notifications | `ZwOpenKey`, `ZwQueryValueKey`, `RtlQueryRegistryValues`, `CmRegisterCallbackEx` | PASSIVE for registry I/O |
| Cache manager | File cache, shared cache maps, lazy writes | `CcCopyRead`, `CcCopyWrite`, `CcFlushCache`, `CcMdlRead` | PASSIVE; requires a file object |
| Process manager | Process and thread objects, tokens, notifications | `PsLookupProcessByProcessId`, `PsCreateSystemThread`, `PsSetCreateProcessNotifyRoutineEx` | PASSIVE for lookups; notify callbacks mostly PASSIVE |
| Kernel / scheduler | Threads, synchronization, DPCs, timers, IRQL | `KeWaitForSingleObject`, `KeAcquireSpinLock`, `KeInitializeDpc`, `KeSetTimer` | varies by primitive; see the decision table below |
| Security reference monitor | Tokens, access checks, privileges, auditing | `SeAccessCheck`, `SeCaptureSubjectContext`, `SeExports`, `SePrivilegeCheck` | PASSIVE |
| HAL | Interrupt controllers, DMA, bus translation | `IoConnectInterruptEx`, `HalTranslateBusAddress` (legacy) | ISR runs at DIRQL |

### IRQL is an execution constraint, not a priority

IRQL is a per-processor value. Raising it masks interrupts at that level and below **on that processor only**; other processors keep running the same driver code. It has nothing to do with thread priority, which is a scheduling concept: a high-priority thread at PASSIVE_LEVEL is still preemptible by an ISR, and a DPC outranks every thread on its processor.

`PASSIVE_LEVEL` is 0, `APC_LEVEL` is 1, `DISPATCH_LEVEL` is 2. Device IRQLs start at 3 and are assigned per interrupt by `IoConnectInterruptEx`; the top of the range is `HIGH_LEVEL` (15 on x64 with the APIC model), with the clock and inter-processor interrupt levels just below it. Because the constraint is per-processor, "it worked on my single-core VM" is not evidence: an SMP guest exposes the race immediately.

The decision table to consult before adding a call:

| Situation | IRQL | Process context | Allowed | Forbidden |
| --- | --- | --- | --- | --- |
| `DriverEntry`, `DriverUnload`, `AddDevice` | PASSIVE | system thread (System process) | any PASSIVE API, registry, file I/O, waits, paged code | assuming a user address space exists |
| Dispatch routine for a user's `DeviceIoControl` | PASSIVE | **caller's** process and thread | probe and copy user buffers, `Zw*`, waits, paged code | assuming the caller keeps its buffer stable (double fetch) |
| Dispatch routine reached from a DPC or a lower driver | PASSIVE to DISPATCH | arbitrary thread | only what the IRQL allows | `Zw*`, paged code, waits if IRQL > PASSIVE |
| I/O completion routine | caller's IRQL, up to DISPATCH | arbitrary thread | `KeSetEvent`, `IoFreeIrp`, non-paged work | `Zw*`, paged code, blocking |
| Work item (`IoQueueWorkItem`) | PASSIVE | system thread | blocking, registry, file I/O, paged code | touching user addresses (no user process) |
| Dedicated system thread | PASSIVE | system thread | everything PASSIVE, long loops | outliving the driver image |
| Timer or queued DPC | DISPATCH | arbitrary thread | spinlocks, non-paged pool, `KeSetEvent`, `IoCompleteRequest` | any page-fault, any wait, `Zw*` |
| ISR (`IoConnectInterruptEx`) | DIRQL (3+) | arbitrary thread | non-paged memory, `KeSynchronizeExecution`, queueing a DPC | everything else; keep it to microseconds |

### Where a driver sits

A driver is a loadable image in the kernel half, reached through a `DRIVER_OBJECT` and one or more `DEVICE_OBJECT`s. Requests arrive as IRPs, each carrying a stack of `IO_STACK_LOCATION` entries, one per device object on the path.

```
user: CreateFileW("\\\\.\\CtxProbe")
          |
          v
   I/O manager: parse the name, build IRP_MJ_CREATE,
   allocate one stack location per device in the chain
          |
   +---------------------------+  <- upper filter (optional)
   |  DEVICE_OBJECT + driver   |     IrpSp->DeviceObject, own extension
   +---------------------------+
   |  DEVICE_OBJECT + driver   |  <- function driver: yours
   +---------------------------+
   |  DEVICE_OBJECT + driver   |  <- lower filter (optional)
   +---------------------------+
   |  PDO from the bus driver  |
   +---------------------------+
          |
          v
   hardware, or nothing at all for a software device
```

Two consequences: the same IRP visits several drivers, so a filter must copy the stack location before forwarding; and the IRQL at which a dispatch routine runs is inherited from whoever called it, not chosen by the driver. A driver that assumes PASSIVE in its dispatch routine will eventually be called at DISPATCH by another kernel component.

## Key APIs and structures

| API / structure | Purpose | IRQL / context | Notes |
| --- | --- | --- | --- |
| `PsGetCurrentProcess`, `IoGetCurrentProcess` | `PEPROCESS` of the current thread | any IRQL | borrowed pointer, no reference; do not cache |
| `PsGetCurrentProcessId`, `PsGetCurrentThreadId` | numeric IDs | any IRQL | IDs are reused after exit; never use as identity |
| `PsGetCurrentThread` | `PETHREAD` | any IRQL | in a DPC this is the interrupted thread |
| `KeGetCurrentIrql` | current `KIRQL` | any IRQL | inline, cheap, ideal for `NT_ASSERT` |
| `KeGetCurrentProcessorNumber` | current CPU index | any IRQL | can change after preemption |
| `PsGetThreadProcess` | owning `PEPROCESS` | any IRQL | borrowed pointer |
| `PsLookupProcessByProcessId`, `PsLookupThreadByThreadId` | referenced object from an ID | PASSIVE | caller must `ObDereferenceObject` |
| `PsGetProcessImageFileName` | 15-character image name | any IRQL | `PCHAR`, not null-terminated beyond 15 chars |
| `KeStackAttachProcess`, `KeUnstackDetachProcess` | switch to another address space | PASSIVE | never block while attached |
| `ProbeForRead`, `ProbeForWrite` | validate a user range | PASSIVE to APC | probe inside `__try`; does not stop double fetch |
| `MmIsAddressValid` | mapped-at-this-instant check | any IRQL | diagnostic only, racy |
| `MmGetSystemAddressForMdlSafe` | map an MDL into system space | up to DISPATCH | mapping is not permanent; do not cache |
| `IoGetRequestorProcess` | process that issued the IRP | any IRQL | valid for buffered and direct IRPs |
| `ExGetPreviousMode` | caller's `KPROCESSOR_MODE` | any IRQL | compare against `UserMode` |
| `ExAllocatePool2` | pool allocation | non-paged up to DISPATCH | paged pool only at PASSIVE; Windows 10 2004+/WDK, use `ExAllocatePoolWithTag` on older targets |
| `KeExpandKernelStackAndCallout` | run a callout on a larger stack | PASSIVE | expensive; deep call chains only |
| `IRP->RequestorMode` | mode of the requesting thread | any IRQL | authoritative input for buffer decisions |
| `ETHREAD`, `EPROCESS`, `KTHREAD`, `PEB`, `TEB` | object bodies | - | opaque to drivers; use documented accessors |

## Step-by-step: classify a routine before writing it

1. **Write the contract as a comment.** One line per routine: `/* IRQL: PASSIVE. Context: arbitrary thread. Buffers: kernel only. */`. Check every callee against it. A routine whose contract is "PASSIVE, any thread" must not be called from a DPC, and the comment is what catches that in review.
2. **Decide whether process context is required.** Anything that touches a user address, a per-process handle, or a `PEB`/`TEB` needs the right process to be current. If the work may be deferred, capture the data at PASSIVE in the caller's context instead of deferring with a user pointer.
3. **Classify every pointer.** Kernel non-paged, kernel paged, MDL-mapped, user, or untrusted. For user pointers the decision is `Irp->RequestorMode == UserMode`; then either use `METHOD_BUFFERED`/`METHOD_DIRECT` and let the I/O manager handle it, or probe and copy once inside `__try` into pool memory.
4. **Choose the deferral.** Inline for short, non-blocking, IRQL-compatible work; a DPC for hardware follow-up at DISPATCH; a work item for blocking or paged work; a system thread for a long-lived loop. Cross-check the target IRQL against step 1.
5. **Budget the stack.** Keep a single frame under about 1 KB and the whole call chain under about 8 KB on x64. Verify by disassembling the prologue (`sub rsp, N`) or by inspecting a captured stack with `kv`; treat any local array over 256 bytes as a defect.
6. **Check re-entrancy and parallelism.** Two processors can be inside the routine simultaneously even if every caller is PASSIVE. Identify the lock, the interlocked operation or the per-processor state that makes the routine safe; if there is none, say why.
7. **Instrument the context once.** Capture PID, TID, IRQL and processor in a small snapshot structure and expose it through a buffered IOCTL (the example below). This turns "which context was that?" into a measurement instead of an argument.
8. **Verify under Driver Verifier.** Enable `verifier /standard /driver <name>.sys` and add `verifier /flags 0x4` (force IRQL checking) to catch paged accesses at raised IRQL; keep `verifier /flags 0x1` (special pool) to catch overruns.

## Complete example

A legacy WDM driver that reports the context it observes, plus a client that compares its own IDs. The timer DPC provides a contrast: the same helper prints DISPATCH_LEVEL and an arbitrary thread instead of the client's context.

```c
/* ctxprobe.c - reports the execution context the driver observes */
#include <ntddk.h>

#define CTXPROBE_DEVICE   L"\\Device\\CtxProbe"
#define CTXPROBE_SYMLINK  L"\\DosDevices\\CtxProbe"

#define IOCTL_CTXPROBE_QUERY \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _CTXPROBE_SNAPSHOT {
    ULONG ProcessId;
    ULONG ThreadId;
    ULONG Processor;
    UCHAR Irql;
    UCHAR Reserved[3];
} CTXPROBE_SNAPSHOT, *PCTXPROBE_SNAPSHOT;

static KTIMER g_CtxTimer;
static KDPC   g_CtxDpc;

DRIVER_DISPATCH CtxProbeCreate;
DRIVER_DISPATCH CtxProbeClose;
DRIVER_DISPATCH CtxProbeDeviceControl;

static VOID CtxProbeLogContext(_In_z_ PCSTR Where)
{
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "CtxProbe[%s] pid=%u tid=%u irql=%u cpu=%u\n",
               Where,
               (ULONG)(ULONG_PTR)PsGetCurrentProcessId(),
               (ULONG)(ULONG_PTR)PsGetCurrentThreadId(),
               (ULONG)KeGetCurrentIrql(),
               KeGetCurrentProcessorNumber());
}

static VOID CtxProbeTimerDpc(_In_ PKDPC Dpc,
                             _In_opt_ PVOID DeferredContext,
                             _In_opt_ PVOID SystemArgument1,
                             _In_opt_ PVOID SystemArgument2)
{
    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(DeferredContext);
    UNREFERENCED_PARAMETER(SystemArgument1);
    UNREFERENCED_PARAMETER(SystemArgument2);

    CtxProbeLogContext("timer-dpc");   /* DISPATCH_LEVEL, arbitrary thread */
}

NTSTATUS CtxProbeCreate(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

NTSTATUS CtxProbeClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    return CtxProbeCreate(DeviceObject, Irp);
}

NTSTATUS CtxProbeDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR information = 0;

    UNREFERENCED_PARAMETER(DeviceObject);

    if (stack->Parameters.DeviceIoControl.IoControlCode == IOCTL_CTXPROBE_QUERY) {
        if (stack->Parameters.DeviceIoControl.OutputBufferLength <
            sizeof(CTXPROBE_SNAPSHOT)) {
            status = STATUS_BUFFER_TOO_SMALL;
        } else {
            PCTXPROBE_SNAPSHOT snapshot =
                (PCTXPROBE_SNAPSHOT)Irp->AssociatedIrp.SystemBuffer;

            RtlZeroMemory(snapshot, sizeof(*snapshot));
            snapshot->ProcessId = (ULONG)(ULONG_PTR)PsGetCurrentProcessId();
            snapshot->ThreadId  = (ULONG)(ULONG_PTR)PsGetCurrentThreadId();
            snapshot->Processor = KeGetCurrentProcessorNumber();
            snapshot->Irql      = (UCHAR)KeGetCurrentIrql();
            information = sizeof(*snapshot);
            status = STATUS_SUCCESS;

            /* Runs in the caller's thread: these values must match the client. */
            CtxProbeLogContext("ioctl");
        }
    }

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

static VOID CtxProbeUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING symlinkName;
    NTSTATUS status;

    KeCancelTimer(&g_CtxTimer);
    KeFlushQueuedDpcs();               /* no DPC may run after this point */

    RtlInitUnicodeString(&symlinkName, CTXPROBE_SYMLINK);
    status = IoDeleteSymbolicLink(&symlinkName);
    if (!NT_SUCCESS(status) && status != STATUS_OBJECT_NAME_NOT_FOUND) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "CtxProbe: IoDeleteSymbolicLink failed 0x%08X\n", status);
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
    UNICODE_STRING deviceName;
    UNICODE_STRING symlinkName;
    LARGE_INTEGER dueTime;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&deviceName, CTXPROBE_DEVICE);
    status = IoCreateDevice(DriverObject, 0, &deviceName, FILE_DEVICE_UNKNOWN,
                            FILE_DEVICE_SECURE_OPEN, FALSE, &deviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    RtlInitUnicodeString(&symlinkName, CTXPROBE_SYMLINK);
    status = IoCreateSymbolicLink(&symlinkName, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(deviceObject);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE]         = CtxProbeCreate;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]          = CtxProbeClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = CtxProbeDeviceControl;
    DriverObject->DriverUnload                         = CtxProbeUnload;

    KeInitializeTimer(&g_CtxTimer);
    KeInitializeDpc(&g_CtxDpc, CtxProbeTimerDpc, NULL);
    dueTime.QuadPart = -2 * 10 * 1000 * 1000;   /* 2 s, relative */
    KeSetTimer(&g_CtxTimer, dueTime, &g_CtxDpc);

    CtxProbeLogContext("DriverEntry");
    return STATUS_SUCCESS;
}
```

The client prints its own identifiers and the driver's view of the same call:

```c
/* ctxclient.c - compare client and driver view of the same call */
#include <windows.h>
#include <stdio.h>

#define IOCTL_CTXPROBE_QUERY \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _CTXPROBE_SNAPSHOT {
    ULONG ProcessId;
    ULONG ThreadId;
    ULONG Processor;
    UCHAR Irql;
    UCHAR Reserved[3];
} CTXPROBE_SNAPSHOT;

int wmain(void)
{
    HANDLE device;
    CTXPROBE_SNAPSHOT snapshot;
    DWORD returned = 0;

    device = CreateFileW(L"\\\\.\\CtxProbe", GENERIC_READ | GENERIC_WRITE,
                         0, NULL, OPEN_EXISTING, 0, NULL);
    if (device == INVALID_HANDLE_VALUE) {
        wprintf(L"CreateFileW failed: %lu\n", GetLastError());
        return 1;
    }

    ZeroMemory(&snapshot, sizeof(snapshot));
    if (!DeviceIoControl(device, IOCTL_CTXPROBE_QUERY, NULL, 0,
                         &snapshot, sizeof(snapshot), &returned, NULL)) {
        wprintf(L"DeviceIoControl failed: %lu\n", GetLastError());
        CloseHandle(device);
        return 2;
    }

    wprintf(L"client  pid=%lu tid=%lu\n",
            GetCurrentProcessId(), GetCurrentThreadId());
    wprintf(L"driver  pid=%lu tid=%lu irql=%u cpu=%lu bytes=%lu\n",
            snapshot.ProcessId, snapshot.ThreadId, snapshot.Irql,
            snapshot.Processor, returned);

    CloseHandle(device);
    return 0;
}
```

Expected: the two PID/TID pairs match, the driver reports IRQL 0, and only the DPC log line shows IRQL 2 with a thread ID that belongs to no user process.

## Variations and advanced use

- **Deferral instead of inline.** When the work needs PASSIVE but the routine may run higher, queue an `IO_WORKITEM` with `IoQueueWorkItem` and complete the IRP from the worker; the device object reference taken by `IoAllocateWorkItem` keeps the device alive, but unload must still drain outstanding items (`wk-work-items`).
- **Attaching to a process.** `KeStackAttachProcess` makes another process's user half visible; it is PASSIVE_LEVEL only, must not be held across a blocking call, and is unnecessary for the common case because the I/O manager already captures buffers. Reach for it only when a documented API requires the target process to be current.
- **Copying across address spaces.** `MmCopyVirtualMemory` performs the attach internally and is the least error-prone way to read another process's memory from a driver, but it requires a referenced `PEPROCESS` and PASSIVE_LEVEL, and it is not a substitute for the access checks the caller's own handle would have performed.
- **Doing it in user mode.** If the task is "read another process's memory", a user-mode service with `OpenProcess`/`ReadProcessMemory` is both safer and supportable. Kernel code should be reserved for things user mode cannot do.
- **32-bit callers on x64.** `IoIs32bitProcess(Irp)` reports a WOW64 caller; the 32-bit `PEB` lives at `PsGetProcessWow64Process`, and structure layouts differ. Never infer the caller's bitness from a pointer value.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
| --- | --- | --- |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA`, faulting address inside a pool allocation, appears after minutes of load | paged pool or freed pool touched at DISPATCH_LEVEL | `!analyze -v`, then `!pool <addr>` and `!irql`; move the allocation to `POOL_FLAG_NON_PAGED` |
| `0xA IRQL_NOT_LESS_OR_EQUAL`, parameter 2 is 2, parameter 1 looks like a user address | user pointer dereferenced in a DPC or completion routine | `!analyze -v`; capture and copy the data at PASSIVE instead |
| `0xD1 DRIVER_IRQL_NOT_LESS_OR_EQUAL`, parameter 4 names your driver | pageable code or data reached above PASSIVE | `!analyze -v` gives the instruction; move the routine out of `PAGE`, or lower the IRQL before calling it |
| `0x7F UNEXPECTED_KERNEL_MODE_TRAP`, parameter 1 = `8` | kernel stack overflow from a large frame or recursion | `!analyze -v`, then `kv` to see the repeating frames; move buffers to pool, cap recursion |
| `0x1E KMODE_EXCEPTION_NOT_HANDLED` or `0x3B SYSTEM_SERVICE_EXCEPTION` in your module | unhandled access violation on a bad pointer | `!analyze -v`; probe user buffers, validate kernel pointers before dereference |
| `0xC0000005` returned to the client with no bugcheck | the driver caught the fault or failed the IRP | inspect `Irp->IoStatus.Status` and the client's `GetLastError` |
| Logged thread ID changes between entry and completion | the request was completed from a DPC or worker | expected; never key state off "the current thread" |
| `0x133 DPC_WATCHDOG_VIOLATION` | a DPC ran far too long, usually a poll loop | `!analyze -v`, `!dpcs`; move work to a work item |
| `STATUS_INVALID_DEVICE_REQUEST` (`0xC0000010`) from the client | the IRP major function has no handler in `MajorFunction` | check the dispatch table assignments in `DriverEntry` |

## Safety, IRQL, and cleanup checklist

- Every routine carries an IRQL/context comment, and `NT_ASSERT(KeGetCurrentIrql() == PASSIVE_LEVEL)` guards anything that paged-pool allocates, waits, or calls `Zw*`.
- No blocking wait above PASSIVE; no `KeWaitForSingleObject` on a user event from a DPC.
- Everything touched at DISPATCH or above comes from non-paged pool or a non-paged global.
- User addresses are only dereferenced at PASSIVE, after `Irp->RequestorMode` is checked and `ProbeForRead`/`ProbeForWrite` succeeds inside `__try`; the copy is taken once.
- Locals stay small; large structures come from pool with a unique tag and are freed on every path, including failure paths.
- Pageable routines are listed in `#pragma alloc_text(PAGE, RoutineName)` and begin with `PAGED_CODE()`.
- Timers and DPCs are cancelled and flushed (`KeCancelTimer`, `KeFlushQueuedDpcs`) before the code that they call is unloaded.
- Device objects are deleted after the symbolic link, and only when no handles remain.
- No pointer to a local or to a stack buffer escapes the routine into a queue, DPC, or work item.
- Driver Verifier with standard flags plus force IRQL checking runs on every test cycle before the driver ships.

## Verification

Build and install on a disposable VM with test signing enabled:

```bat
msbuild ctxprobe.vcxproj /p:Configuration=Debug /p:Platform=x64 /p:TreatWarningAsError=true
sc create CtxProbe type= kernel binPath= C:\drv\ctxprobe.sys
sc start CtxProbe
ctxclient.exe
```

The client output must show matching IDs and IRQL 0; a DebugView capture (with `DPFLTR_IHVDRIVER_ID` enabled) must additionally show one `timer-dpc` line at IRQL 2 within two seconds. A mismatch in PID means the request was completed in another context; a mismatch in TID with a matching PID means completion happened on a different thread of the same process.

In the debugger, with the target broken in:

```
lkd> !drvobj CtxProbe 2
Driver object (ffffa70b`2c3d4e50) is for: \Driver\CtxProbe
DriverEntry:   fffff801`3a1c1070  ctxprobe!DriverEntry
DriverUnload:  fffff801`3a1c1450  ctxprobe!CtxProbeUnload
Device Object list:
ffffa70b`2d10b610  \Device\CtxProbe
lkd> !irql
Debugger saved IRQL for processor 0x0 -- 0 (PASSIVE_LEVEL)
lkd> ?? @$curthread->PreviousMode
```

`!irql` is per-processor and reports the IRQL of the processor the debugger stopped on; `PreviousMode` reads as 0 (`UserMode`) for a thread that entered the kernel through a system call. To confirm a driver is running in the caller's context, break in `CtxProbeDeviceControl` and compare `!thread` with `!process` for the client: the process must be the client's, and `kv` must show the system-call frames beneath the driver.
