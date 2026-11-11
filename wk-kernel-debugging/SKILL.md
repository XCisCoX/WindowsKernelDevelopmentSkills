---
name: wk-kernel-debugging
description: Use when you must stop a running Windows kernel to inspect a driver: configure a VM target for serial-pipe or kdnet network debugging with bcdedit, disable Secure Boot, attach WinDbg or kd, break before load with sxe ld, then read device objects, stacks, pool and IRPs with !drvobj, !devobj, !irp, !irpfind and dt nt!_IRP while stepping a write dispatch.
---

# Full kernel debugging: freeze the machine, then look

A kernel debugger is not an in-process tool. It is a second computer that takes ownership of
the target's processors over a wire, stops all of them on command, and lets you read and write
kernel memory while the operating system is not running. That single fact explains every rule
in this skill: while you are broken in, no thread runs, no timer fires, no DPC executes, and
the clock the guest sees stands still.

The one rule that prevents most disasters: **treat every break as a global stop and snapshot
the VM first**. A target that holds the only copy of a test result, a mounted VHDX, or an
unfinished install can be left unbootable by a detach at the wrong moment. Snapshot, debug,
revert when the session goes wrong.

This skill covers the two ways to get that control — local kernel debugging on a single
machine (read-mostly, cannot break) and full kernel debugging across serial, network, or USB —
and then the workflow for using it on a real driver: break before the driver loads, find its
objects, set a breakpoint in a dispatch routine, and read the IRP it was handed.

## When to use

- A driver loads but a dispatch routine misbehaves and the only evidence is the arguments it
  received. You need the actual `IRP`, its stack location, and its buffer.
- A driver bugchecks only on the target machine and you need the state at the moment of the
  fault, not a dump written after the fact.
- You must stop execution before `DriverEntry` runs, or intercept the first `IRP_MJ_WRITE`.
- You need to inspect kernel objects that have no user-mode view: `DEVICE_OBJECT`,
  `DRIVER_OBJECT`, `FILE_OBJECT`, pending `IRP`s, pool allocations by tag.
- You are validating that unload released everything, or that a VM "set up for debugging"
  actually connects.

Do not use this skill for post-mortem analysis of a `.dmp` file (load `wk-crash-dump-analysis`),
for user-mode debugging technique (load `wk-windbg-fundamentals`), or for non-invasive logging
that keeps the target running (load `wk-kernel-tracing`).

## Prerequisites and related skills

- **Two machines.** A host running WinDbg (or `kd.exe`) and a target. The target should be a
  VM with a snapshot; physical-host debugging works but leaves no undo.
- **Debugging Tools for Windows.** Current WinDbg from the Store, or `kd.exe`/`windbg.exe`
  from the Windows SDK `Debuggers\x64` folder. Version must match the target architecture
  (x64 debugger for x64 target, ARM64 debugger for ARM64 target).
- **Symbols and build.** `_NT_SYMBOL_PATH=srv*C:\symbols*https://msdl.microsoft.com/download/symbols`
  or `.symfix`. Copy the driver's `.pdb` next to the `.sys`, and ship a Debug `/Od` build:
  release builds reorder lines and inline functions, so stepping becomes guesswork.
- **Administrator** on both machines; the target must be able to reboot.
- **Secure Boot off** on the target, always. There is no supported workaround for full kernel
  debugging with Secure Boot enabled, and test signing has the same requirement.
- **Driver already installed but not started** on the target (`sc create`, `StartType` demand),
  so that a breakpoint can be armed before the image is mapped.

Sibling skills: `wk-windbg-fundamentals` for debugger commands and symbol mechanics;
`wk-crash-dump-analysis` for `!analyze -v` and stop codes; `wk-kernel-tracing` when the target
must keep running; `wk-irp-basics` for the structure being inspected here;
`wk-driver-build-deploy` for getting the `.sys` and `.pdb` onto the VM.

## Mental model

Two executables, one protocol. The target runs a tiny transport stub (`kdnet` for network,
`kdserial` for a serial port) that owns a dedicated channel and understands a small packet
protocol: read memory, write memory, read register, continue, break. The host runs the debug
engine (`dbgeng.dll`) plus the symbol handler. Everything you type is executed on the host
against memory fetched over that channel; the target CPU is halted while the engine answers.

```
   HOST                                        TARGET (VM)
 +------------------------------+            +-----------------------------+
 | WinDbg UI / kd.exe           |  transport | ntoskrnl.exe + your driver  |
 |  dbgeng.dll  (command engine)|<==========>| kdstub: kdnet / kdserial    |
 |  symsrv      (PDBs)          |  COM pipe  | all CPUs halted on break    |
 +------------------------------+  net / USB +-----------------------------+
        |                                            |
        |  bp, dt, !irp, g                           |  memory read/write
        +--------------------------------------------+
```

