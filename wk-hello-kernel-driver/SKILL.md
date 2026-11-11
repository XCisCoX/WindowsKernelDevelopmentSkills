---
name: wk-hello-kernel-driver
description: Use when proving a Windows kernel toolchain end to end on a test VM, covering a complete WDM hello driver with DriverEntry, device object, symbolic link and create/close dispatch, a user-mode client for \\\\.\\Hello, test signing, sc create and sc start, DebugView capture, !drvobj verification and clean uninstall, with expected output and a failure branch per step.
---

# Hello kernel driver: the end-to-end baseline

Before a driver does anything useful, the machine must prove it can build one, sign it, load it, let a
user-mode client open it, and unload it without leaving debris. This skill is that proof: one small
WDM driver that creates a named device, publishes a symbolic link, and answers create, cleanup and
close; one console client that opens the device and closes it; and thirteen steps with the exact
command and expected result for each. The mental model is a **ladder of gates** — build, signature,
service, dispatch, unload — where each gate is observed once and never assumed. The one rule: **do not
climb past a failed step**, because a failure carried forward becomes a bugcheck that has nothing to
do with its real cause.

## When to use

- A new host, a new test VM, or a freshly patched WDK, and the toolchain has never produced a loaded
  driver.
- A driver fails somewhere between `msbuild` and `CreateFileW` and the failure is not yet localized to
  a gate.
- A working lab is being rebuilt and needs a known-good baseline to compare against, or a minimal
  reproduction is needed for a build, signing, or service-configuration problem.
- Confirming that DebugView, test signing and `!drvobj` still work after a host change.

Do not use for: the toolchain install itself (`wk-driver-dev-environment`), the reproducibility
pipeline with an INF, catalog and driver store (`wk-driver-build-deploy`), a driver with a real IOCTL
protocol (`wk-driver-anatomy`), or attaching a kernel debugger for the first time
(`wk-kernel-debugging`).

## Prerequisites and related skills

- **Host**: Windows 10 22H2 or Windows 11 x64 with Visual Studio 2022, a matching Windows SDK and WDK,
  and the Spectre-mitigated libraries, verified by the checks in Step 1.
- **Guest**: Windows 10 or 11 x64 virtual machine, UEFI with Secure Boot **off**, local Administrator,
  and a snapshot capability (`Checkpoint-VM`, `VBoxManage snapshot`, or VMware snapshots).
- **Rights**: Administrator on the host (certificate store, `signtool`) and on the guest (`sc`,
  `bcdedit`). No second machine is needed: local kernel debugging is enough for `!drvobj`, and
  DebugView is enough for the traces.

Related skills: `wk-driver-dev-environment` owns the version matrix and the project properties this
lab assumes; `wk-driver-build-deploy` repeats the deployment with an INF, a catalog and a driver store
entry; `wk-device-objects` explains `IoCreateDevice`, `DO_DEVICE_INITIALIZING` and link security;
`wk-driver-anatomy` is the next driver to write once this baseline passes; `wk-kernel-debugging`
covers real breakpoint sessions.

## Mental model

A load is a chain of independent approvals, and each one has a distinct observable. Climb in order.

```
 rung  gate                     observable
 ----  -----------------------  --------------------------------------------------------
  1    toolchain resolves       msbuild prints "Build succeeded", 0 errors
  2    image is a driver        Hello.sys exists, dumpbin shows DriverEntry, machine 8664
  3    signature accepted       signtool verify /pa prints "Successfully verified"
  4    boot policy allows it    bcdedit shows testsigning Yes, Test Mode watermark visible
  5    service is registered    sc create prints [SC] CreateService SUCCESS
  6    loader ran DriverEntry   sc start prints STATE : 4 RUNNING, exit code 0
  7    device and link exist    WinObj shows \Device\Hello and \??\Hello; !devobj resolves
  8    dispatch works           CreateFileW succeeds, DebugView shows "create"
  9    unload is safe           sc stop prints STATE : 1 STOPPED, DebugView shows "unloaded"
 10    nothing leaked           sc query returns 1060 after delete, no device left in WinObj
```

