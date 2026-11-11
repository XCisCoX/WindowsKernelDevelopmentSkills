---
name: wk-kernel-tracing
description: "Use when a driver must explain itself without stopping the machine: DbgPrint/DbgPrintEx component and level filtering with the Debug Print Filter registry value, KdPrint and NT_ASSERT behavior in release builds, DebugView capture, TraceLogging/ETW providers with logman, tracelog, tracefmt and WPA, and WPP DoTraceMessage tracing."
---

# Kernel tracing: making the driver speak without a debugger

A kernel debugger stops the world, which is exactly what you do not want for a timing bug, a
performance problem, or a fault that only appears on a customer's machine. Tracing is the other
half of driver observability: the driver emits records while it runs, a consumer outside the
driver stores or displays them, and the machine never stops.

There are three channels, and they are not interchangeable. The `DbgPrint` family writes text
into a kernel buffer that a debugger or DebugView drains, filtered by a per-component bitmask.
WPP and TraceLogging both ride Event Tracing for Windows (ETW), which means per-CPU buffers, a
real session, and a file you can decode later — WPP with printf-style messages plus a TMF
decoding file, TraceLogging with self-describing typed fields.

The rule that prevents most wasted hours: **decide the channel before writing the first trace
statement**. `DbgPrint` output that never appears because a mask is zero, or a WPP provider that
never registers because `WPP_INIT_TRACING` was skipped, costs more time than choosing ETW up
front and capturing from the start.

## When to use

- A driver misbehaves intermittently and attaching a debugger hides the bug or changes the
  timing.
- You need a record of what happened on a machine you cannot debug interactively.
- You want counts, sizes, status codes, and durations rather than free text, so the data can be
  aggregated in a spreadsheet or WPA.
- Debug output stopped appearing and you need to reason about component IDs, levels, and the
  `Debug Print Filter` mask instead of guessing.
- You are deciding between `DbgPrint`, WPP, and TraceLogging for a new driver component.
- A release build behaves differently from the debug build and you need tracing that survives
  into the release binary.
- You must prove an invariant in the field without a checked build, using assertions that are
  compiled out of release.

Do not use this skill to stop a live kernel or inspect an IRP at a breakpoint (load
`wk-kernel-debugging`), to analyse a crash dump (load `wk-crash-dump-analysis`), or to build and
deploy the driver in the first place (load `wk-driver-build-deploy`).

## Prerequisites and related skills

- **Driver target:** Windows 10 or later for `TraceLoggingProvider.h` and `ExAllocatePool2`;
  `DbgPrintEx` and the `DPFLTR_*` constants exist from Windows XP and later.
- **Headers:** `<ntddk.h>` or `<wdm.h>` for `DbgPrint`, `KdPrint`, and `NT_ASSERT`;
  `<ntassert.h>` for `NT_ASSERT`; `<evntrace.h>` and `<TraceLoggingProvider.h>` for
  TraceLogging. All four are in the WDK and the Windows SDK.
- **WDK version:** TraceLogging in the kernel needs a WDK that ships `TraceLoggingProvider.h`
  with kernel support (Windows 10 WDK and later). WPP needs the WDK WPP preprocessor and a
  project with WPP tracing enabled.
- **Privileges:** starting an ETW session needs an elevated command prompt (Administrators) or
  membership in `Performance Log Users`; DebugView needs elevation for kernel capture.
- **Symbols:** a PDB and, for WPP, the generated `.tmf` files, so the captured events decode to
  names and values rather than raw bytes.

Sibling skills: `wk-kernel-debugging` for breakpoints and kernel sessions;
`wk-driver-build-deploy` for building with WPP and signing the result; `wk-kernel-programming-rules`
for IRQL and status conventions referenced here; `wk-crash-dump-analysis` when the trace ends in
a bugcheck.

## Mental model

Tracing has a producer, an optional filter, and a consumer. The producer is your driver calling a
macro. The filter differs per channel, and getting it wrong is the usual reason for silence.

```
  driver code
     |
     +-- DbgPrint(Ex) -----> Kd_<Component>_Mask --> debugger port, or DebugView
     |                       (Debug Print Filter registry / ed Kd_*_Mask)
     |
     +-- KdPrint(Ex) -------> same path, compiled out of free builds
     |
     +-- NT_ASSERT ---------> int 3 to the debugger, bugcheck when none attached
     |
     +-- WPP DoTraceMessage -> ETW provider (control GUID + flag bits + level)
     |                              \
     +-- TraceLoggingWrite -> ETW provider (self-describing typed fields)
                                    |
                              ETW session: logman / tracelog
                                    |
                              .etl file --> tracefmt / tracerpt / WPA
```