- **All processors stop.** The processor index in the prompt (`0: kd>`, `1: kd>`) is the CPU
  that happened to take the break. `~` lists processors; `~1s` switches. A spinlock held by a
  frozen processor is still held, so stepping through code that waits on it hangs the session.
- **Time stops, and the debugger is the only writer.** Watchdog timers and network keepalives
  freeze; memory edits via `eb`/`ed`/`ew` and register edits via `r` are real writes to the
  halted target, not journalled and not undone by `g`.

Transport choices, in order of preference for driver work:

```
 transport   target side                     host side
 ---------   -----------------------------   -----------------------------------
 net (kdnet) bcdedit /dbgsettings net        windbg -k net:port=50000,key=1.2.3.4
 serial pipe bcdedit /dbgsettings serial     windbg -k com:pipe,port=\\.\pipe\debug
 USB 3       bcdedit /dbgsettings usb        windbg -k usb:targetname=WDUSB
 local       bcdedit /dbgsettings local      windbg -kl   (no break, read-mostly)
```

## Key APIs and structures

### Configuration commands (target, elevated)

| Command | Effect | Notes |
| --- | --- | --- |
| `bcdedit /debug on` | Turns on kernel debugging for the current boot entry | Equivalent to `bcdedit /set {current} debug on` |
| `bcdedit /dbgsettings local` | Selects local debugging | No transport, no host link |
| `bcdedit /dbgsettings serial debugport:1 baudrate:115200` | Selects COM1 at 115200 | `debugport` is the guest COM number, not the host pipe |
| `bcdedit /dbgsettings net hostip:192.168.1.50 port:50000 key:1.2.3.4` | Selects network debugging | Win8+; key is four dot-separated groups, each 1-8 chars |
| `bcdedit /dbgsettings usb targetname:WDUSB` | Selects USB 3 debugging | Host must have a USB 3 controller free |
| `bcdedit /dbgsettings` | Prints the current block | Check `debugtype`, `hostip`, `port`, `key` |
| `bcdedit /enum {current}` | Shows `debug`, `testsigning`, `debugtype` | Use quotes in PowerShell: `"{current}"` |
| `bcdedit /set {dbgsettings} busparams 0.20.0` | Pins the debug NIC | Needed when auto-detect picks the wrong NIC |
| `bcdedit /set testsigning on` | Allows test-signed drivers | **Requires Secure Boot off; error 577 otherwise** |
| `bcdedit /debug off` | Emergency recovery | Run from WinRE if the target no longer boots |

### Host attach commands

| Command | Transport |
| --- | --- |
| `windbg -k net:port=50000,key=1.2.3.4` | Network |
| `windbg -k com:pipe,port=\\.\pipe\debug,resets=0,reconnect` | Named pipe from a VM serial port |
| `windbg -k com:port=COM1,baud=115200` | Physical serial |
| `windbg -k usb:targetname=WDUSB` | USB 3 |
| `windbg -kl` | Local kernel debugging |
| `windbg -k net:port=50000,key=1.2.3.4 -b` | Break at the first instruction of boot |
| `kd -k net:port=50000,key=1.2.3.4` | Console-only engine, same protocol |

`resets=0` stops the debugger from pulsing the target's reset line on connect; `reconnect`
keeps retrying instead of exiting when the target is not yet listening.

### Commands that describe kernel objects

| Command | Answers |
| --- | --- |
| `lm m wdbuff` / `lmvm wdbuff` | Base, size, PDB path, timestamp, whether symbols loaded |
| `!drvobj wdbuff 2` | `DRIVER_OBJECT`, `DriverEntry`, `DriverUnload`, full dispatch table, device list |
| `!devobj <devobj>` | Flags, ref count, `StackSize`, device extension pointer |
| `!devstack <devobj>` | Attached devices above and below in the stack |
| `!object \Device\WdBuff` | Object header, pointer count, handle count, name |
| `!fileobj <fileobj>` | The `FILE_OBJECT` behind a handle, its name and flags |
| `!irp <irp> 1` | Stack trace of the IRP, current stack location address, per-location args |
| `!irpfind` | Every IRP the I/O manager still owns, with device object and thread |
| `!pool <addr>` | Pool allocation containing an address, plus its tag and state |
| `!poolfind WdBf` | All allocations carrying a tag |
| `dt nt!_IRP <addr>` | IRP fields: `AssociatedIrp`, `IoStatus`, `RequestorMode`, `UserBuffer` |
| `dt nt!_IO_STACK_LOCATION <addr>` | `MajorFunction`, `Parameters` union, `DeviceObject`, `FileObject` |
| `!process 0 0 wdbclient.exe` | Process address, `Cid`, PEB, threads |
| `!thread` | Current thread, `Cid`, stack bounds, wait state |
| `!stacks 2` | One line per thread with its current stack |
| `!analyze -v` | Post-bugcheck analysis with the faulting module and stack |