Two details of the lifecycle matter more than the rest.

- **Create is not open, and close is not cleanup.** `IRP_MJ_CREATE` arrives when a handle opens;
  `IRP_MJ_CLEANUP` when the last user handle closes, in the closing thread's context; `IRP_MJ_CLOSE`
  when the last file-object reference drops, possibly much later and in an unrelated context.
  Per-handle state belongs to cleanup; freeing it on close is either late or a double free.
- **Unload must be reachable.** With `DriverObject->DriverUnload == NULL` the Service Control Manager
  cannot stop the driver at all: `sc stop` fails and the image stays mapped until reboot. Setting the
  unload routine is part of the contract, not a refinement.

The unload path mirrors `DriverEntry` in reverse: delete the symbolic link, delete the device object,
free anything else. The device pointer is not passed to `DriverUnload`, so it is reached through
`DriverObject->DeviceObject`.

## Key APIs and structures

| API / structure | Purpose | IRQL / context | Notes |
|---|---|---|---|
| `DriverEntry` | Entry point, called once when the image is mapped | `PASSIVE_LEVEL` | Return `STATUS_SUCCESS` or the image is unloaded with no `DriverUnload` call |
| `DriverObject->DriverUnload` | Stop callback | `PASSIVE_LEVEL` | `NULL` makes the driver unkillable |
| `IoCreateDevice` | Creates the named device object | `PASSIVE_LEVEL` | Second argument is the extension size, not a pointer |
| `IoCreateSymbolicLink` | Creates `\??\Hello` → `\Device\Hello` | `PASSIVE_LEVEL` | Takes two `PUNICODE_STRING`; failure requires `IoDeleteDevice` |
| `IoDeleteSymbolicLink` / `IoDeleteDevice` | Undo the pair | `PASSIVE_LEVEL` | Reverse order, in `DriverUnload` |
| `IRP_MJ_CREATE` handler | Accept or refuse an open | `PASSIVE_LEVEL` | Complete with `STATUS_SUCCESS`, `Information = 0` |
| `IRP_MJ_CLEANUP` / `IRP_MJ_CLOSE` handlers | Last user handle, then last file-object reference | `PASSIVE_LEVEL` | Per-handle state is released at cleanup; close can arrive much later |
| `IoGetCurrentIrpStackLocation` | The `IO_STACK_LOCATION` for this driver | any IRQL | Read `MajorFunction` and `Parameters.*` here |
| `IoCompleteRequest` | Returns the IRP to the I/O manager | `<= DISPATCH_LEVEL` | After the call the `IRP*` must not be touched |
| `DbgPrintEx(DPFLTR_IHVDRIVER_ID, level, fmt, args)` | Filtered trace | any IRQL | Levels `ERROR` 0, `WARNING` 1, `TRACE` 2, `INFO` 3 |

## Step-by-step

### Step 0 — Take a checkpoint

```powershell
Checkpoint-VM -Name DevVM -SnapshotName HelloBaseline
Get-VMSnapshot -VMName DevVM | Select-Object Name, CreationTime
```

Expected: the checkpoint appears with the current time. VirtualBox equivalent:
`VBoxManage snapshot DevVM take HelloBaseline`.

**If this step fails:** without a snapshot there is no rollback, and a driver that bugchecks at boot
turns a 30-minute lab into a rebuild. Fix the VM tooling first.

### Step 1 — Confirm the toolchain

```powershell
Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\Include" | Select-Object Name
Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\bin\10.0.26100.0\x64" -Filter *.exe |
    Where-Object Name -in 'signtool.exe','inf2cat.exe','stampinf.exe'
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * `
    -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