Why the channels behave so differently:

- **`DbgPrint` is a kernel buffer plus an interrupt-like handoff.** With a debugger attached,
  every line is pushed across the transport and the target pays for it. Without a debugger, the
  text lands in the debug buffer and is discarded unless someone is draining it.
- **ETW is per-CPU ring buffers written without locks.** If no session enabled the provider,
  `TraceLoggingWrite` and `DoTraceMessage` reduce to an enabled check and a branch. That is why
  ETW is usable in hot paths where `DbgPrint` is not.
- **A provider is registered, not discovered.** `TraceLoggingRegister` publishes the provider so
  a session can enable it by GUID or name. Forgetting the unregister leaves a dangling callback
  into an unloaded image.
- **Decoding is separate from capture.** The `.etl` file holds binary events; `tracefmt`,
  `tracerpt`, or WPA turn them into text or columns. WPP needs the `.tmf` metadata; TraceLogging
  carries its own.

## Key APIs and structures

### Print APIs

| API | Purpose | IRQL | Notes |
| --- | --- | --- | --- |
| `DbgPrint(Format, arguments)` | Plain debug text, variadic | `PASSIVE_LEVEL` to `DISPATCH_LEVEL` | Uses component `DPFLTR_DEFAULT_ID` (101) at `DPFLTR_INFO_LEVEL` (3) |
| `DbgPrintEx(ComponentId, Level, Format, arguments)` | Filtered debug text, variadic | Any IRQL | Returns a `ULONG` that is really an `NTSTATUS` |
| `vDbgPrintEx(ComponentId, Level, Format, va_list)` | Variadic wrapper taking a `va_list` | Any IRQL | Use inside your own variadic helper |
| `vDbgPrintExWithPrefix(Prefix, ComponentId, Level, Format, va_list)` | Same, with a fixed prefix | Any IRQL | Best way to make your driver's lines greppable |
| `KdPrint` | Debug-only plain text | Any IRQL | Double-parenthesised argument list; compiles to nothing in free builds |
| `KdPrintEx` | Debug-only filtered text | Any IRQL | Same form: `KdPrintEx((ComponentId, Level, Format, arguments))` |
| `DbgSetDebugFilterState(ComponentId, Level, Enable)` | Change a component mask at run time | `PASSIVE_LEVEL` | Declared in `wdm.h`, undocumented; same as the native call |

Component IDs, from `<dpfilter.h>`: `DPFLTR_IHVDRIVER_ID` = 77 (use this for a normal driver),
`DPFLTR_IHVVIDEO_ID` = 78, `DPFLTR_IHVAUDIO_ID` = 79, `DPFLTR_IHVNETWORK_ID` = 80,
`DPFLTR_IHVSTREAMING_ID` = 81, `DPFLTR_IHVBUS_ID` = 82, `DPFLTR_DEFAULT_ID` = 101.

Levels: `DPFLTR_ERROR_LEVEL` = 0, `DPFLTR_WARNING_LEVEL` = 1, `DPFLTR_TRACE_LEVEL` = 2,
`DPFLTR_INFO_LEVEL` = 3.

**The filter arithmetic:** a `Level` below 32 is converted to the bit `1 << Level`; a `Level` of
32 or more is used as a mask directly. The message is emitted only when that value ANDed with
the component's current mask is non-zero. `DbgPrint` therefore needs bit 3 (`0x8`) in the
`DEFAULT` mask, which is why every "DbgPrint does not appear" fix sets `DEFAULT` to `8`.

| Mask source | How to set it | Takes effect |
| --- | --- | --- |
| Registry | `HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter`, `DWORD` value named after the component without prefix or suffix (`DEFAULT`, `IHVDRIVER`) | Next boot |
| Kernel debugger | `ed Kd_DEFAULT_Mask 0x8`, `ed Kd_IHVDRIVER_Mask 0xf` | Immediately |
| Native API | `NtSetDebugFilterState`, or the kernel `DbgSetDebugFilterState` | Immediately; user-mode callers need `SeDebugPrivilege` |

