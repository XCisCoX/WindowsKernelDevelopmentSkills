---
name: wk-process-notifications
description: Use when a driver must observe or veto process creation and exit. Covers PsSetCreateProcessNotifyRoutineEx and Ex2, PS_CREATE_NOTIFY_INFO fields, blocking a process through CreationStatus, the PASSIVE/APC-disabled callback contract, a bounded non-paged event queue, delivery to user mode by a pending inverted-call IOCTL with cancellation, and an unload that cannot race in-flight callbacks.
---

# Process notifications: in-line process observation with a veto

`PsSetCreateProcessNotifyRoutineEx` gives a driver a callback inside the process-creation path, so no process can be created without your code running and no short-lived process can slip between polls. The callback can veto creation by writing a failure status into `CreateInfo->CreationStatus`, which no user-mode mechanism can do. The one rule that prevents most disasters: **the callback runs at `PASSIVE_LEVEL` with normal kernel APCs already disabled, inside the creator's own process creation, so it must do bounded work with no file, registry, or network I/O, and it must enqueue rather than deliver**. Delivery happens later, on a client thread, through a pending IRP.

Registration is also where most projects fail first: the driver image must be linked with `/INTEGRITYCHECK`, or the registration call returns `STATUS_ACCESS_DENIED` (`0xC0000022`) and every later assumption is void.

## When to use

- A security product must log every process start and exit with parent, creator thread, image path, and command line.
- Process creation must be blocked for a policy reason (deny-list, licence check, EDR verdict) with a real error returned to the caller.
- A driver needs a process inventory that cannot miss short-lived processes (installers, `conhost.exe` helpers, script hosts).
- Per-process state must be attached and torn down exactly once, in creation order.
- A user-mode service needs a push channel of process events instead of polling.
- You are diagnosing why `PsSetCreateProcessNotifyRoutineEx` fails, why the callback deadlocks, or why unload bugchecks with pending IRPs.
- Do not use for: thread or image events (`wk-thread-image-notifications`); handle-open interception (`wk-object-callbacks`); registry change filtering (`wk-registry-callbacks`); plain ETW-only telemetry (`wk-kernel-tracing`).

## Prerequisites and related skills

- Kernel C with `<ntddk.h>`; WDK 10. `ExAllocatePool2` needs Windows 10 2004 or later; use `ExAllocatePoolWithTag(NonPagedPool, ...)` on older targets.
- The driver project must pass `/INTEGRITYCHECK` to the linker. Without it `PsSetCreateProcessNotifyRoutineEx` returns `STATUS_ACCESS_DENIED` regardless of test signing.
- `PsSetCreateProcessNotifyRoutineEx2` and `IsSubsystemProcess` require Windows 10 1607 or later.
- The driver must own a device object before it registers, because the callback needs somewhere to enqueue.
- Sibling skills: `wk-irql` (what `PASSIVE_LEVEL` with APCs disabled still forbids), `wk-irp-queues-cancel` (cancel-safe queue mechanics in depth), `wk-driver-client-protocol` (IOCTL contract, `CTL_CODE`, `GetLastError` mapping), `wk-synchronization` (push locks versus fast mutexes), `wk-crash-dump-analysis` (reading `0xCE` and `0x9F`).

## Mental model

The kernel keeps a fixed table of process-notify registrations. During `NtCreateUserProcess`, before the new process is fully constructed and before its first thread runs, the process manager walks that table in registration order and calls each driver. A failure status written by any callback aborts the creation and **stops the walk**, so later drivers never see the process. Exit notifications take the same path when the last thread of a process terminates.

```text
  CreateProcess / NtCreateUserProcess
        |
        v
  process manager walks the notify table (fixed number of slots)
        |
        +--> driver callback @ PASSIVE, APCs disabled, creator's context
        |        allow -> CreationStatus stays STATUS_SUCCESS
        |        veto  -> CreationStatus = STATUS_ACCESS_DENIED, creation aborts
        v
  process object lives  ------------------------------+
        |                                             |
        |  exit: last thread terminates               |  event record
        v                                             v
  exit callback @ PASSIVE, APCs disabled        bounded queue (non-paged)
                                                      |
  client: DeviceIoControl(IOCTL_PW_WAIT_FOR_EVENT)    |
        |  queue empty -> mark pending, queue the IRP |
        |  queue full  -> complete immediately        v
        +----------------------------------> batch of PW_EVENT records
```

Three properties explain the whole design.

1. **The callback is in-line and cannot be missed**, but it is also in the middle of somebody else's syscall. Everything you do there delays a process launch, and anything you wait on can deadlock against the very subsystem that is creating the process.
2. **Only the `Ex` and `Ex2` variants receive `PS_CREATE_NOTIFY_INFO`**, so only they can veto. The legacy `PsSetCreateProcessNotifyRoutine` callback gets two IDs and a `BOOLEAN`; it can observe but never block.
3. **The veto is a status, not an exception.** Writing `STATUS_ACCESS_DENIED` into `CreationStatus` makes the creator's `CreateProcess` fail with that status. Writing any success status leaves creation unaffected.

The callback therefore does three things only: copy what it needs out of `PS_CREATE_NOTIFY_INFO`, decide the policy, and append a record to a queue. A client thread later turns queue records into completed IRPs.

## Key APIs and structures

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `PsSetCreateProcessNotifyRoutineEx(Routine, Remove)` | Register/unregister with veto | `PASSIVE` | `Remove = TRUE` unregisters; returns `NTSTATUS` |
| `PsSetCreateProcessNotifyRoutineEx2(Type, Routine, Remove)` | Same, plus Pico/WSL1 processes | `PASSIVE` | `Type` is `PsCreateProcessNotifySubsystems` (0) |
| `PsSetCreateProcessNotifyRoutine(Routine, Remove)` | Legacy, observe only | `PASSIVE` | Deprecated; no `CreateInfo`, no veto |
| `PCREATE_PROCESS_NOTIFY_ROUTINE_EX` | `VOID (PEPROCESS, HANDLE, PPS_CREATE_NOTIFY_INFO)` | `PASSIVE` | `CreateInfo == NULL` means exit |
| `PCREATE_PROCESS_NOTIFY_ROUTINE` | `VOID (HANDLE ParentId, HANDLE ProcessId, BOOLEAN Create)` | `PASSIVE` | Cannot block |
| `PsGetProcessExitStatus(Process)` | Exit code in the exit branch | `PASSIVE` | Only valid once the process is terminating |
| `KeQuerySystemTimePrecise(&Time)` | UTC timestamp for the record | any | Windows 8+; else `KeQuerySystemTime` |
| `IoCsqInitialize / IoCsqInsertIrp / IoCsqRemoveNextIrp` | Cancel-safe queue of pending IRPs | `<= DISPATCH` | Removal clears the cancel routine for you |
| `IoMarkIrpPending` | Set `SL_PENDING_RETURNED` | `<= DISPATCH` | Call before the IRP is reachable by a completer |
| `ExInitializePushLock / ExAcquirePushLockShared / Exclusive` | Protect the policy table | `<= APC_LEVEL` | Never hold while allocating or completing |
| `RtlEqualUnicodeString(&A, &B, TRUE)` | Case-insensitive path compare | any | Both strings must have valid `Length` |