```

Expected: a `10.0.xxxxx.0` folder under `Include`, three executable names, and a Visual Studio path.

**If this step fails:** the SDK and WDK do not match, or the C++ workload is missing. Install in the
order SDK then WDK and re-check with `wk-driver-dev-environment`; a project created on a broken
toolchain produces errors that look like code errors.

### Step 2 — Create the project

*File → New → Project*, search `WDM`, choose **Empty WDM Driver**, name it `Hello`, target `x64`.
Then remove the generated INF: right-click `Hello.inf` under *Driver Files* → *Remove*, and delete the
file. This lab installs through the Service Control Manager, which needs no INF.

Expected: *Solution Explorer* shows `Hello` with *Driver Files* and *Source Files*, and the toolbar
shows `Debug` and `x64`.

**If this step fails:** the driver templates are not registered, so the WDK's Visual Studio extension
did not install. Close Visual Studio, rerun the WDK installer, choose *Change*, install the extension.

### Step 3 — Add the source and set the properties that matter

Add `Hello.c` (*Source Files → Add → New Item → C++ File*, renamed to `.c`) and paste the listing from
*Complete example*. Then, with *All Configurations* and *All Platforms* selected:

| Property | Value |
|---|---|
| *General → Configuration Type* | `Driver` |
| *Driver Settings → Target Platform Version* | the installed SDK, for example `10.0.26100.0` |
| *C/C++ → General → Warning Level* | `Level4 (/W4)` |
| *C/C++ → General → Treat Warnings As Errors* | `Yes (/WX)` |
| *C/C++ → Code Generation → Spectre Mitigation* | `Enabled (/Qspectre)` |
| *Driver Signing → Sign Mode* | `Off`, because Step 4 signs explicitly |
| *Driver Settings → Run Inf2Cat* | `No`, because there is no INF |

Expected: the file compiles with no unused-parameter warning, because every unused parameter is
wrapped in `UNREFERENCED_PARAMETER`.

**If this step fails:** `C4100` under `/WX` means an unused parameter is not annotated; `MSB8020`
means the v143 toolset is missing. Neither is fixed by disabling `/WX`.

### Step 4 — Build and sign

```powershell
.\build.ps1 -Configuration Debug
```

Expected:

```
Build succeeded.
    0 Warning(s)
    0 Error(s)
Built and signed: C:\dev\Hello\x64\Debug\Hello.sys
```

`build.ps1` (in *Complete example*) initializes the Visual Studio environment, runs `msbuild` with an
explicit `OutDir`, creates the test certificate on first use, signs the binary and verifies it.

**If this step fails:** `LNK2019: unresolved external symbol DriverEntry` means the entry point is
missing or renamed (a `.c` file needs no `extern "C"`). `LNK2038` on `SpectreMitigation` means the
Spectre libraries are absent; `MSB8020` means `Enter-VsDevShell` did not run inside `build.ps1`.

### Step 5 — Verify the signature by hand

```powershell
& signtool.exe verify /pa /v .\x64\Debug\Hello.sys
Get-ChildItem Cert:\LocalMachine\Root | Where-Object Subject -like '*Contoso Test*'
```

Expected: `Successfully verified`, and the certificate present in both `Root` and
`TrustedPublisher`. Those are two of the three conditions a test certificate must satisfy; the third
is the code-signing EKU, set by `-Type CodeSigningCert`.

**If this step fails:** `A certificate chain processed, but terminated in a root certificate which is
not trusted` means the `.cer` never reached `LocalMachine\Root`. `No certificates were found that
match the given criteria` means `signtool` looked in the wrong store: the certificate lives in
`LocalMachine\My`, so sign from an elevated console.

### Step 6 — Enable test signing and reboot

```powershell
bcdedit /set testsigning on
bcdedit /enum "{current}" | Select-String -Pattern 'testsigning'
Restart-Computer
```

Expected after the reboot: `testsigning Yes`, and a **Test Mode** watermark in the lower-right corner
of the desktop. The watermark, not the command output, is the proof.

**If this step fails:** `The value is protected by Secure Boot policy and cannot be modified or
deleted` means Secure Boot is on; turn it off in the VM firmware and retry. A watermark that never
appears after a successful command means the machine booted a different entry; check
`bcdedit /enum all`.

### Step 7 — Copy the driver to the guest

```powershell
# PowerShell Direct, Hyper-V host to guest, no network required
$session = New-PSSession -VMName DevVM -Credential (Get-Credential)
Copy-Item -ToSession $session .\x64\Debug\Hello.sys -Destination C:\Test\Hello.sys
Invoke-Command -Session $session -ScriptBlock { Get-Item C:\Test\Hello.sys | Select-Object Length }
```

Expected: a size of roughly 10–30 KB. A 2–3 KB file means the linker produced an empty image.

**If this step fails:** PowerShell Direct needs a Windows 10/11 guest, guest administrator
credentials, and a running VM. Fall back to a shared folder, an ISO, or SMB; never set `binPath` to a
network path, because a driver is loaded from a local file.

### Step 8 — Create and start the service

```powershell
# elevated, on the guest
sc.exe create hello type= kernel binPath= C:\Test\Hello.sys start= demand
sc.exe start hello
sc.exe query hello
```

Expected:

```
[SC] CreateService SUCCESS