### Assertion and stop APIs

| API | Purpose | Release build | Notes |
| --- | --- | --- | --- |
| `NT_ASSERT(expr)` | Verify an invariant | Expression not evaluated | Breakpoint with a debugger, bugcheck without; never put side effects in it |
| `NT_ASSERTMSG(msg, expr)` | Same with a message | Not evaluated | Message appears in the dump analysis |
| `NT_ASSERTMSGW(msg, expr)` | Wide-string variant | Not evaluated | Same semantics |
| `ASSERT(expr)` | Legacy assert from `wdm.h` | Not evaluated | Expands to `RtlAssert`; `NT_ASSERT` is preferred in new code |
| `RtlAssert(FailedAssertion, FileName, LineNumber, Message)` | The underlying routine | Not called | Bugchecks when no debugger is attached |
| `DbgBreakPoint()` | `int 3` into the debugger | Still present | No debugger means a breakpoint exception the kernel cannot handle |
| `DbgBreakPointWithStatus(Status)` | Break with a status code | Still present | Used by `RtlAssert` |
| `KeBugCheckEx(Code, P1, P2, P3, P4)` | Deliberate, unrecoverable stop | Present | Last resort; the four parameters show up in `!analyze -v` |

### ETW and TraceLogging APIs

| API / macro | Purpose | Notes |
| --- | --- | --- |
| `TRACELOGGING_DEFINE_PROVIDER(var, name, guid)` | Declares the provider handle | GUID as `(d1, d2, d3, b1, b2, b3, b4, b5, b6, b7, b8)` |
| `TraceLoggingRegister(provider)` | Publishes the provider | Call in `DriverEntry` before the first write |
| `TraceLoggingUnregister(provider)` | Withdraws it | Call in unload before the image can go away |
| `TraceLoggingWrite(provider, event, fields)` | Emits one event | Safe up to `DISPATCH_LEVEL` |
| `TraceLoggingLevel(level)` | Sets event level | `TRACE_LEVEL_ERROR` 2, `WARNING` 3, `INFORMATION` 4, `VERBOSE` 5 |
| `TraceLoggingKeyword(mask)` | Sets keyword bits | Session filters on these |
| `TraceLoggingValue(value, "Name")` | Typed field, inferred | Most convenient; type comes from the argument |
| `TraceLoggingUInt32/Int32/UInt64(value, "Name")` | Explicit integer field | Use when the width matters |
| `TraceLoggingUnicodeString(ptr, "Name")` | `UNICODE_STRING` field | Takes a pointer, not the structure |
| `TraceLoggingString(ptr, "Name")` | ANSI string field | Pointer plus implicit length |
| `TraceLoggingBinary(ptr, bytes, "Name")` | Raw bytes field | Pair with a length field for decoding |
| `TraceLoggingNTStatus(status, "Name")` | `NTSTATUS` field | Decodes to the symbolic name |
| `TRACE_LEVEL_*` from `<evntrace.h>` | Level constants | 1 critical, 2 error, 3 warning, 4 information, 5 verbose |

### WPP macros

| Macro | Purpose | Notes |
| --- | --- | --- |
| `WPP_CONTROL_GUIDS` | Declares the control GUID and flag bits | Must appear before `trace.tmh` is included |
| `WPP_DEFINE_CONTROL_GUID(name, guid, bits)` | One provider | `name` is used by `WPP_CONTROL` |
| `WPP_DEFINE_BIT(FLAG)` | One flag bit | First bit is `0x1`, then `0x2`, `0x4`, in declaration order |
| `WPP_INIT_TRACING(DriverObject, RegistryPath)` | Registers with ETW | First statement in `DriverEntry` |
| `DoTraceMessage(FLAG, fmt, arguments)` | Emits a formatted message | Enabled by the flag mask and the session level |
| `WPP_CLEANUP(DriverObject)` | Unregisters | Last statement in unload, after in-flight work drains |
| `WPP_LEVEL_FLAGS_ENABLED(level, flags)` | Enable test for a level-aware wrapper | Pair with `WPP_LEVEL_FLAGS_LOGGER`; see the WPP note below |

## Step-by-step