### Breakpoint and stepping commands

| Command | Use |
| --- | --- |
| `bu wdbuff!WdbWrite` | Unresolved breakpoint; re-evaluated on every module load |
| `bp wdbuff!WdbWrite` | Resolved breakpoint, needs symbols already loaded |
| `bp /p <process> wdbuff!WdbWrite` | Fires only when that process is current |
| `ba w 8 <addr>` | Hardware write watchpoint, maximum 4 per processor |
| `sxe ld:wdbuff` | Break when the module is mapped, before `DriverEntry` |
| `sxe -c "commands" ld:wdbuff` | Run commands automatically at that break, then continue |
| `bl` / `bc *` / `bd <n>` / `be <n>` | List, clear all, disable, enable |
| `g`, `p`, `t`, `pc`, `gu` | Go, step over, step into, step to next call, run to return |
| `k`, `kb`, `kv` | Stack; `kb` also prints the first three arguments |

## Step-by-step

1. **Snapshot the target VM.** `Checkpoint-VM -Name WDVM -SnapshotName pre-kd` on Hyper-V,
   `VBoxManage snapshot WDVM take pre-kd` on VirtualBox. *Check:* the snapshot exists and the
   VM is off before any firmware change.

2. **Turn Secure Boot off.** `Set-VMFirmware -VMName WDVM -EnableSecureBoot Off`, or the
   equivalent firmware setting on hardware. *Check:* `Confirm-SecureBootUEFI` returns `False`.

3. **Pick the transport.** Network for Windows 8 or later targets, serial pipe otherwise.
   Record the host address, port, and key before touching the guest.

4. **Configure the guest boot entry**, elevated:

   ```
   bcdedit /debug on
   bcdedit /dbgsettings serial debugport:1 baudrate:115200
   bcdedit /dbgsettings
   ```

   Network targets replace the middle line with one command:

   ```
   bcdedit /dbgsettings net hostip:192.168.1.50 port:50000 key:1.2.3.4
   ```

   *Check:* the printed block matches the host command you will type, character for character;
   this mismatch is the most common reason a session never opens.

5. **Wire the transport** as the matching row of the setup matrix below says. *Check:* the pipe
   exists, no other process owns it, and `Get-VMComPort WDVM` shows the mapping.

6. **Open the host firewall for network debugging:**

   ```
   New-NetFirewallRule -DisplayName "KDNET 50000" -Direction Inbound `
     -Protocol UDP -LocalPort 50000 -Action Allow
   ```

   *Check:* `Get-NetFirewallRule -DisplayName "KDNET 50000"` reports `Enabled True`.

7. **Boot the guest, then attach on the host, elevated:**

   ```
   windbg -k com:pipe,port=\\.\pipe\debug,resets=0,reconnect
   windbg -k net:port=50000,key=1.2.3.4
   ```

   *Check:* `Kernel Debugger connection established` and a `0: kd>` prompt. `Waiting to
   reconnect` means no packets are arriving; go to the failure table before changing the guest.

8. **Confirm symbols.** `.symfix`, `.reload`, `lm m wdbuff`. *Check:* the module line says
   `(private pdb symbols)` and shows your PDB path, not `(deferred)` or `(no symbols)`.

9. **Break before the driver loads.** With the driver installed but stopped:

   ```
   0: kd> bu wdbuff!WdbWrite
   0: kd> g
   ```

   In the guest run `sc start wdbuff`. *Check:* `bl` shows the breakpoint resolving at load.
   To arm commands at load instead, use
   `sxe -c ".reload /f wdbuff.sys; bp wdbuff!WdbWrite; g" ld:wdbuff`.

10. **Map the objects.** `!drvobj wdbuff 2`, then `!devobj <devobj>` and `!devstack <devobj>`.
    *Check:* `IRP_MJ_WRITE` points into `wdbuff`, not to `nt!IopInvalidDeviceRequest`, and the
    flags word contains `0x4` (`DO_BUFFERED_IO`) or `0x10` (`DO_DIRECT_IO`) as you set it.

11. **Trigger the path.** Run the client in the guest. *Check:* the breakpoint hits inside your
    module and the prompt shows the processor that took it.

12. **Read the request.** On x64 the dispatch routine receives `PDEVICE_OBJECT` in `RCX` and
    `PIRP` in `RDX`: run `r rcx,rdx`, `!irp @rdx 1`, `dt nt!_IRP @rdx`, then
    `dt nt!_IO_STACK_LOCATION <current stack location>` and `dx @$sp->Parameters.Write.Length`.
    *Check:* the length equals the client's byte count and `MajorFunction` is `0x4`.

13. **Read the buffer.** Buffered I/O keeps it at `Irp->AssociatedIrp.SystemBuffer`, direct I/O
    in the pages described by `Irp->MdlAddress`, and neither I/O leaves a raw user pointer in
    `Irp->UserBuffer` that must not be touched at `DISPATCH_LEVEL`. Dump with
    `db <addr> L<len>`. *Check:* the bytes match what the client wrote.

14. **Step to completion.** `pc` to the `IoCompleteRequest` call, `p` over it, then
    `dt nt!_IRP @rdx IoStatus`. *Check:* `Status` is `0` and `Information` equals the length
    you reported.

15. **Detach cleanly.** `bc *`, then `.detach`. *Check:* the guest resumes, the client prints
    its result, and guest time advances again.

## Complete example

A complete driver, its client, and a session. The driver keeps the last write in a tagged
non-paged pool buffer, so pool inspection has something real to show.

```c
/* WdBuff.c - write target for kernel debugging; ExAllocatePool2 needs Win10 2004+ */
#include <ntddk.h>