SERVICE_NAME: hello
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
        WIN32_EXIT_CODE    : 0  (0x0)
```

`PID : 0` is normal: a driver is not a process.

**If this step fails:** `FAILED 87` is `sc` syntax (a space is required after `type=`, `binPath=` and
`start=`); `FAILED 577` is an unsigned or untrusted image; `FAILED 1275` is a policy block or memory
integrity; `FAILED 31` with `STATE : 1 STOPPED` means `DriverEntry` returned a failure status. Read
DebugView before changing anything.

### Step 9 — Open the device from user mode

```powershell
# on the guest, in a Visual Studio developer prompt
cl /nologo /W4 HelloClient.c
.\HelloClient.exe
```

Expected, in the client console:

```
Opened \\.\Hello (handle 000000000000012C)
Press Enter to close the handle
Handle closed
```

**If this step fails:** error 2 (`ERROR_FILE_NOT_FOUND`) means no symbolic link or a different name —
compare with `WinObj` → `\??\Hello`. Error 5 (`ERROR_ACCESS_DENIED`) means the device security
descriptor denies the caller; run the client elevated for this lab. Error 3
(`ERROR_PATH_NOT_FOUND`) means the `\\.\` prefix was lost in the source string.

### Step 10 — Read the trace in DebugView

Run `Dbgview.exe` **elevated**, press `Ctrl+K` (*Capture → Capture Kernel*), disable *Capture Win32*,
then stop and start the driver and open the client.

Expected, in order:

```
Hello: DriverEntry, service key \REGISTRY\MACHINE\SYSTEM\ControlSet001\Services\hello
Hello: ready as \\.\Hello
Hello: create, open handles 1
Hello: cleanup, open handles 0
Hello: close, open handles 0
Hello: unload, open handles 0
Hello: unloaded
```

**If this step fails:** no output while `sc query` reports `RUNNING` means the debug-print filter is
masking the messages; set the `Debug Print Filter` values from `wk-driver-dev-environment` and
reboot. Output that stops after `ready` means the client never reached the driver.

### Step 11 — Confirm the driver object with the debugger

Enable local kernel debugging (`bcdedit /debug on`, `bcdedit /dbgsettings local`), reboot, and attach
WinDbg through *File → Attach to Kernel → Local kernel debugging*. With the driver started, `!drvobj`
must show the unload routine and the dispatch table, and `!devobj` must show the device:

```text
lkd> !drvobj hello 2
Driver object (ffffb20c`5a1c1b60) is for:
 \Driver\hello
DriverEntry:   fffff805`0a2f1000	hello!DriverEntry
DriverUnload:  fffff805`0a2f1200	hello!HelloUnload

Dispatch routines:
[00] IRP_MJ_CREATE                      fffff805`0a2f1050	hello!HelloCreateClose
[02] IRP_MJ_CLOSE                       fffff805`0a2f1050	hello!HelloCreateClose
[0e] IRP_MJ_DEVICE_CONTROL              fffff805`0a2f1080	hello!HelloUnsupported
[12] IRP_MJ_CLEANUP                     fffff805`0a2f1050	hello!HelloCreateClose

lkd> !devobj \Device\Hello
Device object (ffffb20c`5a1e2a30) is for:
 Hello \Driver\hello DriverObject fffb20c`5a1c1b60
Current Irp 00000000 RefCount 0 Type 00000022 Flags 00000000
Device queue is not busy.
```