`PS_CREATE_NOTIFY_INFO` is read-only except for `CreationStatus`:

| Field | Type | Meaning | Trap |
|---|---|---|---|
| `Size` | `SIZE_T` | Size of the structure | Version gate; check before trusting later fields |
| `Flags.FileOpenNameAvailable` | bitfield | `ImageFileName` is valid | **Do not read `ImageFileName` when this bit is clear** |
| `Flags.IsSubsystemProcess` | bitfield | Pico process (WSL1) | Only set when registered with `Ex2` |
| `ParentProcessId` | `HANDLE` | Declared parent, an ID not a handle | Differs from the creator when `PROC_THREAD_ATTRIBUTE_PARENT_PROCESS` was used |
| `CreatingThreadId` | `CLIENT_ID` | `UniqueProcess` / `UniqueThread` of the caller | This is the real origin of the launch |
| `FileObject` | `PFILE_OBJECT` | Image file object | Valid only when `FileOpenNameAvailable`; do not dereference after return |
| `ImageFileName` | `PCUNICODE_STRING` | NT path of the image | Not null-terminated; copy `Length` bytes, never store the pointer |
| `CommandLine` | `PCUNICODE_STRING` | Full command line, may be `NULL` | Same lifetime and termination rules |
| `CreationStatus` | `NTSTATUS` | **Writable**; failure vetoes creation | A success value means "allow" |

Registration outcomes worth distinguishing: `STATUS_SUCCESS`; `STATUS_ACCESS_DENIED` (`0xC0000022`) for a missing `IMAGE_DLLCHARACTERISTICS_FORCE_INTEGRITY` flag; `STATUS_INVALID_PARAMETER` (`0xC000000D`) when the same routine is already registered or when a routine is registered with both `Ex` and `Ex2`; `STATUS_INSUFFICIENT_RESOURCES` (`0xC000009A`) when the system-wide table (64 entries in current builds) is exhausted.

### IRQL analysis

| Path | IRQL | What that forbids |
|---|---|---|
| Process notify callback, create and exit | `PASSIVE_LEVEL`, normal APCs disabled | File I/O, registry writes, `ZwCreateProcess`, waiting on an event whose completion needs an APC |
| Enqueue into the event queue | `PASSIVE`, then `DISPATCH` inside the spinlock | Paged pool, page faults, anything unbounded under the lock |
| Complete an inverted-call IRP from the callback | `PASSIVE` | Nothing beyond the completion rules |
| CSQ lock callbacks (`AcquireLock`/`ReleaseLock`) | `DISPATCH` with the global cancel lock sometimes held | Blocking, pageable memory, acquiring your spinlock in the wrong order |
| Cancel-completed callback | `DISPATCH`, possibly with the CSQ lock held | Touching the queue, allocating, waiting |
| IOCTL dispatch | `PASSIVE` | Nothing special; this is where allocation and validation belong |
| `DriverUnload` | `PASSIVE` | May wait, may unregister, may delete the device |

## Step-by-step

1. **Add `/INTEGRITYCHECK` and prove it.** Linker → Command Line → Additional Options: `/INTEGRITYCHECK`. After the build: `dumpbin /headers x64\Debug\ProcWatch.sys | findstr /i integrity` must show `Integrity` in `DllCharacteristics`. If it does not, do not debug the registration call; fix the linker first.
2. **Create the device and the extension before registering.** The callback needs the queue. `IoCreateDevice` with `sizeof(PW_DEVICE_EXT)` of extension, `FILE_DEVICE_SECURE_OPEN` characteristics, and `Exclusive = TRUE` so one consumer owns the event stream.
3. **Initialize every lock and the drain event before the register call.** `KeInitializeSpinLock`, `InitializeListHead` for both lists, `ExInitializePushLock`, `KeInitializeEvent(&ext->CallbacksDrained, NotificationEvent, FALSE)`, `IoCsqInitialize`. A callback that runs before the queue is initialized is a guaranteed `0x50`.
4. **Register last, and unwind in reverse on failure.** `PsSetCreateProcessNotifyRoutineEx(PwProcessNotify, FALSE)`. If it fails, delete the symbolic link, free the policy array, delete the device, and return the status unchanged so the failure is visible at `sc start`.
5. **Write the callback as a filter, not a worker.** Increment the in-flight counter, bail out if `ShuttingDown` is set, read only fields the flags say are valid, copy the image path by `Length`, decide the verdict, enqueue, decrement, and signal if the count reached zero. No `DbgPrint` of the path with `%wZ` on an unvalidated pointer, no allocation between the increment and the decrement beyond one pool block.
6. **Veto correctly.** `CreateInfo->CreationStatus = STATUS_ACCESS_DENIED;` and nothing else. Never touch `ProcessId`, `FileObject`, or the strings. Do not add the process to any tracking table when you veto it, because the exit notification you would rely on for cleanup may never describe a process that never fully existed.
7. **Deliver from the client's thread.** The dispatch routine validates `OutputBufferLength`, marks the IRP pending, inserts it into the CSQ, and returns `STATUS_PENDING`. A pump routine, called both by the producer after an enqueue and by the dispatch after an insert, removes waiters and copies out whole records. Completion happens outside every lock the queue owns.
8. **Drain on unload in a fixed order:** set `ShuttingDown`, unregister, wait for the in-flight counter to reach zero, fail every pending IRP with `STATUS_DELETE_PENDING`, free queued records, delete the symbolic link, delete the device. Never free anything the callback can still touch, and never unregister while holding a lock a callback needs.