1. **Make `DbgPrint` visible on the target.** Add the `DEFAULT` value (8 for information-level
   `DbgPrint`, `0xF` for the first four levels) and, if you use `DbgPrintEx` with
   `DPFLTR_IHVDRIVER_ID`, an `IHVDRIVER` value:

   ```
   reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter" /v DEFAULT /t REG_DWORD /d 0x8 /f
   reg add "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter" /v IHVDRIVER /t REG_DWORD /d 0xF /f
   ```

   *Check:* `reg query` on that key lists both values. The registry is read at boot,
   so reboot before expecting a change; a debugger can change the mask immediately with
   `ed Kd_IHVDRIVER_Mask 0xf`.

2. **Verify the mask actually took effect.** With a kernel debugger attached, `ed
   Kd_DEFAULT_Mask 0x8` and then `DbgPrint` from a test driver. *Check:* the text appears in the
   debugger output as soon as the driver runs.

3. **Capture without a debugger.** Run DebugView elevated on the target, enable `Capture Kernel`
   (Ctrl+K) and `Enable Verbose Kernel Output`. *Check:* existing kernel debug output appears;
   if not, the tool is not elevated or the mask is still zero.

4. **Give every line the same prefix and an explicit level.** Either wrap `vDbgPrintEx` and
   `vDbgPrintExWithPrefix` in a variadic helper, or start each format string with the driver
   name as the example below does. *Check:* every captured line starts with the prefix, so
   DebugView's filter can isolate the driver.

5. **Decide whether ETW is needed.** If you need typed fields, long captures, or release-build
   diagnostics, add TraceLogging. *Check:* you can state which events you need and what fields
   each carries before writing code.

6. **Register the provider in `DriverEntry` and unregister in unload.** `TraceLoggingRegister`
   first, `TraceLoggingUnregister` last. *Check:* `logman query providers "{GUID}"` shows the
   provider with `Enabled` state `0` before any session and the process ID of the target after
   the driver loads.

7. **Start a session by GUID, run the workload, stop it:**

   ```
   logman create trace WdTrace -p "{7c9e6d2a-4b31-4f88-9a15-2d6b8e0c3f47}" 0xffffffff 5 -o C:\traces\wdtrace.etl -ets
   logman query WdTrace -ets
   ```

   *Check:* `logman query` shows the session running, the provider GUID, and the output file.

8. **Reproduce the behavior and stop the session:**

   ```
   logman stop WdTrace -ets
   logman delete WdTrace
   ```

   *Check:* `C:\traces\wdtrace.etl` exists and has a non-trivial size.

9. **Decode the capture.** `tracerpt C:\traces\wdtrace.etl -o C:\traces\wdtrace.csv -of CSV -y`
   for columns, `tracefmt C:\traces\wdtrace.etl -o C:\traces\wdtrace.txt` for text, or
   `wpa C:\traces\wdtrace.etl` for a timeline. *Check:* event names, field names, and values are
   readable; raw hex means the decoder lacks metadata.

10. **Confirm the fields and levels are what you designed.** In the decoded output check the
    event name, level, keyword, and each property. *Check:* a verbose event is absent when the
    session was started with level 4 and present with level 5.

11. **Add WPP only if the driver already uses it or you need printf-style decoding with a TMF.**
    Create `trace.h` with the control GUID and flags, include it in the driver, add
    `WPP_INIT_TRACING`, `DoTraceMessage` calls, and `WPP_CLEANUP`. *Check:* the build produces
    `trace.tmh` and `.tmf` files.

12. **Capture WPP by control GUID and decode with the TMF path:**

    ```
    tracelog -start WdWpp -guid #{2f4a1b2c-8d3e-4c57-a162-9b4d7e305188} -f C:\traces\wpp.etl -flags 0x3 -level 4
    tracelog -stop WdWpp
    tracefmt C:\traces\wpp.etl -o C:\traces\wpp.txt -p C:\Test\Wpp\x64\Debug
    ```

    *Check:* the text output contains your format strings with arguments substituted, not a GUID
    and a hex dump.

13. **Leave the tracing in the release build deliberately.** TraceLogging and WPP stay compiled
    in; `NT_ASSERT` and `KdPrint` do not. *Check:* `dumpbin /imports wdtrace.sys` or a strings
    scan finds the provider name in the release binary, and a release capture produces events.

## Complete example

A driver with TraceLogging, leveled `DbgPrintEx` lines, and a real assertion, plus a client
that exercises it and the capture and decode commands.