The listing is abridged: unshown major functions also point at `hello!HelloUnsupported`, and addresses
vary per boot. `Type 00000022` is `FILE_DEVICE_UNKNOWN`; `Flags` must **not** contain `0x80`
(`DO_DEVICE_INITIALIZING`), and `DriverUnload` must be non-zero.

**If this step fails:** `!drvobj` finding no object means the driver is not loaded; check with
`lm m hello`. Symbols that do not resolve to `hello!HelloCreateClose` mean the `.pdb` next to the
`.sys` is stale; rebuild and keep the pair together.

### Step 12 — Clean uninstall

Close the client first, then:

```powershell
# on the guest, elevated
sc.exe stop hello
sc.exe delete hello
sc.exe query hello
Remove-Item C:\Test\Hello.sys -Force
```

Expected: `STATE : 1 STOPPED` from the stop, `[SC] DeleteService SUCCESS` from the delete, then

```
[SC] EnumQueryServicesStatus:OpenService FAILED 1060:
The specified service does not exist as an installed service.
```

**If this step fails:** `sc stop` returning `FAILED 1052` or hanging means `DriverUnload` was never
set or a client still holds a handle; close every handle, and if the driver still will not stop,
reboot the guest and delete the service. `FAILED 1072` on delete means the service is marked for
deletion because a handle is open; close Device Manager and retry.

## Complete example

Three files: two complete programs and a build script. Deployment is the three commands in Step 8.