After each step: step 5 is proven by a `DbgPrintEx` counter that matches the number of `notepad.exe` launches; step 8 is proven by unloading while a client has a wait IRP outstanding and seeing no `0xCE`.

## Complete example

A driver, a shared protocol header, and a client. The driver keeps a bounded queue of process events in non-paged pool, serves a single consumer through a pending IOCTL, and can veto images that match a rule.

```c
/* ProcWatchPublic.h -- shared by ProcWatch.sys and ProcWatchClient.exe.
 * Include <ntddk.h> in kernel mode or <windows.h> + <winioctl.h> in user mode
 * before this file; it deliberately includes nothing itself. */
#pragma once

#define PW_DEVICE_NAME   L"\\Device\\ProcWatch"
#define PW_SYMLINK_NAME  L"\\??\\ProcWatch"
#define PW_DOS_NAME      L"\\\\.\\ProcWatch"

#define PW_PROTOCOL_VERSION 1
#define PW_MAX_IMAGE_CHARS  260
#define PW_MAX_EVENTS       256
#define PW_MAX_BATCH_BYTES  (64 * 1024)
#define PW_MAX_RULES        16
#define PW_MAX_RULE_CHARS   260

typedef enum _PW_EVENT_TYPE {
    PwEventProcessCreate = 1,
    PwEventProcessExit   = 2
} PW_EVENT_TYPE;

#define PW_EVENT_FLAG_NAME_AVAILABLE   0x00000001
#define PW_EVENT_FLAG_HAS_COMMAND_LINE 0x00000002
#define PW_EVENT_FLAG_NAME_TRUNCATED   0x00000004
#define PW_EVENT_FLAG_SUBSYSTEM        0x00000008
#define PW_EVENT_FLAG_BLOCKED          0x00000010

typedef struct _PW_EVENT {
    LARGE_INTEGER Time;              /* UTC, 100 ns units since 1601 */
    NTSTATUS      Status;            /* create: verdict; exit: exit code */
    ULONG         Type;              /* PW_EVENT_TYPE */
    ULONG         ProcessId;
    ULONG         ParentProcessId;
    ULONG         CreatingProcessId;
    ULONG         CreatingThreadId;
    ULONG         Flags;             /* PW_EVENT_FLAG_* */
    ULONG         ImageChars;        /* characters, excluding terminator */
    WCHAR         ImageName[PW_MAX_IMAGE_CHARS];
} PW_EVENT, *PPW_EVENT;

typedef struct _PW_BATCH {
    ULONG    Version;
    ULONG    EventCount;
    ULONG    DroppedEvents;          /* events lost since the previous batch */
    ULONG    Reserved;
    PW_EVENT Events[1];
} PW_BATCH, *PPW_BATCH;

#define PW_BATCH_HEADER_SIZE FIELD_OFFSET(PW_BATCH, Events)
#define PW_BATCH_BYTES(n)    (PW_BATCH_HEADER_SIZE + (n) * sizeof(PW_EVENT))

typedef struct _PW_RULE {
    ULONG Flags;                     /* PW_RULE_FLAG_SUFFIX to match a tail */
    ULONG ImageChars;
    WCHAR ImageName[PW_MAX_RULE_CHARS];
} PW_RULE, *PPW_RULE;

#define PW_RULE_FLAG_SUFFIX 0x00000001

typedef struct _PW_STATS {
    ULONG Version;
    ULONG QueuedEvents;
    ULONG DroppedEvents;
    ULONG RuleCount;
    ULONG BlockedProcesses;
    ULONG DeliveredEvents;
} PW_STATS, *PPW_STATS;

#define IOCTL_PW_WAIT_FOR_EVENT \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x801, METHOD_BUFFERED, FILE_READ_DATA)
#define IOCTL_PW_ADD_RULE \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x802, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_PW_CLEAR_RULES \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x803, METHOD_BUFFERED, FILE_WRITE_DATA)
#define IOCTL_PW_QUERY_STATS \
    CTL_CODE(FILE_DEVICE_UNKNOWN, 0x804, METHOD_BUFFERED, FILE_READ_DATA)
```