```c
/* WdTrace.c - TraceLogging provider, levelled debug print, and an assertion */
#include <ntddk.h>
#include <evntrace.h>
#include <TraceLoggingProvider.h>

#define WD_KEYWORD_IO   0x1
#define WD_KEYWORD_STATE 0x2
#define WD_MAX_PAYLOAD  0x100

TRACELOGGING_DEFINE_PROVIDER(
    g_WdProvider,
    "WdTrace",
    (0x7c9e6d2a, 0x4b31, 0x4f88, 0x9a, 0x15, 0x2d, 0x6b, 0x8e, 0x0c, 0x3f, 0x47));

typedef struct _WD_DEVICE_EXTENSION {
    ULONG   Writes;
    UCHAR   LastPayload[WD_MAX_PAYLOAD];
    ULONG   LastLength;
} WD_DEVICE_EXTENSION, *PWD_DEVICE_EXTENSION;

static VOID WdUnload(PDRIVER_OBJECT DriverObject);
static NTSTATUS WdCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp);
static NTSTATUS WdWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp);

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(L"\\Device\\WdTrace");
    UNICODE_STRING symLink    = RTL_CONSTANT_STRING(L"\\??\\WdTrace");
    PDEVICE_OBJECT deviceObject = NULL;
    NTSTATUS status;

    TraceLoggingRegister(g_WdProvider);
    TraceLoggingWrite(g_WdProvider, "DriverEntry",
                      TraceLoggingLevel(TRACE_LEVEL_INFORMATION),
                      TraceLoggingKeyword(WD_KEYWORD_STATE),
                      TraceLoggingUnicodeString(RegistryPath, "RegistryPath"));

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "WdTrace: DriverEntry path %wZ\n", RegistryPath);

    status = IoCreateDevice(DriverObject, sizeof(WD_DEVICE_EXTENSION), &deviceName,
                            FILE_DEVICE_UNKNOWN, 0, FALSE, &deviceObject);
    if (!NT_SUCCESS(status)) {
        TraceLoggingWrite(g_WdProvider, "DeviceCreateFailed",
                          TraceLoggingLevel(TRACE_LEVEL_ERROR),
                          TraceLoggingNTStatus(status, "Status"));
        TraceLoggingUnregister(g_WdProvider);
        return status;
    }

    deviceObject->Flags |= DO_BUFFERED_IO;
    deviceObject->Flags &= ~DO_DEVICE_INITIALIZING;

    status = IoCreateSymbolicLink(&symLink, &deviceName);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(deviceObject);
        TraceLoggingUnregister(g_WdProvider);
        return status;
    }

    NT_ASSERT(DriverObject->DeviceObject == deviceObject);

    DriverObject->DriverUnload = WdUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = WdCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]  = WdCreateClose;
    DriverObject->MajorFunction[IRP_MJ_WRITE]  = WdWrite;
    return STATUS_SUCCESS;
}

static VOID WdUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\WdTrace");

    IoDeleteSymbolicLink(&symLink);
    if (DriverObject->DeviceObject != NULL) {
        IoDeleteDevice(DriverObject->DeviceObject);
    }
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "WdTrace: unload\n");
    TraceLoggingUnregister(g_WdProvider);
}

static NTSTATUS WdCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS WdWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PWD_DEVICE_EXTENSION ext = (PWD_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    ULONG length = sp->Parameters.Write.Length;
    PUCHAR buffer = (PUCHAR)Irp->AssociatedIrp.SystemBuffer;
    ULONG copy = 0;
    NTSTATUS status = STATUS_SUCCESS;

    if (length == 0 || buffer == NULL) {
        status = STATUS_INVALID_PARAMETER;
    } else {
        copy = min(length, (ULONG)WD_MAX_PAYLOAD);
        RtlCopyMemory(ext->LastPayload, buffer, copy);
        ext->LastLength = copy;
        ext->Writes++;
    }

    TraceLoggingWrite(g_WdProvider, "Write",
                      TraceLoggingLevel(NT_SUCCESS(status) ? TRACE_LEVEL_INFORMATION
                                                           : TRACE_LEVEL_ERROR),
                      TraceLoggingKeyword(WD_KEYWORD_IO),
                      TraceLoggingUInt32(length, "RequestedLength"),
                      TraceLoggingUInt32(copy, "CopiedLength"),
                      TraceLoggingUInt32(ext->Writes, "WriteCount"),
                      TraceLoggingBinary(ext->LastPayload, copy, "Payload"),
                      TraceLoggingNTStatus(status, "Status"));

    DbgPrintEx(DPFLTR_IHVDRIVER_ID,
               NT_SUCCESS(status) ? DPFLTR_INFO_LEVEL : DPFLTR_ERROR_LEVEL,
               "WdTrace: write length %u copied %u count %u status 0x%08X\n",
               length, copy, ext->Writes, status);

    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = NT_SUCCESS(status) ? length : 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

```c
/* wdtclient.c - writes a payload to \\.\WdTrace; cl /W4 wdtclient.c */
#include <windows.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
    const char *payload = (argc > 1) ? argv[1] : "tracing payload";
    DWORD length = (DWORD)strlen(payload), written = 0;
    HANDLE h = CreateFileW(L"\\\\.\\WdTrace", GENERIC_WRITE, 0, NULL,
                           OPEN_EXISTING, 0, NULL);
    if (h == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }
    WriteFile(h, payload, length, &written, NULL);
    printf("wrote %lu of %lu\n", written, length);
    CloseHandle(h);
    return 0;
}
```

Deploy, then capture on the target from an elevated prompt:

```
sc create wdtrace type= kernel binPath= C:\Test\wdtrace.sys
sc start wdtrace
copy wdtclient.exe C:\Test\
logman create trace WdTrace -p "{7c9e6d2a-4b31-4f88-9a15-2d6b8e0c3f47}" 0xffffffff 5 -o C:\traces\wdtrace.etl -ets
logman query providers "{7c9e6d2a-4b31-4f88-9a15-2d6b8e0c3f47}"
C:\Test\wdtclient.exe "trace me"
logman stop WdTrace -ets
logman delete WdTrace
tracerpt C:\traces\wdtrace.etl -o C:\traces\wdtrace.csv -of CSV -y
tracefmt C:\traces\wdtrace.etl -o C:\traces\wdtrace.txt
```

`logman query providers` proves registration before you capture: the provider line shows the
GUID, the process ID of the loaded driver, and the keywords and levels it accepts. `tracerpt`
then produces a CSV whose event names come from the `TraceLoggingWrite` calls (`DriverEntry`,
`Write`) and whose columns carry `RequestedLength`, `CopiedLength`, `WriteCount`, `Payload`, and
`Status`. `tracefmt` prints the same events as text, one line per event, with the fields in
name-value form. The text below is illustrative of the shape, not of a specific machine:

```
WdTrace/Write  INFO  Keyword=0x1  RequestedLength=9 CopiedLength=9 WriteCount=1
   Payload=[74 72 61 63 65 20 6d 65] Status=STATUS_SUCCESS