```c
// Hello.c - one device, one symbolic link, create/cleanup/close only, clean unload.
#include <ntddk.h>

#define HELLO_DEVICE_NAME   L"\\Device\\Hello"
#define HELLO_SYMLINK_NAME  L"\\??\\Hello"

// Currently open user handles, updated atomically from any context.
static volatile LONG g_OpenHandles;

static VOID
HelloUnload(
    _In_ PDRIVER_OBJECT DriverObject
    )
{
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(HELLO_SYMLINK_NAME);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "Hello: unload, open handles %d\n", (int)g_OpenHandles);

    // Reverse of DriverEntry: the link first, then the device it pointed at.
    IoDeleteSymbolicLink(&symLink);
    IoDeleteDevice(DriverObject->DeviceObject);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "Hello: unloaded\n");
}

// Create, cleanup and close differ only in bookkeeping.
static NTSTATUS
HelloCreateClose(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
    )
{
    PIO_STACK_LOCATION stack;
    LONG openHandles;

    UNREFERENCED_PARAMETER(DeviceObject);

    stack = IoGetCurrentIrpStackLocation(Irp);

    switch (stack->MajorFunction) {
    case IRP_MJ_CREATE:
        openHandles = InterlockedIncrement(&g_OpenHandles);
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "Hello: create, open handles %d\n", (int)openHandles);
        break;

    case IRP_MJ_CLEANUP:
        openHandles = InterlockedDecrement(&g_OpenHandles);
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "Hello: cleanup, open handles %d\n", (int)openHandles);
        break;

    case IRP_MJ_CLOSE:
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "Hello: close, open handles %d\n", (int)g_OpenHandles);
        break;

    default:
        break;
    }

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

// Anything else is refused explicitly instead of being left to the I/O manager.
static NTSTATUS
HelloUnsupported(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
    )
{
    UNREFERENCED_PARAMETER(DeviceObject);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_WARNING_LEVEL,
               "Hello: unsupported major function 0x%02X\n",
               IoGetCurrentIrpStackLocation(Irp)->MajorFunction);

    Irp->IoStatus.Status = STATUS_INVALID_DEVICE_REQUEST;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_INVALID_DEVICE_REQUEST;
}

NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT DriverObject,
    _In_ PUNICODE_STRING RegistryPath
    )
{
    PDEVICE_OBJECT deviceObject = NULL;
    NTSTATUS status;
    ULONG index;
    UNICODE_STRING devName = RTL_CONSTANT_STRING(HELLO_DEVICE_NAME);
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(HELLO_SYMLINK_NAME);

    // RegistryPath is valid only for the duration of this call.
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "Hello: DriverEntry, service key %wZ\n", RegistryPath);

    // Dispatch table first: a handle can be opened as soon as the link exists.
    for (index = 0; index <= IRP_MJ_MAXIMUM_FUNCTION; index++) {
        DriverObject->MajorFunction[index] = HelloUnsupported;
    }
    DriverObject->MajorFunction[IRP_MJ_CREATE] = HelloCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLEANUP] = HelloCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = HelloCreateClose;
    DriverObject->DriverUnload = HelloUnload;

    status = IoCreateDevice(DriverObject,
                            0,
                            &devName,
                            FILE_DEVICE_UNKNOWN,
                            0,
                            FALSE,
                            &deviceObject);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "Hello: IoCreateDevice failed 0x%08X\n", status);
        return status;
    }

    status = IoCreateSymbolicLink(&symLink, &devName);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "Hello: IoCreateSymbolicLink failed 0x%08X\n", status);
        // DriverUnload does not run after a failed DriverEntry: unwind here.
        IoDeleteDevice(deviceObject);
        return status;
    }

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "Hello: ready as \\\\.\\Hello\n");
    return STATUS_SUCCESS;
}
```

```c
// HelloClient.c - open \\.\Hello, hold the handle, close it.
#include <windows.h>
#include <stdio.h>

int
main(void)
{
    HANDLE device;

    device = CreateFileW(L"\\\\.\\Hello",
                         GENERIC_READ | GENERIC_WRITE,
                         0,
                         NULL,
                         OPEN_EXISTING,
                         0,
                         NULL);
    if (device == INVALID_HANDLE_VALUE) {
        printf("CreateFileW failed, GetLastError=%lu\n", GetLastError());
        return 1;
    }

    printf("Opened \\\\.\\Hello (handle %p)\n", (void *)device);
    printf("Press Enter to close the handle\n");
    (void)getchar();

    if (!CloseHandle(device)) {
        printf("CloseHandle failed, GetLastError=%lu\n", GetLastError());
        return 1;
    }
    printf("Handle closed\n");
    return 0;
}
```

```powershell
# build.ps1 - configure the VS environment, build Hello.sys, sign and verify it.
[CmdletBinding()]
param(
    [ValidateSet('Debug', 'Release')][string]$Configuration = 'Debug',
    [string]$CertSubject = 'CN=Contoso Test Driver Signing'
)

$ErrorActionPreference = 'Stop'

$vswhere = Join-Path ${env:ProgramFiles(x86)} 'Microsoft Visual Studio\Installer\vswhere.exe'
if (-not (Test-Path $vswhere)) { throw 'vswhere.exe not found; install Visual Studio 2022.' }
$vsPath = & $vswhere -latest -products * `
    -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