```c
/* ProcWatch.c -- build as an Empty WDM Driver, x64, /INTEGRITYCHECK. */
#include <ntddk.h>
#include "ProcWatchPublic.h"

#define PW_TAG 'PwcH'

typedef struct _PW_EVENT_NODE {
    LIST_ENTRY Link;
    PW_EVENT   Event;
} PW_EVENT_NODE, *PPW_EVENT_NODE;

typedef struct _PW_DEVICE_EXT {
    KSPIN_LOCK   Lock;              /* events, waiters, ShuttingDown */
    LIST_ENTRY   Events;            /* PW_EVENT_NODE */
    LIST_ENTRY   Waiters;           /* IRPs tracked by the CSQ */
    ULONG        EventCount;
    IO_CSQ       Csq;
    EX_PUSH_LOCK PolicyLock;        /* Rules, RuleCount */
    PPW_RULE     Rules;
    ULONG        RuleCount;
    LONG         DroppedEvents;
    LONG         BlockedProcesses;
    LONG         DeliveredEvents;
    volatile LONG ShuttingDown;
    volatile LONG ActiveCallbacks;
    KEVENT       CallbacksDrained;
} PW_DEVICE_EXT, *PPW_DEVICE_EXT;

static PPW_DEVICE_EXT g_PwExt;

static VOID PwPump(PPW_DEVICE_EXT Ext);

static VOID PwComplete(_In_ PIRP Irp, _In_ NTSTATUS Status, _In_ ULONG_PTR Information)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = Information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
}

/* --- cancel-safe queue callbacks; all run with Ext->Lock held --------- */

static VOID PwCsqInsertIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    PPW_DEVICE_EXT ext = CONTAINING_RECORD(Csq, PW_DEVICE_EXT, Csq);
    InsertTailList(&ext->Waiters, &Irp->Tail.Overlay.ListEntry);
}

static VOID PwCsqRemoveIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(Csq);
    RemoveEntryList(&Irp->Tail.Overlay.ListEntry);
}

static PIRP PwCsqPeekNextIrp(_In_ PIO_CSQ Csq, _In_opt_ PIRP Irp, _In_opt_ PVOID Context)
{
    PPW_DEVICE_EXT ext = CONTAINING_RECORD(Csq, PW_DEVICE_EXT, Csq);
    PLIST_ENTRY next = (Irp == NULL) ? ext->Waiters.Flink
                                     : Irp->Tail.Overlay.ListEntry.Flink;
    UNREFERENCED_PARAMETER(Context);

    if (next == &ext->Waiters) {
        return NULL;
    }
    return CONTAINING_RECORD(next, IRP, Tail.Overlay.ListEntry);
}

static VOID PwCsqAcquireLock(_In_ PIO_CSQ Csq, _Out_ PKIRQL Irql)
{
    PPW_DEVICE_EXT ext = CONTAINING_RECORD(Csq, PW_DEVICE_EXT, Csq);
    KeAcquireSpinLock(&ext->Lock, Irql);
}

static VOID PwCsqReleaseLock(_In_ PIO_CSQ Csq, _In_ KIRQL Irql)
{
    PPW_DEVICE_EXT ext = CONTAINING_RECORD(Csq, PW_DEVICE_EXT, Csq);
    KeReleaseSpinLock(&ext->Lock, Irql);
}

static VOID PwCsqCompleteCanceledIrp(_In_ PIO_CSQ Csq, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(Csq);
    PwComplete(Irp, STATUS_CANCELLED, 0);
}

/* --- event production ------------------------------------------------- */

static VOID PwQueueEvent(_In_ PPW_DEVICE_EXT Ext, _In_ ULONG Type,
                         _In_ HANDLE ProcessId, _In_ HANDLE ParentId,
                         _In_ HANDLE CreatingProcessId, _In_ HANDLE CreatingThreadId,
                         _In_ NTSTATUS Status, _In_ ULONG Flags,
                         _In_opt_ PCUNICODE_STRING Image)
{
    PPW_EVENT_NODE node;
    PPW_EVENT ev;
    ULONG chars = 0;
    KIRQL irql;

    node = (PPW_EVENT_NODE)ExAllocatePool2(POOL_FLAG_NON_PAGED,
                                           sizeof(PW_EVENT_NODE), PW_TAG);
    if (node == NULL) {
        InterlockedIncrement(&Ext->DroppedEvents);
        return;
    }

    ev = &node->Event;
    RtlZeroMemory(ev, sizeof(*ev));
    KeQuerySystemTimePrecise(&ev->Time);
    ev->Type = Type;
    ev->Status = Status;
    ev->ProcessId = HandleToULong(ProcessId);
    ev->ParentProcessId = HandleToULong(ParentId);
    ev->CreatingProcessId = HandleToULong(CreatingProcessId);
    ev->CreatingThreadId = HandleToULong(CreatingThreadId);
    ev->Flags = Flags;

    if (Image != NULL && Image->Buffer != NULL && Image->Length > 0) {
        chars = Image->Length / sizeof(WCHAR);
        if (chars >= PW_MAX_IMAGE_CHARS) {
            chars = PW_MAX_IMAGE_CHARS - 1;
            ev->Flags |= PW_EVENT_FLAG_NAME_TRUNCATED;
        }
        RtlCopyMemory(ev->ImageName, Image->Buffer, chars * sizeof(WCHAR));
    }
    ev->ImageName[chars] = L'\0';
    ev->ImageChars = chars;

    KeAcquireSpinLock(&Ext->Lock, &irql);
    if (Ext->ShuttingDown) {
        KeReleaseSpinLock(&Ext->Lock, irql);
        ExFreePoolWithTag(node, PW_TAG);
        return;
    }
    if (Ext->EventCount >= PW_MAX_EVENTS) {
        /* Drop the oldest: a monitor is worth more when it reports recent
         * activity. Flip to dropping the newest if you need FIFO integrity. */
        PLIST_ENTRY head = RemoveHeadList(&Ext->Events);
        ExFreePoolWithTag(CONTAINING_RECORD(head, PW_EVENT_NODE, Link), PW_TAG);
        Ext->EventCount--;
        InterlockedIncrement(&Ext->DroppedEvents);
    }
    InsertTailList(&Ext->Events, &node->Link);
    Ext->EventCount++;
    KeReleaseSpinLock(&Ext->Lock, irql);

    PwPump(Ext);
}

/* --- delivery --------------------------------------------------------- */

static VOID PwPump(_In_ PPW_DEVICE_EXT Ext)
{
    for (;;) {
        PIRP irp;
        PIO_STACK_LOCATION sp;
        PPW_BATCH batch;
        ULONG capacity, count = 0, dropped, i;
        LIST_ENTRY taken;
        KIRQL irql;

        KeAcquireSpinLock(&Ext->Lock, &irql);
        if (IsListEmpty(&Ext->Events) || IsListEmpty(&Ext->Waiters)) {
            KeReleaseSpinLock(&Ext->Lock, irql);
            return;
        }
        KeReleaseSpinLock(&Ext->Lock, irql);

        irp = IoCsqRemoveNextIrp(&Ext->Csq, NULL);
        if (irp == NULL) {
            return;
        }

        sp = IoGetCurrentIrpStackLocation(irp);
        if (sp->Parameters.DeviceIoControl.OutputBufferLength <
            PW_BATCH_HEADER_SIZE + sizeof(PW_EVENT)) {
            PwComplete(irp, STATUS_BUFFER_TOO_SMALL, 0);
            continue;
        }
        batch = (PPW_BATCH)irp->AssociatedIrp.SystemBuffer;
        capacity = (sp->Parameters.DeviceIoControl.OutputBufferLength -
                    PW_BATCH_HEADER_SIZE) / sizeof(PW_EVENT);

        InitializeListHead(&taken);
        KeAcquireSpinLock(&Ext->Lock, &irql);
        while (count < capacity && !IsListEmpty(&Ext->Events)) {
            InsertTailList(&taken, RemoveHeadList(&Ext->Events));
            Ext->EventCount--;
            count++;
        }
        dropped = (ULONG)InterlockedExchange(&Ext->DroppedEvents, 0);
        KeReleaseSpinLock(&Ext->Lock, irql);

        if (count == 0) {
            /* Another pump won the race and took the events. Put this IRP
             * back; completing it empty would busy-loop the client. */
            IoCsqInsertIrp(&Ext->Csq, irp, NULL);
            return;
        }

        batch->Version = PW_PROTOCOL_VERSION;
        batch->EventCount = 0;
        batch->DroppedEvents = dropped;
        batch->Reserved = 0;
        i = 0;
        while (!IsListEmpty(&taken)) {
            PPW_EVENT_NODE node =
                CONTAINING_RECORD(RemoveHeadList(&taken), PW_EVENT_NODE, Link);
            batch->Events[batch->EventCount++] = node->Event;
            ExFreePoolWithTag(node, PW_TAG);
            i++;
        }
        InterlockedExchangeAdd(&Ext->DeliveredEvents, (LONG)batch->EventCount);
        PwComplete(irp, STATUS_SUCCESS, PW_BATCH_BYTES(batch->EventCount));
    }
}
```