```

WPA opens the same file for a timeline view: add the `Events` table or a generic events graph
and filter by provider name to see when each `Write` happened.

### WPP variant

If the driver must use WPP (an existing codebase, or a team standard), the control GUID lives in
`trace.h` and the build generates `trace.tmh` from it.

```c
/* trace.h - WPP control GUID and flag bits */
#define WPP_CONTROL_GUIDS \
    WPP_DEFINE_CONTROL_GUID(WdWppGuid, \
        (0x2f4a1b2c, 0x8d3e, 0x4c57, 0xa1, 0x62, 0x9b, 0x4d, 0x7e, 0x30, 0x51, 0x88), \
        WPP_DEFINE_BIT(FLAG_ERROR) \
        WPP_DEFINE_BIT(FLAG_INFO))

#include "trace.tmh"
```

```c
/* WppDemo.c - minimal driver using WPP; WPP tracing must be enabled in the project */
#include <ntddk.h>
#include "trace.h"

static VOID WppUnload(PDRIVER_OBJECT DriverObject);

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    WPP_INIT_TRACING(DriverObject, RegistryPath);
    DoTraceMessage(FLAG_INFO, "DriverEntry path %wZ", RegistryPath);
    DriverObject->DriverUnload = WppUnload;
    return STATUS_SUCCESS;
}

static VOID WppUnload(PDRIVER_OBJECT DriverObject)
{
    DoTraceMessage(FLAG_INFO, "unload");
    WPP_CLEANUP(DriverObject);
}
```

`DoTraceMessage` is filtered by its flag bit and by the level the session requests. Per-call
levels need a level-aware wrapper: declare it in a `// begin_wpp config` block in `trace.h`,
define `WPP_LEVEL_FLAGS_LOGGER` and `WPP_LEVEL_FLAGS_ENABLED` so the generated code compares the
session level, then call the wrapper with an explicit `TRACE_LEVEL_*` value.