#define WB_TAG 'fBdW'          /* displays as WdBf in !pool */
#define WB_MAX_SNAPSHOT 0x100

typedef struct _WB_DEVICE_EXTENSION {
    PUCHAR      Snapshot;
    ULONG       Capacity;
    ULONG       LastLength;
    ULONG       WriteCount;
    KSPIN_LOCK  Lock;
} WB_DEVICE_EXTENSION, *PWB_DEVICE_EXTENSION;

static VOID WdbUnload(PDRIVER_OBJECT DriverObject);
static NTSTATUS WdbCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp);
static NTSTATUS WdbWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp);

NTSTATUS DriverEntry(PDRIVER_OBJECT DriverObject, PUNICODE_STRING RegistryPath)
{
    UNICODE_STRING deviceName = RTL_CONSTANT_STRING(L"\\Device\\WdBuff");
    UNICODE_STRING symLink    = RTL_CONSTANT_STRING(L"\\??\\WdBuff");
    PDEVICE_OBJECT deviceObject = NULL;
    PWB_DEVICE_EXTENSION ext = NULL;
    NTSTATUS status;

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "WdBuff: DriverEntry %wZ\n", RegistryPath);

    status = IoCreateDevice(DriverObject, sizeof(WB_DEVICE_EXTENSION), &deviceName,
                            FILE_DEVICE_UNKNOWN, 0, FALSE, &deviceObject);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PWB_DEVICE_EXTENSION)deviceObject->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));
    ext->Capacity = WB_MAX_SNAPSHOT;
    ext->Snapshot = (PUCHAR)ExAllocatePool2(POOL_FLAG_NON_PAGED, WB_MAX_SNAPSHOT, WB_TAG);
    if (ext->Snapshot == NULL) {
        IoDeleteDevice(deviceObject);
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    KeInitializeSpinLock(&ext->Lock);

    deviceObject->Flags |= DO_BUFFERED_IO;
    deviceObject->Flags &= ~DO_DEVICE_INITIALIZING;

    status = IoCreateSymbolicLink(&symLink, &deviceName);
    if (!NT_SUCCESS(status)) {
        ExFreePoolWithTag(ext->Snapshot, WB_TAG);
        IoDeleteDevice(deviceObject);
        return status;
    }

    DriverObject->DriverUnload = WdbUnload;
    DriverObject->MajorFunction[IRP_MJ_CREATE] = WdbCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE]  = WdbCreateClose;
    DriverObject->MajorFunction[IRP_MJ_WRITE]  = WdbWrite;
    return STATUS_SUCCESS;
}

static VOID WdbUnload(PDRIVER_OBJECT DriverObject)
{
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(L"\\??\\WdBuff");
    PDEVICE_OBJECT deviceObject = DriverObject->DeviceObject;

    IoDeleteSymbolicLink(&symLink);
    if (deviceObject != NULL) {
        PWB_DEVICE_EXTENSION ext = (PWB_DEVICE_EXTENSION)deviceObject->DeviceExtension;
        if (ext->Snapshot != NULL) {
            ExFreePoolWithTag(ext->Snapshot, WB_TAG);
            ext->Snapshot = NULL;
        }
        IoDeleteDevice(deviceObject);
    }
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "WdBuff: unloaded\n");
}