The policy check and the callback itself. The callback is the only code that runs inside process creation, so it stays small:

```c
static BOOLEAN PwIsBlocked(_In_ PPW_DEVICE_EXT Ext, _In_opt_ PCUNICODE_STRING Image)
{
    BOOLEAN blocked = FALSE;
    ULONG i;

    if (Image == NULL || Image->Buffer == NULL || Image->Length < sizeof(WCHAR)) {
        return FALSE;
    }

    ExAcquirePushLockShared(&Ext->PolicyLock);
    for (i = 0; i < Ext->RuleCount && !blocked; i++) {
        UNICODE_STRING rule;
        rule.Buffer = Ext->Rules[i].ImageName;
        rule.Length = (USHORT)(Ext->Rules[i].ImageChars * sizeof(WCHAR));
        rule.MaximumLength = rule.Length;

        if ((Ext->Rules[i].Flags & PW_RULE_FLAG_SUFFIX) != 0) {
            if (Image->Length >= rule.Length) {
                UNICODE_STRING tail;
                tail.Buffer = (PWCHAR)((PUCHAR)Image->Buffer +
                                       (Image->Length - rule.Length));
                tail.Length = rule.Length;
                tail.MaximumLength = rule.Length;
                blocked = RtlEqualUnicodeString(&tail, &rule, TRUE);
            }
        } else {
            blocked = RtlEqualUnicodeString(Image, &rule, TRUE);
        }
    }
    ExReleasePushLockShared(&Ext->PolicyLock);
    return blocked;
}

static VOID PwProcessNotify(_Inout_ PEPROCESS Process, _In_ HANDLE ProcessId,
                            _Inout_opt_ PPS_CREATE_NOTIFY_INFO CreateInfo)
{
    PPW_DEVICE_EXT ext = g_PwExt;

    if (ext == NULL) {
        return;
    }

    InterlockedIncrement(&ext->ActiveCallbacks);
    if (ext->ShuttingDown == 0) {
        if (CreateInfo != NULL) {
            /* The name is valid only when the flag says so. Reading
             * ImageFileName unconditionally is a NULL or stale pointer. */
            PCUNICODE_STRING image = CreateInfo->FileOpenNameAvailable
                                         ? CreateInfo->ImageFileName : NULL;
            ULONG flags = 0;
            BOOLEAN blocked;

            if (image != NULL) flags |= PW_EVENT_FLAG_NAME_AVAILABLE;
            if (CreateInfo->CommandLine != NULL) flags |= PW_EVENT_FLAG_HAS_COMMAND_LINE;
            if (CreateInfo->IsSubsystemProcess) flags |= PW_EVENT_FLAG_SUBSYSTEM;

            blocked = PwIsBlocked(ext, image);
            if (blocked) {
                flags |= PW_EVENT_FLAG_BLOCKED;
                InterlockedIncrement(&ext->BlockedProcesses);
            }

            PwQueueEvent(ext, PwEventProcessCreate, ProcessId,
                         CreateInfo->ParentProcessId,
                         CreateInfo->CreatingThreadId.UniqueProcess,
                         CreateInfo->CreatingThreadId.UniqueThread,
                         blocked ? STATUS_ACCESS_DENIED : STATUS_SUCCESS,
                         flags, image);

            if (blocked) {
                CreateInfo->CreationStatus = STATUS_ACCESS_DENIED;
            }
        } else {
            PwQueueEvent(ext, PwEventProcessExit, ProcessId, NULL, NULL, NULL,
                         PsGetProcessExitStatus(Process), 0, NULL);
        }
    }

    if (InterlockedDecrement(&ext->ActiveCallbacks) == 0) {
        KeSetEvent(&ext->CallbacksDrained, IO_NO_INCREMENT, FALSE);
    }
}
```

Dispatch, unload, and entry point:

```c
static NTSTATUS PwCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    PwComplete(Irp, STATUS_SUCCESS, 0);
    return STATUS_SUCCESS;
}

static NTSTATUS PwDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PPW_DEVICE_EXT ext = (PPW_DEVICE_EXT)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    NTSTATUS status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR info = 0;
    KIRQL irql;

    if (ext->ShuttingDown) {
        status = STATUS_DELETE_PENDING;
        goto Complete;
    }

    switch (sp->Parameters.DeviceIoControl.IoControlCode) {
    case IOCTL_PW_WAIT_FOR_EVENT: {
        ULONG outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
        if (sp->Parameters.DeviceIoControl.InputBufferLength != 0) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        if (outLen < PW_BATCH_HEADER_SIZE + sizeof(PW_EVENT)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        if (outLen > PW_MAX_BATCH_BYTES ||
            Irp->AssociatedIrp.SystemBuffer == NULL) {
            status = STATUS_INVALID_BUFFER_SIZE;
            break;
        }
        IoMarkIrpPending(Irp);           /* before anyone can complete it */
        IoCsqInsertIrp(&ext->Csq, Irp, NULL);
        PwPump(ext);                     /* events may already be queued */
        return STATUS_PENDING;
    }

    case IOCTL_PW_ADD_RULE: {
        PPW_RULE in;
        ULONG count;
        if (sp->Parameters.DeviceIoControl.InputBufferLength != sizeof(PW_RULE) ||
            sp->Parameters.DeviceIoControl.OutputBufferLength < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        in = (PPW_RULE)Irp->AssociatedIrp.SystemBuffer;
        if (in->ImageChars == 0 || in->ImageChars >= PW_MAX_RULE_CHARS ||
            in->ImageName[in->ImageChars] != L'\0' ||
            (in->Flags & ~PW_RULE_FLAG_SUFFIX) != 0) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        ExAcquirePushLockExclusive(&ext->PolicyLock);
        if (ext->RuleCount >= PW_MAX_RULES) {
            ExReleasePushLockExclusive(&ext->PolicyLock);
            status = STATUS_INSUFFICIENT_RESOURCES;
            break;
        }
        ext->Rules[ext->RuleCount] = *in;   /* full struct copy, then count */
        count = ++ext->RuleCount;
        ExReleasePushLockExclusive(&ext->PolicyLock);
        *(PULONG)Irp->AssociatedIrp.SystemBuffer = count;
        info = sizeof(ULONG);
        status = STATUS_SUCCESS;
        break;
    }

    case IOCTL_PW_CLEAR_RULES:
        ExAcquirePushLockExclusive(&ext->PolicyLock);
        ext->RuleCount = 0;
        ExReleasePushLockExclusive(&ext->PolicyLock);
        status = STATUS_SUCCESS;
        break;

    case IOCTL_PW_QUERY_STATS: {
        PPW_STATS stats;
        if (sp->Parameters.DeviceIoControl.OutputBufferLength < sizeof(PW_STATS)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        stats = (PPW_STATS)Irp->AssociatedIrp.SystemBuffer;
        RtlZeroMemory(stats, sizeof(*stats));
        stats->Version = PW_PROTOCOL_VERSION;
        KeAcquireSpinLock(&ext->Lock, &irql);
        stats->QueuedEvents = ext->EventCount;
        KeReleaseSpinLock(&ext->Lock, irql);
        stats->DroppedEvents = (ULONG)InterlockedCompareExchange(
                                   &ext->DroppedEvents, 0, 0);
        stats->BlockedProcesses = (ULONG)InterlockedCompareExchange(
                                   &ext->BlockedProcesses, 0, 0);
        stats->DeliveredEvents = (ULONG)InterlockedCompareExchange(
                                   &ext->DeliveredEvents, 0, 0);
        ExAcquirePushLockShared(&ext->PolicyLock);
        stats->RuleCount = ext->RuleCount;
        ExReleasePushLockShared(&ext->PolicyLock);
        info = sizeof(PW_STATS);
        status = STATUS_SUCCESS;
        break;
    }

    default:
        break;
    }

Complete:
    PwComplete(Irp, status, info);
    return status;
}

static VOID PwUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PPW_DEVICE_EXT ext = g_PwExt;
    PDEVICE_OBJECT deviceObject = DriverObject->DeviceObject;
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(PW_SYMLINK_NAME);
    LIST_ENTRY orphans;
    KIRQL irql;

    InterlockedExchange(&ext->ShuttingDown, 1);
    (VOID)PsSetCreateProcessNotifyRoutineEx(PwProcessNotify, TRUE);

    /* No callback can start now; wait out the ones already running. */
    for (;;) {
        if (InterlockedCompareExchange(&ext->ActiveCallbacks, 0, 0) == 0) {
            break;
        }
        KeClearEvent(&ext->CallbacksDrained);
        if (InterlockedCompareExchange(&ext->ActiveCallbacks, 0, 0) == 0) {
            break;
        }
        KeWaitForSingleObject(&ext->CallbacksDrained, Executive, KernelMode,
                              FALSE, NULL);
    }

    for (;;) {
        PIRP irp = IoCsqRemoveNextIrp(&ext->Csq, NULL);
        if (irp == NULL) {
            break;
        }
        PwComplete(irp, STATUS_DELETE_PENDING, 0);
    }

    InitializeListHead(&orphans);
    KeAcquireSpinLock(&ext->Lock, &irql);
    while (!IsListEmpty(&ext->Events)) {
        InsertTailList(&orphans, RemoveHeadList(&ext->Events));
        ext->EventCount--;
    }
    KeReleaseSpinLock(&ext->Lock, irql);
    while (!IsListEmpty(&orphans)) {
        PPW_EVENT_NODE node =
            CONTAINING_RECORD(RemoveHeadList(&orphans), PW_EVENT_NODE, Link);
        ExFreePoolWithTag(node, PW_TAG);
    }

    if (ext->Rules != NULL) {
        ExFreePoolWithTag(ext->Rules, PW_TAG);
        ext->Rules = NULL;
    }
    IoDeleteSymbolicLink(&symLink);
    IoDeleteDevice(deviceObject);
    g_PwExt = NULL;
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    PDEVICE_OBJECT deviceObject = NULL;
    PPW_DEVICE_EXT ext;
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(PW_DEVICE_NAME);
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(PW_SYMLINK_NAME);

    UNREFERENCED_PARAMETER(RegistryPath);

    status = IoCreateDevice(DriverObject, sizeof(PW_DEVICE_EXT), &deviceName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            TRUE, &deviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PPW_DEVICE_EXT)deviceObject->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));
    KeInitializeSpinLock(&ext->Lock);
    InitializeListHead(&ext->Events);
    InitializeListHead(&ext->Waiters);
    ExInitializePushLock(&ext->PolicyLock);
    KeInitializeEvent(&ext->CallbacksDrained, NotificationEvent, FALSE);
    IoCsqInitialize(&ext->Csq, PwCsqInsertIrp, PwCsqRemoveIrp, PwCsqPeekNextIrp,
                    PwCsqAcquireLock, PwCsqReleaseLock, PwCsqCompleteCanceledIrp);

    ext->Rules = (PPW_RULE)ExAllocatePool2(POOL_FLAG_NON_PAGED,
                                           PW_MAX_RULES * sizeof(PW_RULE), PW_TAG);
    if (ext->Rules == NULL) {
        IoDeleteDevice(deviceObject);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    status = IoCreateSymbolicLink(&symLink, &deviceName);
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(ext->Rules, PW_TAG);
        IoDeleteDevice(deviceObject);
        return status;
    }

    g_PwExt = ext;
    status = PsSetCreateProcessNotifyRoutineEx(PwProcessNotify, FALSE);
    if (!NT_SUCCESS(status)) {
        g_PwExt = NULL;
        IoDeleteSymbolicLink(&symLink);
        ExFreePoolWithTag(ext->Rules, PW_TAG);
        IoDeleteDevice(deviceObject);
        return status;
    }

    DriverObject->DriverUnload = PwUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = PwCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = PwCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = PwDeviceControl;
    return STATUS_SUCCESS;
}
```