Enable WPP in the driver project (Wpp Tracing, Run Wpp tracing = Yes) so the build scans
`trace.h`, emits `trace.tmh`, and leaves `.tmf` files next to the binary. Capture and decode:

```
tracelog -start WdWpp -guid #{2f4a1b2c-8d3e-4c57-a162-9b4d7e305188} -f C:\traces\wpp.etl -flags 0x3 -level 4 -rt
tracelog -stop WdWpp
tracefmt C:\traces\wpp.etl -o C:\traces\wpp.txt -p C:\Test\Wpp\x64\Debug
```

`-flags 0x3` enables both flag bits; `-level 4` drops verbose messages. Without the `-p` TMF
path, `tracefmt` prints the message GUID and raw arguments instead of the format string.

## Comparison

| Mechanism | Setup cost | Runtime overhead | Survives release build | Structured data | Tooling | Filtering |
| --- | --- | --- | --- | --- | --- | --- |
| `DbgPrint` | None | High: global lock, text formatting, debugger handoff per line | Yes (`KdPrint` is compiled out) | No, plain text | Debugger, DebugView | Component mask + level bit |
| `DbgPrintEx` | One constant | High, as above | Yes | No | Debugger, DebugView | Component mask + level bit, per call |
| `NT_ASSERT` | None | Zero in release; a break when it fires | No, compiled out | No | Debugger, dump analysis | None |
| WPP | `trace.h`, project property, TMF files | Low: enabled check, then ETW write | Yes, if WPP is enabled for that configuration | Typed arguments via TMF, still printf-shaped | tracefmt, traceview, WPA | Flags + level, per session |
| TraceLogging / ETW | Provider GUID and registration | Very low when disabled, low when enabled | Yes | Yes: named, typed, self-describing fields | logman, tracelog, tracerpt, tracefmt, WPA | Keywords + level, per session |

Approximate costs, useful for choosing rather than for benchmarks: a filtered-out `DbgPrint`
costs hundreds of nanoseconds, an emitted one costs microseconds and serialises on a global
lock; an emitted ETW event costs a few hundred nanoseconds into a per-CPU buffer; a
`TraceLoggingWrite` with no session enabled costs a handful of instructions.

## Variations and advanced use

- **Real-time ETW while the workload runs.** Add `-rt` to `tracelog -start` (or use `logman`
  with a real-time consumer) to watch events live instead of writing an `.etl`; useful when the
  interesting moment is a boot or a plug-in event that is hard to bracket.
- **Keywords as channels.** Split events with `TraceLoggingKeyword(WD_KEYWORD_IO)` and start
  separate sessions with different `-p provider flags` masks, or one session at `0xffffffff`, so
  a support scenario can collect only I/O events at low volume.
- **Provider groups.** `TraceLoggingRegisterEx` and provider traits let a tool enable a group of
  providers together; use it when a driver ships several providers and a single collection
  profile must cover all of them.
- **Ring-buffer capture for crashes.** Add `-cnf 00:05:00` to a small circular file
  (`logman create trace CrashRing -p "{7c9e6d2a-4b31-4f88-9a15-2d6b8e0c3f47}" 0xffffffff 5`
  `-o C:\traces\ring.etl -cnf 00:05:00 -ets`) so the last minutes survive a bugcheck, then decode
  the file after reboot.
- **Debug output from a distance.** `!dbgprint` in a kernel session dumps the debug buffer when
  you cannot attach DebugView to the target.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