if (-not $vsPath) { throw 'No Visual Studio instance with the C++ x64 toolset.' }
Import-Module (Join-Path $vsPath 'Common7\Tools\Microsoft.VisualStudio.DevShell.dll')
Enter-VsDevShell -VsInstallPath $vsPath -SkipAutomaticLocation `
    -DevCmdArguments '-arch=x64 -host_arch=x64' | Out-Null

$outDir = Join-Path $PSScriptRoot "x64\$Configuration"
& msbuild.exe (Join-Path $PSScriptRoot 'Hello\Hello.vcxproj') /nologo /m /v:minimal `
    "/p:Configuration=$Configuration" '/p:Platform=x64' "/p:OutDir=$outDir\"
if ($LASTEXITCODE -ne 0) { throw "msbuild failed with exit code $LASTEXITCODE." }

$sys = Join-Path $outDir 'Hello.sys'
if (-not (Test-Path $sys)) { throw "Expected $sys; check OutDir and the project name." }

# Test certificate: created and trusted once, reused afterwards.
$cert = Get-ChildItem Cert:\LocalMachine\My |
    Where-Object { $_.Subject -eq $CertSubject -and $_.HasPrivateKey } |
    Sort-Object NotAfter -Descending | Select-Object -First 1
if (-not $cert) {
    $cert = New-SelfSignedCertificate -Type CodeSigningCert -Subject $CertSubject `
        -CertStoreLocation Cert:\LocalMachine\My -NotAfter (Get-Date).AddYears(3)
    $cer = Join-Path $PSScriptRoot 'HelloTest.cer'
    Export-Certificate -Cert $cert -FilePath $cer | Out-Null
    foreach ($store in 'Root', 'TrustedPublisher') {
        Import-Certificate -FilePath $cer -CertStoreLocation "Cert:\LocalMachine\$store" | Out-Null
    }
    Write-Host "Created and trusted $CertSubject."
}

$signtool = Get-ChildItem (Join-Path ${env:ProgramFiles(x86)} 'Windows Kits\10\bin') -Directory |
    Where-Object { $_.Name -match '^10\.\d+\.\d+\.\d+$' } |
    Sort-Object { [version]$_.Name } | Select-Object -Last 1 |
    ForEach-Object { Join-Path $_.FullName 'x64\signtool.exe' }

& $signtool sign /fd SHA256 /s My /sha1 $cert.Thumbprint /v $sys
if ($LASTEXITCODE -ne 0) { throw "signtool sign failed (exit $LASTEXITCODE)." }
& $signtool verify /pa /v $sys
if ($LASTEXITCODE -ne 0) { throw 'Signature does not verify; check the trusted root.' }
Write-Host "Built and signed: $sys" -ForegroundColor Green
```

## Variations and advanced use

**Install the same binary as a PnP software device.** An INF with a `ROOT\Hello` hardware ID puts the
driver in the driver store and in Device Manager, and needs a catalog plus a different removal
sequence; `wk-driver-build-deploy` has the INF, the scripts and the store commands.

**Add a protocol once the baseline passes.** The next increment is an IOCTL: a shared header with
`CTL_CODE`, buffered input and output, length validation. `wk-driver-anatomy` builds that driver and
`wk-ioctl-design` covers numbering and buffering. Do not add a protocol before all thirteen steps
pass — a protocol failure would hide a pipeline failure.

**KMDF instead of WDM.** *Empty KMDF Driver* replaces `IoCreateDevice` with `WdfDriverCreate` and
`EvtDeviceAdd` and needs the KMDF co-installer on the guest. The observables are identical, which
makes the KMDF build a useful second data point.

## Failure modes and diagnosis

| Symptom | Likely cause | First command / fix |
|---|---|---|
| `MSB8020: build tools for v143 cannot be found` | `Enter-VsDevShell` did not run, or the C++ x64 toolset is absent | Re-run the `build.ps1` bootstrap; install *Desktop development with C++* |
| `LNK2019: unresolved external symbol DriverEntry` | Entry point missing, renamed, or C++-mangled | `dumpbin /exports Hello.sys`; use a `.c` file or `extern "C"` |
| `LNK2038: mismatch for 'SpectreMitigation'` | Spectre-mitigated libraries missing | Install the MSVC v143 Spectre libraries for all configurations |
| `sc create` → `FAILED 87` | `ERROR_INVALID_PARAMETER`: missing space after `type=`, `binPath=` or `start=` | `sc.exe create hello type= kernel binPath= C:\Test\Hello.sys start= demand` |
| `sc start` → `FAILED 577` | `ERROR_INVALID_IMAGE_HASH`: unsigned, or test-signed with test signing off | `bcdedit /enum {current}`; enable testsigning, reboot, disable Secure Boot |
| `sc start` → `FAILED 1275` | `ERROR_DRIVER_BLOCKED`: policy block list, memory integrity, security product | `Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard` |
| `sc start` → `FAILED 31`, `STATE : 1 STOPPED` | `DriverEntry` returned a failure status | DebugView shows `IoCreateDevice failed 0xC0000035` (name collision) or the link failure |
| `sc stop` → `FAILED 1052`, driver stays loaded | `DriverObject->DriverUnload` is `NULL` | Set `DriverUnload`; a driver without it stops only on reboot |
| `CreateFileW` → error 2 | No symbolic link, or a different name | `WinObj` → `\??\Hello`; check the link status in DebugView |
| DebugView silent, `sc query` says `RUNNING` | Debug-print filter masks `INFO`, or *Capture Kernel* is off | Set the filter values, reboot, run DbgView elevated with `Ctrl+K` |

## Safety, IRQL, and cleanup checklist

- [ ] `DriverEntry` and `DriverUnload` run at `PASSIVE_LEVEL`; nothing here blocks, waits, or touches
      paged memory at raised IRQL.
- [ ] Every `DriverEntry` failure path unwinds what it created: `IoDeleteDevice` after a failed
      `IoCreateSymbolicLink`, and nothing after a failed `IoCreateDevice`.
- [ ] `DriverUnload` deletes the symbolic link before the device object.
- [ ] `IoCompleteRequest` is called exactly once per IRP on every path; the `IRP*` is not touched
      afterwards.
- [ ] `g_OpenHandles` changes only through `InterlockedIncrement`/`InterlockedDecrement`, so a create
      on one CPU and a cleanup on another cannot lose an update.
- [ ] The client closes its handle before `sc stop`; an open handle keeps the driver resident.
- [ ] Test signing and the test certificate are lab-only: turn testsigning off and remove the test
      root before the VM is used for anything else.

## Verification

1. **Build.** `.\build.ps1` ends with `Built and signed` and `0 Error(s)`.
2. **Image.** `dumpbin /headers .\x64\Debug\Hello.sys` reports `8664 machine (x64)`, and
   `dumpbin /exports` lists `DriverEntry`.
3. **Signature and policy.** `signtool verify /pa /v` prints `Successfully verified`,
   `bcdedit /enum {current}` shows `testsigning Yes`, and the Test Mode watermark is visible.
4. **Service.** `sc query hello` shows `STATE : 4 RUNNING` and `WIN32_EXIT_CODE : 0 (0x0)`.
5. **Dispatch and trace.** `HelloClient.exe` prints `Opened \\.\Hello` and `Handle closed`, and
   DebugView shows `DriverEntry`, `ready`, `create`, `cleanup`, `close` for one cycle.
7. **Debugger view.** `!drvobj hello 2` shows a non-zero `DriverUnload` and `hello!HelloCreateClose`
   at `[00]`, `[02]` and `[12]`; `!devobj` shows `Type 00000022` and no `0x80` flag.
8. **Unload.** `sc stop hello` prints `STATE : 1 STOPPED`, DebugView shows `unloaded`, and
   `sc query hello` after the delete returns `1060`.
9. **Repeatable.** Start, open, close and stop five times; each cycle produces one
   create/cleanup/close triple and one unload pair, with no `STATUS_OBJECT_NAME_COLLISION` on the
   second start.