```c
/* ProcWatchClient.c -- console consumer; Ctrl+C cancels the pending wait. */
#include <windows.h>
#include <winioctl.h>
#include <stdio.h>
#include <stdlib.h>
#include "ProcWatchPublic.h"

static HANDLE g_Device = INVALID_HANDLE_VALUE;
static volatile LONG g_Stop;

static BOOL WINAPI OnCtrlC(DWORD Type)
{
    UNREFERENCED_PARAMETER(Type);
    InterlockedExchange(&g_Stop, 1);
    if (g_Device != INVALID_HANDLE_VALUE) {
        CancelIoEx(g_Device, NULL);   /* handler runs on its own thread */
    }
    return TRUE;
}

static void PrintTime(const LARGE_INTEGER *Time)
{
    FILETIME ft, local;
    SYSTEMTIME st;
    ft.dwLowDateTime = Time->LowPart;
    ft.dwHighDateTime = Time->HighPart;
    FileTimeToLocalFileTime(&ft, &local);
    FileTimeToSystemTime(&local, &st);
    printf("%02u:%02u:%02u.%03u ", (unsigned)st.wHour, (unsigned)st.wMinute,
           (unsigned)st.wSecond, (unsigned)st.wMilliseconds);
}

static void PrintBatch(const PW_BATCH *Batch, DWORD Bytes)
{
    ULONG i, count = Batch->EventCount;

    if (PW_BATCH_BYTES(count) > Bytes) {
        count = (Bytes - PW_BATCH_HEADER_SIZE) / sizeof(PW_EVENT);
    }
    if (Batch->DroppedEvents != 0) {
        printf("[%u events dropped by the driver]\n", Batch->DroppedEvents);
    }
    for (i = 0; i < count; i++) {
        const PW_EVENT *e = &Batch->Events[i];
        PrintTime(&e->Time);
        if (e->Type == PwEventProcessCreate) {
            printf("create pid=%u ppid=%u creator=%u:%u %ls%s\n", e->ProcessId,
                   e->ParentProcessId, e->CreatingProcessId, e->CreatingThreadId,
                   e->ImageChars ? e->ImageName : L"(no name)",
                   (e->Flags & PW_EVENT_FLAG_BLOCKED) ? "  [BLOCKED]" : "");
        } else {
            printf("exit   pid=%u status=0x%08X\n", e->ProcessId,
                   (unsigned)e->Status);
        }
    }
    fflush(stdout);
}

static BOOL AddRule(HANDLE Device, const wchar_t *Pattern)
{
    PW_RULE *rule = (PW_RULE *)calloc(1, sizeof(PW_RULE));
    DWORD bytes = 0, count = 0;
    size_t len = wcslen(Pattern);
    BOOL ok;

    if (rule == NULL || len == 0 || len >= PW_MAX_RULE_CHARS) {
        free(rule);
        return FALSE;
    }
    rule->Flags = PW_RULE_FLAG_SUFFIX;   /* match the file name, not the path */
    rule->ImageChars = (ULONG)len;
    wcscpy_s(rule->ImageName, PW_MAX_RULE_CHARS, Pattern);
    ok = DeviceIoControl(Device, IOCTL_PW_ADD_RULE, rule, sizeof(*rule),
                         &count, sizeof(count), &bytes, NULL);
    printf("rule \"%ls\" -> %ls (rules now %u)\n", Pattern,
           ok ? L"added" : L"rejected", ok ? count : 0);
    free(rule);
    return ok;
}

int wmain(int argc, wchar_t **argv)
{
    PW_BATCH *batch;
    int i;

    g_Device = CreateFileW(PW_DOS_NAME, GENERIC_READ | GENERIC_WRITE, 0, NULL,
                           OPEN_EXISTING, 0, NULL);
    if (g_Device == INVALID_HANDLE_VALUE) {
        printf("CreateFile failed: %lu\n", GetLastError());
        return 1;
    }
    for (i = 1; i < argc; i++) {
        AddRule(g_Device, argv[i]);
    }

    batch = (PW_BATCH *)HeapAlloc(GetProcessHeap(), 0, PW_MAX_BATCH_BYTES);
    if (batch == NULL) {
        CloseHandle(g_Device);
        return 1;
    }
    SetConsoleCtrlHandler(OnCtrlC, TRUE);

    while (g_Stop == 0) {
        DWORD bytes = 0;
        if (!DeviceIoControl(g_Device, IOCTL_PW_WAIT_FOR_EVENT, NULL, 0, batch,
                             PW_MAX_BATCH_BYTES, &bytes, NULL)) {
            DWORD error = GetLastError();
            if (error != ERROR_OPERATION_ABORTED) {
                printf("wait failed: %lu\n", error);
            }
            break;
        }
        if (bytes >= PW_BATCH_HEADER_SIZE) {
            PrintBatch(batch, bytes);
        }
    }

    HeapFree(GetProcessHeap(), 0, batch);
    CloseHandle(g_Device);
    return 0;
}
```

## Variations and advanced use