| --- | --- | --- |
| No `DbgPrint` output at all | `DEFAULT` mask absent, so bit 3 is clear | Set `DEFAULT` to `0x8` under `Debug Print Filter` and reboot, or `ed Kd_DEFAULT_Mask 0x8` |
| `DbgPrintEx` silent while `DbgPrint` works | Mask set for the wrong component name | Set `IHVDRIVER` to `0xF` for `DPFLTR_IHVDRIVER_ID`, not `DEFAULT` |
| Garbled or truncated strings | Wrong specifier, or the 512-byte per-call limit | `%wZ` for `PUNICODE_STRING`, `%ws` for `WCHAR*`, `%s`/`%hs` for `char*`; split long lines |
| Text appears only while a debugger is attached | DebugView not elevated, kernel capture off, or the mask only set in the debugger | Run DebugView as administrator, Ctrl+K, and set the registry value |
| Bugcheck `0x50` inside a print call | Format string or `%s` target in paged memory at `DISPATCH_LEVEL` | Keep strings and buffers non-paged, or lower the IRQL before printing |
| Assertion fires in debug, nothing in release | Expected: `NT_ASSERT` and `ASSERT` vanish in free builds | Use a status code or an ETW error event for release-visible checks |
| Assertion never fires although the condition is false | Expression has side effects and is compiled out in release | Evaluate the call first, then assert on the stored result |
| ETW capture is empty | Provider never registered, session not started, or level/keyword filtered out | `logman query providers "{GUID}"`; start with level 5 and `0xffffffff` |
| `logman` fails with `0x80070005` | Prompt is not elevated | Rerun as administrator, or join `Performance Log Users` |
| `logman create trace <name> -ets` fails with `0x800700B7` | A session with that name already exists | `logman stop <name> -ets`, or `tracelog -stop <name>` |
| `logman stop` reports the session does not exist | It was created without `-ets` and lives in the registry | `logman stop <name>` without `-ets` |
| `.etl` decodes to GUIDs and hex | WPP TMF files missing from the search path | `tracefmt <file>.etl -p <build output dir>`; TraceLogging needs no TMF |
| Driver bugchecks or hangs on unload while tracing | Provider not unregistered before the image disappears | `TraceLoggingUnregister` or `WPP_CLEANUP` before the last cleanup step |
| Events lost under load | Session buffers too small for the event rate | Raise buffer size and count (`logman update <name> -bs 1024 -nb 16 64`), or reduce the event rate with keywords |
| Timestamps look wrong across CPUs | Comparing raw `QPC` values without the session clock resolution | Decode with `tracerpt`/`tracefmt`, which normalise timestamps |

## Safety, IRQL, and cleanup checklist

- **`DbgPrint` is callable at almost any IRQL**, but every pointer it dereferences must be valid
  at that IRQL. A format string or `%s` target in paged memory at `DISPATCH_LEVEL` is a `0x50`.
- **Keep a fixed prefix** (`vDbgPrintExWithPrefix` or a literal) so a capture can be filtered and
  so lines from different drivers do not blend.
- **Never put side effects inside `NT_ASSERT`.** The whole expression disappears in release.
- **Register before writing, unregister before unloading.** A provider that outlives the image
  is a callback into freed memory.
- **Drain in-flight events before `WPP_CLEANUP` and `TraceLoggingUnregister`**: stop accepting
  new requests, wait for outstanding ones, then unregister, then delete the device.
- **Keep event payloads small and non-paged.** Binary blobs of a few hundred bytes are fine;
  whole buffers are not, and the ETW buffers are finite.
- **Do not trace secrets.** ETW files are readable by administrators and often copied off the
  machine; paths, keys, and user data belong in a redacted form.
- **Leave a session stopped and deleted.** A forgotten session keeps consuming buffers and can
  prevent the next capture from starting with the same name.

## Verification

1. `reg query "HKLM\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter"` shows
   `DEFAULT` and `IHVDRIVER`, and `DbgPrint` text appears in DebugView after a reboot.
2. With `IHVDRIVER` set to `0x1`, only error-level lines appear; with `0x8`, information-level
   lines appear too, and every line starts with the `WdTrace: ` prefix.
3. `logman query providers "{7c9e6d2a-4b31-4f88-9a15-2d6b8e0c3f47}"` lists the provider after
   the driver loads, and stops listing it after unload.
4. A capture at level 5 contains both `DriverEntry` and `Write`; a capture at level 4 omits any
   `VERBOSE` event you add; a session started with keyword `0x1` contains `Write` but not
   `DriverEntry`.
5. `tracerpt` output shows the named fields (`RequestedLength`, `CopiedLength`, `WriteCount`,
   `Payload`, `Status`) with correct values for a known payload length.
6. `tracefmt` with the TMF path decodes WPP messages with substituted arguments; without the
   path it degrades to GUIDs and raw values, which confirms the TMF is what was missing.
7. A release build still emits the TraceLogging events, and `NT_ASSERT` is confirmed absent by
   checking that a deliberately false assertion does nothing in the release binary.