static NTSTATUS WdbCreateClose(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS WdbWrite(PDEVICE_OBJECT DeviceObject, PIRP Irp)
{
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PWB_DEVICE_EXTENSION ext = (PWB_DEVICE_EXTENSION)DeviceObject->DeviceExtension;
    ULONG length = sp->Parameters.Write.Length;
    ULONG copy = 0;
    NTSTATUS status = STATUS_SUCCESS;
    KIRQL oldIrql;

    if (length == 0 || Irp->AssociatedIrp.SystemBuffer == NULL) {
        status = STATUS_INVALID_PARAMETER;
        goto done;
    }

    copy = min(length, ext->Capacity);
    RtlCopyMemory(ext->Snapshot, Irp->AssociatedIrp.SystemBuffer, copy);

    KeAcquireSpinLock(&ext->Lock, &oldIrql);
    ext->LastLength = copy;
    ext->WriteCount++;
    KeReleaseSpinLock(&ext->Lock, oldIrql);

done:
    Irp->IoStatus.Status = status;
    Irp->IoStatus.Information = (status == STATUS_SUCCESS) ? length : 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

```c
/* wdbclient.c - console write client; cl /W4 wdbclient.c */
#include <windows.h>
#include <stdio.h>
#include <string.h>

int main(int argc, char **argv)
{
    const char *text = (argc > 1) ? argv[1] : "hello kernel";
    DWORD length = (DWORD)strlen(text), written = 0;
    HANDLE h = CreateFileW(L"\\\\.\\WdBuff", GENERIC_WRITE, 0, NULL,
                           OPEN_EXISTING, 0, NULL);
    BOOL ok;
    if (h == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }
    ok = WriteFile(h, text, length, &written, NULL);
    printf("ok=%d written=%lu\n", ok, written);
    CloseHandle(h);
    return ok ? 0 : 2;
}
```

Deploy on the target without starting it, keeping the PDB beside the binary:

```
sc create wdbuff type= kernel binPath= C:\Test\wdbuff.sys
copy wdbuff.pdb C:\Test\
copy wdbclient.exe C:\Test\
```

### Session transcript

Every address, PDB path, and processor index below is illustrative of one session; the shapes
and field names are the contract, the numbers are not. Output is trimmed where it repeats.

```
0: kd> sxe ld:wdbuff
0: kd> g
Breakpoint 0 hit
nt!DebugService2+0x5:
fffff801`3c1b0f10 cc              int     3
```

The image is mapped and its symbols are loaded, but `DriverEntry` has not run yet. This is the
most reliable moment to arm a breakpoint before any of the driver code executes.

```
0: kd> lm m wdbuff
start             end                 module name
fffff801`4a2b0000 fffff801`4a2b9000   wdbuff     (private pdb symbols)
0: kd> bp wdbuff!WdbWrite
0: kd> g
```

In the guest: `sc start wdbuff`. The driver creates its device now; the dispatch breakpoint stays
dormant until a write arrives.

```
0: kd> !drvobj wdbuff 2
Driver object (ffffa70b`12345678) is for:
 \Driver\WdBuff
DriverEntry:   fffff801`4a2b3000	wdbuff!GsDriverEntry
DriverStartIo: 00000000`00000000
DriverUnload:  fffff801`4a2b3100	wdbuff!WdbUnload
Dispatch routines:
[00] IRP_MJ_CREATE                      fffff801`4a2b3200	wdbuff!WdbCreateClose
[02] IRP_MJ_CLOSE                       fffff801`4a2b3200	wdbuff!WdbCreateClose
[04] IRP_MJ_WRITE                       fffff801`4a2b3300	wdbuff!WdbWrite
Device object list:
ffffa70b`23456780
```

`[04]` is `IRP_MJ_WRITE`; the pointer into `wdbuff` proves the dispatch table is wired. Now
the device and its stack:

```
0: kd> !devobj ffffa70b`23456780
Device object (ffffa70b23456780) is for:
 WdBuff \Driver\WdBuff DriverObject ffffa70b12345678
Current Irp 00000000 RefCount 1 Type 00000022 Flags 00000004
DeviceExtension: ffffa70b`234568f0
Device queue is not busy.
```

`Flags 00000004` is `DO_BUFFERED_IO`, and `Type 00000022` is `FILE_DEVICE_UNKNOWN`. The
extension address is what you inspect next; `!devstack` on the same address shows a single
device with nothing attached above or below it, which is what a plain WDM driver looks like.

Confirm the breakpoint and release the target:

```
0: kd> bl
     1 e Disable Clear  fffff801`4a2b3300  [c:\src\wdbuff.c @ 88]  0001 (0001)
0: kd> g
```

In the guest: `C:\Test\wdbclient.exe "hello kernel"`. The breakpoint fires on whichever
processor handled the write:

```
1: kd> k
 # Child-SP          RetAddr           Call Site
00 ffffbe88`b3f4f6a8 fffff801`4a2b32c0 wdbuff!WdbWrite [c:\src\wdbuff.c @ 88]
01 ffffbe88`b3f4f6d8 fffff801`3c1f0a10 nt!IofCallDriver+0x60
02 ffffbe88`b3f4f700 fffff801`3c2b8e22 nt!IopSynchronousServiceTail+0x2a2
03 ffffbe88`b3f4f7a0 fffff801`3c1b0f10 nt!NtWriteFile+0x6d2
```

Arguments, then the IRP itself. `RCX` is the device object, `RDX` is the IRP:

```
1: kd> r rcx,rdx
rcx=ffffa70b`23456780 rdx=ffffa70b`34567890
1: kd> !irp @rdx 1
Irp is active with 1 stacks 1 is current (= 0xffffa70b`345679b8)
 No Mdl: System buffer ffffa70b`3456a000: Thread ffffa70b`2c1a2080:  Irp stack trace.
     cmd  flg cl Device   File     Completion-Context
 [4, 0]   0  0 ffffa70b`23456780 00000000 00000000-00000000
           \Driver\WdBuff
            Args: 0000000c 00000000 00000000 00000000
```

`Args: 0000000c` is the 12-byte length. `No Mdl` and a `System buffer` confirms the buffered
path. Now the fields that matter:

```
1: kd> dt nt!_IRP @rdx
   +0x000 Type             : 0n6
   +0x008 MdlAddress       : (null)
   +0x018 AssociatedIrp
      +0x000 SystemBuffer  : 0xffffa70b`3456a000 Void
   +0x030 IoStatus
      +0x000 Status         : 0n0
      +0x008 Information    : 0n0
   +0x040 RequestorMode    : 1 ( UserMode )
   +0x041 PendingReturned  : 0 ''
   +0x070 UserBuffer       : 0x000000d4`3f9ff6a0 Void
```

`RequestorMode 1` means the request came from user mode, so any pointer the driver reads must
be probed or, better, already captured by the I/O manager into `SystemBuffer`. `UserBuffer`
is the raw user address; it is not valid at raised IRQL. The stack location:

```
1: kd> dt nt!_IO_STACK_LOCATION 0xffffa70b`345679b8
   +0x000 MajorFunction    : 0x4 ''
   +0x001 MinorFunction    : 0 ''
   +0x008 Parameters       : <unnamed-tag>
   +0x020 DeviceObject     : 0xffffa70b`23456780 _DEVICE_OBJECT
   +0x028 FileObject       : 0xffffa70b`34567a10 _FILE_OBJECT
   +0x030 CompletionRoutine : (null)
   +0x038 Context          : (null)
```

`Parameters` is a union, so ask for the member that belongs to this major function:

```
1: kd> dx @$sp = (nt!_IO_STACK_LOCATION*)0xffffa70b`345679b8
1: kd> dx @$sp->Parameters.Write.Length
@$sp->Parameters.Write.Length : 0xc
1: kd> dx @$sp->Parameters.Write.ByteOffset
@$sp->Parameters.Write.ByteOffset : 0
```

Read the caller's bytes from the system buffer, and identify the file object behind the handle:

```
1: kd> db 0xffffa70b`3456a000 L0xc
ffffa70b`3456a000  68 65 6c 6c 6f 20 6b 65-72 6e 65 6c              hello kernel
1: kd> !fileobj 0xffffa70b`34567a10
File object is for: \Device\WdBuff
FileName: \??\WdBuff
1: kd> !object \Device\WdBuff
Object: ffffa70b`23456780  Type: (ffffa70b`1a2b3c40) Device
    HandleCount: 1  PointerCount: 3  Name: WdBuff
```

Step to the copy and confirm the destination is the tagged pool buffer:

```
1: kd> p
wdbuff!WdbWrite+0x38:
fffff801`4a2b20a8 488b4c2440      mov     rcx,qword ptr [rsp+40h]
1: kd> p
1: kd> dt wdbuff!_WB_DEVICE_EXTENSION 0xffffa70b`234568f0
   +0x000 Snapshot         : 0xffffa70b`3456b000
   +0x008 Capacity         : 0x100
   +0x00c LastLength       : 0xc
   +0x010 WriteCount       : 1
   +0x018 Lock             : 0
1: kd> !pool 0xffffa70b`3456b000
(allocation header lines omitted)
 ffffa70b`3456b000 size:  110 previous size:    0  (Allocated) *WdBf
```

`*WdBf` is the tag: `*` marks an allocated block, and the literal `'fBdW'` in the source reads
backwards here on purpose. `!poolfind` finds every block with that tag if you suspect a leak:

```
1: kd> !poolfind WdBf
Scanning large pool allocation table for tag WdBf
Searching Nonpaged pool (ffffa70b00000000 : ffffc70b00000000) for Tag WdBf
ffffa70b`3456b000 size:  110 previous size:    0  (Allocated) WdBf
```

Now walk to completion. `pc` steps to the next call instruction, which is
`IoCompleteRequest`; step over it and read the result the I/O manager will hand back:

```
1: kd> pc
wdbuff!WdbWrite+0x71:
fffff801`4a2b20e1 ff15190a0000    call    qword ptr [wdbuff!_imp_IoCompleteRequest]
1: kd> p
1: kd> dt nt!_IRP @rdx IoStatus
   +0x030 IoStatus
      +0x000 Status         : 0n0
      +0x008 Information    : 0n12
```

`Information 0n12` matches the 12 bytes copied, which is what `WriteFile` will report in
`written`. Let it finish and detach:

```
1: kd> g
0: kd> bc *
0: kd> .detach
```

The guest prints `ok=1 written=12`. If a later check shows the driver
still holding references, `!devobj` reports a non-zero `RefCount` after unload was attempted.

## Setup matrix

| Transport | Target configuration | Host command | VM or hardware wiring | Firewall and gotchas |
| --- | --- | --- | --- | --- |
| Local | `bcdedit /debug on` then `bcdedit /dbgsettings local` | `windbg -kl` (elevated) | Same machine, no wire | No breakpoints, no stepping; device must support it |
| Serial, physical | `bcdedit /dbgsettings serial debugport:1 baudrate:115200` | `windbg -k com:port=COM1,baud=115200` | Null-modem cable, host COM port free | `debugport` is the guest-visible COM number; 115200 is the maximum reliable rate on most UARTs |
| Serial pipe, Hyper-V Gen2 | `bcdedit /dbgsettings serial debugport:1 baudrate:115200` | `windbg -k com:pipe,port=\\.\pipe\debug,resets=0,reconnect` | `Set-VMComPort WDVM -Number 1 -Path "\\.\pipe\debug"` with the VM off | Pipe name must be unique; `Get-VMComPort` proves the mapping; no UI exists for Gen2 |
| Serial pipe, VMware | Same as above | Same as above | Serial port, "Use named pipe", this end is the server, name `\\.\pipe\debug`, other end is the VM | `.vmx` equivalent: `serial0.present`, `serial0.fileType = "pipe"`, `serial0.pipe.endPoint = "server"` |
| Serial pipe, VirtualBox | Same as above | Same as above | `VBoxManage modifyvm WDVM --uart1 0x3F8 4 --uartmode1 server "\\.\pipe\debug"` | `--uartmode1 disconnected` releases a stale pipe |
| Network (kdnet) | `bcdedit /dbgsettings net hostip:192.168.1.50 port:50000 key:1.2.3.4` | `windbg -k net:port=50000,key=1.2.3.4` | External switch or bridged NIC so the host address is routable | Inbound UDP 50000 on the host; `busparams` if the wrong NIC is chosen; Win8+ only |
| USB 3 | `bcdedit /dbgsettings usb targetname:WDUSB` | `windbg -k usb:targetname=WDUSB` | Dedicated USB 3 host controller and cable | Debugger owns the controller; no hubs; Win8+ with xHCI |

Two supporting commands worth memorising: `kdnet.exe 192.168.1.50 50000` on the target writes
the network settings and prints a generated key, and `bcdedit /dbgsettings` reprints whatever
is currently configured.

## Local kernel debugging and its limits

`bcdedit /debug on` plus `bcdedit /dbgsettings local`, one reboot, then an elevated `windbg -kl`.
It attaches to the kernel of the machine it runs on: good for a snapshot of object state,
useless for controlling execution. **No breakpoints, no stepping, no memory writes, no user-mode
stacks**; the machine keeps running and its state moves under you. It occupies the local debug
channel, so no full-debug session can be active at the same time.

## Variations and advanced use

- **Break at boot.** Add `-b` (`windbg -k net:port=50000,key=1.2.3.4 -b`) to stop at the first
  instruction; with `bcdedit /set {default} bootdebug on` you can catch a driver that loads
  before any user-mode tool could arm a breakpoint.
- **Watch a field instead of a line.** `ba w 8 ffffa70b`234568f0+0x0c` stops on any write to
  `LastLength`. Four hardware watchpoints per processor, consumed by the debugger.
- **Filter by process.** `!process 0 0 wdbclient.exe` yields the process address, then
  `bp /p ffffa70b`2c1a2080 wdbuff!WdbWrite` fires only for that client.
- **Capture and verify.** `.dump /f C:\dumps\kd.dmp` writes a full dump from a live session, and
  `verifier /standard /driver wdbuff.sys` with `!verifier 3 wdbuff` stops the machine at a
  violation with a precise stack instead of letting the corruption surface later.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
| --- | --- | --- |
| `Waiting to reconnect` forever, or `Opened \\.\pipe\debug` then silence | `debugport` differs from the VM COM mapping, pipe name differs, another debugger owns the pipe, or the guest booted without debugging | `bcdedit /dbgsettings` in the guest; `Get-VMComPort WDVM` on the host; kill stale `windbg.exe`; `bcdedit /enum {current}` must show `debug Yes` |
| kdnet never connects | Wrong `hostip`, blocked UDP port, NAT'd switch, wrong key | `New-NetFirewallRule` for UDP 50000; `bcdedit /dbgsettings`; run `kdnet.exe` on the guest |
| `Could not connect to target` with `0x80070005` | Debugger not elevated | Relaunch WinDbg as administrator |
| Breakpoint listed but never resolves | Module name typo, or the driver never loaded | `lm m wdbuff`; `sc query wdbuff`; `sxe ld:wdbuff` |
| `*** ERROR: Module load completed but symbols could not be loaded` | PDB missing, stale, or not on the symbol path | Copy the PDB beside the `.sys`; `.reload /f wdbuff.sys`; `!sym noisy` |
| Breakpoint hits but no source line, `bl` shows no file | Source path absent from the PDB or the tree moved | `.srcpath+ C:\src`; `.lines`; rebuild Debug with `/Od` |
| Stepping hangs and never returns | Stepped into a path waiting on a lock held by a frozen processor, or into an ISR | `g` to resume, then `ba` on the data instead of single-stepping |
| `0x50` while dumping memory | Address paged out or invalid for the current context | `.process /r /p <proc>` then retry; `!pte <addr>` |
| `0x9F` on stop or unload | Driver unloaded with a pending IRP or power request | `!irpfind`; `!devobj` ref count; drain work in unload first |
| Target will not boot after enabling debug | Debug port absent in firmware, or serial conflict | Boot WinRE, `bcdedit /debug off`, re-plan the transport |
| Driver will not start on the target, error 577 | Test signing off, or Secure Boot still on | `bcdedit /set testsigning on`; `Confirm-SecureBootUEFI` must be `False` |
| `!devobj` shows `RefCount 0` but `!object` shows handles | Handles still open in user mode | Close the client handles; the device cannot be deleted until the last handle closes |
| `!irp` prints `Irp is not active` | The IRP completed and the pointer is stale | Re-break at entry; never cache IRP addresses across `g` |

## Safety, IRQL, and cleanup checklist

- **Snapshot before every session** and revert rather than repair when a session goes wrong.
- **Assume every other processor is halted at an arbitrary point**, possibly inside a spinlock
  or on a DPC; do not expect forward progress while broken in.
- **Clear breakpoints and detach.** `bc *` then `.detach` before walking away; a stray `bp` in
  a hot path freezes the machine later.
- **Do not edit live memory unless you can name the invariant you are breaking.** `eb` on a
  refcount or a list head turns a clean bug into machine corruption.
- **`RequestorMode` decides trust.** User-mode requests carry user pointers; never dereference
  `UserBuffer` at raised IRQL.
- **Know the buffering model before reading a buffer.** `No Mdl` plus a `System buffer` means
  buffered, a `Mdl` means direct, `UserBuffer` alone means neither and requires probing.
- **Undo the boot configuration** on any machine that is not a dedicated target:
  `bcdedit /debug off`, `bcdedit /set testsigning off`, Secure Boot back on.

## Verification

1. The host attach command prints `Kernel Debugger connection established` and a `0: kd>`
   prompt within about 30 seconds of the guest booting.
2. `lm m wdbuff` shows `(private pdb symbols)` and the path to your `.pdb`.
3. `!drvobj wdbuff 2` lists your entry points, and `IRP_MJ_WRITE` points inside `wdbuff`.
4. `bu wdbuff!WdbWrite` resolves at load and the client's write hits it.
5. At the break, `!irp @rdx 1` shows `[4, 0]`, and `dx @$sp->Parameters.Write.Length` equals the
   client's byte count.
6. `db` of `AssociatedIrp.SystemBuffer` reproduces the client's bytes; after
   `IoCompleteRequest`, `IoStatus` shows `Status 0` and `Information 12`, and the client prints
   `ok=1 written=12`.
7. `!poolfind WdBf` shows one allocation while loaded and none after unload; `!object
   \Device\WdBuff` fails after `sc stop wdbuff`; `.detach` resumes the guest and
   `bcdedit /enum {current}` still shows the boot configuration you intended.