- **Shared-memory ring instead of an IOCTL.** Map a section into both the driver and the client and have the callback write straight into it. Latency drops to the cost of a memcpy, but you must own the section lifetime, handle a client that dies mid-write, and give up the natural back-pressure that a bounded IRP queue provides.
- **`PsSetCreateProcessNotifyRoutineEx2` for WSL1.** Register with `PsCreateProcessNotifySubsystems` to see Pico processes. The same routine cannot be registered with both `Ex` and `Ex2` (`STATUS_INVALID_PARAMETER`), so pick one and gate on the `IsSubsystemProcess` flag.
- **Veto at image load instead of process creation.** A minifilter blocking `IRP_MJ_ACQUIRE_FOR_SECTION_SYNCHRONIZATION` refuses the image before the process exists, which also covers `LoadLibrary` in an existing process; it costs an altitude, a registered filter, and `fltmc` deployment.
- **Fan-out to many consumers.** Replace `Exclusive = TRUE` with a per-handle context and one queue per handle, registered through an `IO_CSQ` per client. Do not simply allow multiple openers on one queue: a batch taken by client A is invisible to client B.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `PsSetCreateProcessNotifyRoutineEx` returns `0xC0000022` | Driver not linked with `/INTEGRITYCHECK` | `dumpbin /headers x.sys \| findstr /i integrity`; add `/INTEGRITYCHECK` to the linker command line |
| Returns `0xC000000D` | Same routine registered twice, or registered with both `Ex` and `Ex2` | Unregister first; use one variant |
| Returns `0xC000009A` | The 64-slot notify table is full | Unload other monitors; `!drvobj` your driver to confirm a single registration |
| `0xCE DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | Pending wait IRPs left when the image unloaded | Drain with `IoCsqRemoveNextIrp` before `IoDeleteDevice`; `!irpfind -a` before unload |
| `0x9F` with parameter 1 = 3 | A device object held an IRP across a power transition; the pending wait IRP was never completed on a device that sits in a stack | `!analyze -v`, then `!devobj <param3>` and `!irp` on the stuck IRP |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA` | Read `ImageFileName` while `FileOpenNameAvailable` was clear; stored a `UNICODE_STRING` and used it after the callback returned; double free of a queue node | Copy by `Length` inside the callback; check the flag before the pointer |
| `0xD1 DRIVER_IRQL_NOT_LESS_OR_EQUAL` | CSQ lock callback touched paged memory or called a pageable routine at `DISPATCH_LEVEL` | Keep the CSQ callbacks to `KeAcquireSpinLock`/`KeReleaseSpinLock` only |
| Hang, sometimes `0xC2`, right after a launch | Callback blocked on a lock held by a thread that is waiting for that same process creation, or waited for an APC while APCs were disabled | `!analyze -hang`, `!stacks 2`, `!locks`; remove all waits from the callback |
| Client waits forever after a burst | Queue overflowed and the drop policy discarded the events the client was waiting for | Read `PW_STATS.DroppedEvents`; raise `PW_MAX_EVENTS` or drain faster |
| Veto has no effect | Legacy `PsSetCreateProcessNotifyRoutine` registered (no `CreateInfo`), or `CreationStatus` set to a success value | Register the `Ex` variant; always write a failing `NTSTATUS` |
| A suffix rule matches nothing | The rule is a full path in DOS form while `ImageFileName` is NT form (`\Device\HarddiskVolumeN\...`) | Use suffix rules, or normalize both sides before comparing |
| Second client gets `ERROR_ACCESS_DENIED` | Device created with `Exclusive = TRUE` | Intended for a single consumer; implement per-handle queues for fan-out |

## Safety, IRQL, and cleanup checklist

- [ ] `/INTEGRITYCHECK` is in the link line and verified with `dumpbin`, not assumed.
- [ ] The callback allocates at most one non-paged block, holds no lock across the allocation, and performs no file, registry, or network I/O.
- [ ] `ImageFileName` and `CommandLine` are read only when their flags allow it, copied by `Length`, and never stored as pointers.
- [ ] A vetoed process is **not** added to any tracking table; do not rely on an exit notification for a creation you aborted.
- [ ] Every queue node is non-paged, tagged, and freed exactly once, in the drain loop or by the delivery path.
- [ ] The spinlock protects events, waiters, and `ShuttingDown` only; the push lock protects the policy table only; they are never nested.
- [ ] `IoMarkIrpPending` precedes `IoCsqInsertIrp`, and the dispatch returns `STATUS_PENDING` without completing the IRP.
- [ ] Unload sets `ShuttingDown` **before** unregistering, and unregisters **before** freeing anything the callback touches.
- [ ] The drain waits on a counter plus a notification event, clearing and re-checking to close the lost-wakeup window.
- [ ] `IoDeleteSymbolicLink` runs before `IoDeleteDevice`, and the device is created with `FILE_DEVICE_SECURE_OPEN`.
- [ ] The default device ACL is not a security boundary; for a real product install the device with an explicit SDDL through `IoCreateDeviceSecure` (see `wk-object-manager`).

## Verification

1. **Build.** `msbuild ProcWatch.vcxproj /p:Configuration=Debug /p:Platform=x64`. Then `dumpbin /headers x64\Debug\ProcWatch.sys | findstr /i integrity` — expect `Integrity` in `DllCharacteristics`. A missing flag here explains every later `0xC0000022`.
2. **Load.** On a test-signed VM: `sc create ProcWatch type= kernel binPath= C:\Test\ProcWatch.sys` then `sc start ProcWatch`. `sc start` failing with `0xC0000022` means the integrity flag, not signing.
3. **Prove the callback fires.** `notepad.exe`, then in the client you should see a `create` line with the real image path and a matching `exit` line when Notepad closes. If `create` appears without `exit`, your exit branch is reading a field that is not valid for a terminating process.
4. **Prove the veto.** `ProcWatchClient.exe calc.exe`, then launch Calculator: the launch fails and the client prints `[BLOCKED]` with `STATUS_ACCESS_DENIED` (`0x%08X` = `C0000022`). Confirm the error dialog/text names access denied, not "file not found".
5. **Prove cancellation.** Start the client with no activity, press Ctrl+C while it is blocked in the wait. The driver's `PwCsqCompleteCanceledIrp` runs, the client prints nothing and exits, and `!irpfind -a` on the live kernel shows no leftover IRP for `\Device\ProcWatch`.
6. **Prove unload safety.** Stop the client, then `sc stop ProcWatch`. Expect a clean stop. Then reinstall, leave the client running, and run `sc stop ProcWatch` — the stop fails because the handle is open; that failure is the invariant your unload relies on.
7. **Stress.** Run a build (`cmd /c "for /l %i in (1,1,300) do @start /min cmd /c exit"`) with the client attached and Driver Verifier enabled: `verifier /standard /driver ProcWatch.sys`. Reboot, repeat, then `!verifier 3 ProcWatch` and check for `0xC1`/`0xC2`/leak reports.
8. **Leak check.** After unload: `!poolused 2 ProcWatch` in WinDbg should show no blocks for tag `PwcH`. A nonzero count is a node or rule array you did not free.
