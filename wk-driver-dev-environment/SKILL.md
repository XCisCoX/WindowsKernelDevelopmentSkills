---
name: wk-driver-dev-environment
description: Use when setting up or repairing a Windows kernel driver toolchain, covering pinned Visual Studio 2022, Windows SDK and WDK versions, Empty WDM Driver projects, the project properties that change the binary (Target Platform Version, /W4, Spectre, Inf2Cat, signing), the DriverEntry and DriverUnload contract, a test VM with test signing, and the first build, sc load and DebugView trace.
---

# Driver development environment: from a clean host to a loaded driver

A working driver environment is a chain of exact versions. Visual Studio 2022 supplies the compiler,
the linker and MSBuild; the Windows SDK supplies the C/C++ headers, the import libraries and the
signing tools; the WDK supplies the kernel headers, the kernel libraries, the project templates and
Inf2Cat. Break one link — a WDK that does not match the installed SDK, a missing Spectre-mitigated
library, a project built for the wrong platform — and the failure surfaces as a linker error or a
refused load that looks like a code bug. The mental model that saves the most time: **building and
loading are two separate gates, and the second gate is enforced by the kernel image loader, not by
your code.** The one rule: pin every version and prove the whole pipeline with a trivial driver
before writing a single line of real driver logic.

## When to use

- Installing Visual Studio 2022, the Windows SDK and the WDK on a fresh host, or repairing a host
  where the templates, kernel headers or `inf2cat.exe` are missing.
- Deciding which driver project type to create and which project properties actually change the
  output binary.
- A build fails with missing headers, missing kernel libraries, `MSB8020`, `LNK2019` on
  `DriverEntry`, or a warning promoted to an error.
- A `.sys` builds but `sc start` refuses to load it, and the returned code is 577, 1275, 31, 39 or
  `0xC0000428`.
- Setting up a test VM: checkpoints, UEFI Secure Boot, test signing, and the debug-print filter.
- `DbgPrint`/`DbgPrintEx` output never appears in DebugView, or appears for one component and not
  another.
- Choosing between `sc create`, `pnputil` and `devcon` for a first deployment.
- Writing `DriverEntry` or `DriverUnload` and needing the exact contract (return value, IRQL,
  cleanup obligation, what happens on failure).

Do not use for: a full driver with a real protocol and IOCTLs (`wk-driver-anatomy`), the reproducible
INF/catalog/signing pipeline and driver-store management (`wk-driver-build-deploy`), kernel debugging
transport setup (`wk-kernel-debugging`), or the IRQL rules behind every kernel call
(`wk-kernel-programming-rules`).

## Prerequisites and related skills

- **Host**: Windows 10 22H2 or Windows 11, x64, 8 GB RAM minimum, 40 GB free disk, local
  Administrator rights. Driver installation and `bcdedit` both require elevation.
- **Guest**: a Windows 10 or Windows 11 x64 virtual machine, 2 vCPU and 4 GB RAM is enough. Take a
  checkpoint before every deployment; a bad driver can make the guest unbootable.
- **Firmware**: UEFI with Secure Boot **off** on the guest if test signing is required. On a physical
  host with Secure Boot on, `bcdedit /set testsigning on` is rejected by policy.
- **Headers**: `<ntddk.h>` for WDM drivers, `<ntifs.h>` when file-system or process/thread APIs are
  needed (include it *before* `<ntddk.h>`, or instead of it — it includes the latter).
- **No debugger is required** to read `DbgPrintEx` output (DebugView works), but a bugcheck can only
  be diagnosed with a kernel debugger attached: see `wk-kernel-debugging` and
  `wk-windbg-fundamentals`.

Related skills:

| Skill | Why it matters here |
|---|---|
| `wk-hello-kernel-driver` | The guided end-to-end lab that this skill prepares the machine for. |
| `wk-driver-build-deploy` | Turns the ad-hoc build in this skill into a repeatable pipeline with an INF. |
| `wk-kernel-programming-rules` | The status-code and IRQL rules that `DriverEntry` must obey. |
| `wk-device-objects` | Deep detail on `IoCreateDevice`, symbolic links and device flags. |
| `wk-kernel-tracing` | When `DbgPrintEx` is not enough: WPP and ETW. |

## Mental model

Three independent gates decide whether a driver runs. They fail with different errors, which is why
"the driver does not load" is a useless problem statement until the gate is identified.

1. **Build gate** — versions. MSBuild must find a platform toolset (v143), a Windows SDK, the WDK
   content root, and the WDK's kernel-mode property sheets. If any is missing, the build fails
   before a binary exists.
2. **Signature gate** — policy. On x64 the kernel refuses to map an image whose signature the current
   boot policy does not accept. Test signing, a trusted test certificate, Secure Boot state and
   memory integrity (HVCI) all participate.
3. **Service gate** — configuration. The image loader is invoked by the Service Control Manager on
   behalf of a service entry (`HKLM\SYSTEM\CurrentControlSet\Services\<name>`). A wrong `StartType`,
   a wrong `ImagePath`, or a `DriverEntry` that returns a failure status produces a loaded-or-not
   answer that has nothing to do with the code.

```
 .c / .h / .inf  sources
        |
        v
   MSBuild (Visual Studio 2022, v143 toolset, WDK props)
        |                        |
        |                        +--> hello.sys, hello.pdb, hello.cat (INF projects only)
        v
   kernel headers  <-- Windows Kits\10\Include\<sdk version>\km
   kernel libs     <-- Windows Kits\10\Lib\<sdk version>\km\<arch>
        |
        v
   SignTool (SHA-256) --> signature inside the PE + signature on the catalog
        |
        v
   copy to guest --> sc create / sc start (or pnputil for an INF package)
        |
        v
   ntoskrnl image loader: boot policy + signature + StartType
        |
        v
   DriverEntry at PASSIVE_LEVEL --> device object + symbolic link --> client CreateFileW
```

Two consequences follow directly from the diagram. **A build that succeeds proves nothing about
loading**, because the signature gate is evaluated later, by a different component, on the guest.
And **a load that fails leaves no trace in your code**: `DriverEntry` is not called when the image
loader rejects the image, so no `DbgPrint` appears and no breakpoint hits.

The device-object lifecycle adds a fourth, subtler gate: `IoCreateDevice` returns the object with
`DO_DEVICE_INITIALIZING` (0x80) set in `Flags`. The I/O manager clears that bit for every device on
the driver's device list once `DriverEntry` returns success. A device created later — at runtime, or
in a KMDF `EvtDeviceAdd` — must clear it manually (`DeviceObject->Flags &= ~DO_DEVICE_INITIALIZING`)
or opens never reach the driver. A stale `0x80` in `!devobj` output is the fastest confirmation.

## Key APIs and structures

Entry points and the calls that make the smallest useful driver:

| API / structure | Purpose | IRQL / context | Notes |
|---|---|---|---|
| `DriverEntry` | Driver entry point; called once when the image is mapped | `PASSIVE_LEVEL`, system thread | Must return `STATUS_SUCCESS`; a failure status unloads the image and **`DriverUnload` is never called**, so `DriverEntry` must unwind everything itself |
| `PDRIVER_OBJECT->DriverUnload` | Stop callback | `PASSIVE_LEVEL` | Leaving it `NULL` makes the driver non-unloadable; `sc stop` then fails and only a reboot removes the image |
| `PUNICODE_STRING RegistryPath` | `\Registry\Machine\System\CurrentControlSet\Services\<name>` | valid **only during `DriverEntry`** | Copy the string with `ExAllocatePool2` + `RtlStringCbCopyUnicodeString` if it is needed later; the buffer belongs to the loader |
| `IoCreateDevice` | Allocates `DEVICE_OBJECT` (+ optional extension) from non-paged pool | `PASSIVE_LEVEL` | Sets `DO_DEVICE_INITIALIZING`; a name of `NULL` creates an unnamed device |
| `IoCreateSymbolicLink` | Creates `\??\<name>` (the DOS device map) pointing at `\Device\<name>` | `PASSIVE_LEVEL` | Both arguments are `PUNICODE_STRING`; failure requires `IoDeleteDevice` |
| `IoDeleteSymbolicLink` / `IoDeleteDevice` | Undo the two calls above | `PASSIVE_LEVEL` | Reverse order; the device pointer comes from `DriverObject->DeviceObject` in `DriverUnload` |
| `RTL_CONSTANT_STRING` | Compile-time `UNICODE_STRING` from a literal | any | No runtime length scan; literal strings only |
| `RtlInitUnicodeString` | Runtime `UNICODE_STRING` from a counted literal | `<= DISPATCH_LEVEL` | Points at the caller's buffer; do not free it |
| `IoCompleteRequest` | Completes an IRP and returns it to the I/O manager | `<= DISPATCH_LEVEL` | After this call the `IRP*` is poison; never read `Irp->IoStatus` afterwards |
| `IO_NO_INCREMENT` | Priority boost argument (value 0) | n/a | Correct default for a request completed synchronously |
| `DbgPrintEx(DPFLTR_IHVDRIVER_ID, level, fmt, args)` | Filtered kernel trace | any IRQL; cost is real | Format string must live in non-paged memory; `DPFLTR_ERROR_LEVEL` = 0, `WARNING` = 1, `TRACE` = 2, `INFO` = 3 |
| `KeQueryInterruptTime` | 100 ns tick counter | any IRQL | Basis for measuring DPC duration |

Version matrix — record what is installed, then pin the project to it:

| Component | What to install | Exact check |
|---|---|---|
| Visual Studio 2022 | 17.x, workload **Desktop development with C++** | `& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -property installationVersion` |
| MSVC toolset | v143 (14.4x) with x64 build tools | `Get-ChildItem "${env:ProgramFiles}\Microsoft Visual Studio\2022\*\VC\Tools\MSVC"` |
| Spectre libraries | Individual component **MSVC v143 - VS 2022 C++ x64/x86 Spectre-mitigated libs** | `Test-Path "${env:ProgramFiles}\Microsoft Visual Studio\2022\*\VC\Tools\MSVC\*\lib\spectre"` |
| Windows SDK | 10.0.26100.0 (Windows 11 24H2) or 10.0.22621.0, Debugging Tools included | `Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\Include"` |
| WDK | The release whose version folder equals the SDK's version folder | `Test-Path "${env:ProgramFiles(x86)}\Windows Kits\10\build\10.0.26100.0"` |
| Signing tools | Ship with the SDK: `signtool.exe`, `inf2cat.exe`, `makecab.exe` | `Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\bin"` |
| Sysinternals | DebugView, WinObj, Process Explorer, Sigcheck | `Get-Command Dbgview.exe` after unzipping the suite |

**The SDK and the WDK must be the same version family.** The WDK installer copies kernel headers and
libraries into `Windows Kits\10\<Include|Lib>\<version>\km`; a project that targets a different
`Target Platform Version` than the installed WDK silently loses the kernel libraries and fails at
link time with unresolved `__security_cookie`, `IoCreateDevice` or `GsDriverEntry`.

## Step-by-step

### Step 1 — Inventory the host before installing anything

```powershell
[Environment]::OSVersion.Version.ToString()
(Get-CimInstance Win32_Processor).Architecture          # 9 = x64
Get-CimInstance Win32_LogicalDisk -Filter "DeviceID='C:'" | Select-Object FreeSpace
Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\Include" -ErrorAction SilentlyContinue
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * -property installationVersion
```

Check after: OS build 19045 (Windows 10 22H2) or 22621/26100 (Windows 11); at least 15 GB free; note
the SDK `Include` folder names, because those strings are the only legal values for
`Target Platform Version`.

### Step 2 — Install in this order

1. **Visual Studio 2022**, workload *Desktop development with C++*, plus the individual component
   *MSVC v143 - VS 2022 C++ x64/x86 Spectre-mitigated libs (Latest)*. The Spectre libraries are not
   optional: driver projects compile with `/Qspectre` and the link fails without them
   (`LNK2038: mismatch detected for 'SpectreMitigation'`).
2. **Windows 11 SDK** of the chosen version. In the installer, keep *Debugging Tools for Windows*
   selected — it provides `cdb.exe`, `windbgx` components and the symbol tooling used later.
3. **WDK for Windows 11**, same version family as the SDK. On the final wizard screen leave *Install
   Windows Driver Kit Visual Studio extension* checked; that extension is what adds the driver
   project templates and the WDK property sheets to MSBuild.
4. **Sysinternals Suite** unzipped to a tools folder. DebugView, WinObj and Process Explorer are used
   constantly; keep the folder on `PATH` or call the executables by full path.

Install order matters: the SDK must exist before the WDK, and the WDK extension must be installed
while Visual Studio is closed.

### Step 3 — Verify the installation before creating a project

```powershell
# 1) driver templates are registered
Test-Path "${env:ProgramFiles}\Microsoft Visual Studio\2022\*\Common7\IDE\VC\VCTargets\Driver"
# 2) kernel headers and libraries for the SDK version exist
Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\Include\10.0.26100.0\km" -ErrorAction SilentlyContinue |
    Select-Object -First 5 Name
# 3) the tools resolve
Get-ChildItem "${env:ProgramFiles(x86)}\Windows Kits\10\bin\10.0.26100.0\x64" -Filter *.exe |
    Where-Object Name -in 'signtool.exe','inf2cat.exe','stampinf.exe','makecab.exe'
# 4) the Spectre runtime exists
Get-ChildItem "${env:ProgramFiles}\Microsoft Visual Studio\2022\*\VC\Tools\MSVC\*\lib\spectre\x64" -ErrorAction SilentlyContinue
```

Expected: four `*.exe` names from step 3 (`inf2cat.exe`, `makecab.exe`, `signtool.exe`,
`stampinf.exe`), at least one `.h` under `km`, and a `spectre\x64` folder containing `*.lib` files.
If any check returns nothing, fix the install now; every later step depends on it.

### Step 4 — Create the driver project

In Visual Studio: *File → New → Project →* search `WDM` → **Empty WDM Driver** (C++). Name the
project `Hello`, keep *Place solution and project in the same directory* unchecked, target
`x64`, and create it.

The wizard produces a project with a single `Hello.inf` under *Driver Files*. For a first driver that
is deployed with `sc create`, an INF is not needed: right-click the INF, choose *Remove*, then delete
`Hello.inf` from disk. Removing it from the project (rather than deleting the file only) is what stops
Inf2Cat from running with nothing to catalog.

Add `Hello.c`: right-click *Source Files* → *Add → New Item → C++ File (.cpp)*, then rename it to
`Hello.c`. A `.c` file needs no `extern "C"`; a `.cpp` file **must** wrap `DriverEntry` in
`extern "C"`, otherwise the linker cannot find the entry point on any architecture and reports
`LNK2019: unresolved external symbol DriverEntry`.

Check after: *Solution Explorer* shows *Driver Files* (empty is fine) and `Hello.c`; the
configuration dropdown reads `Debug` and the platform dropdown reads `x64`.

### Step 5 — Set the properties that change the binary

Right-click the project → *Properties*. Set the configuration dropdown to *All Configurations* and
the platform to *All Platforms* so the change survives a Debug/Release switch.

| Property (path in the dialog) | Value | Why it matters |
|---|---|---|
| *Configuration Properties → General → Configuration Type* | `Driver` | Selects the kernel-mode link rules; `Application` produces a user-mode `.exe` |
| *Driver Settings → General → Target Platform Version* | the installed SDK folder, e.g. `10.0.26100.0` | Chooses the WDK content roots; a mismatch loses `km` libraries |
| *Driver Settings → General → Target Platform* | `Desktop` | Not `Universal`; Universal adds API-set constraints that reject most WDM code |
| *C/C++ → General → Warning Level* | `Level4 (/W4)` | Kernel code is reviewed by the compiler, not by a runtime |
| *C/C++ → General → Treat Warnings As Errors* | `Yes (/WX)` | Keep it on; disabling it to silence C4100 hides real bugs |
| *C/C++ → Code Generation → Spectre Mitigation* | `Enabled (/Qspectre)` | Must match the Spectre libraries; required by driver projects |
| *C/C++ → Code Generation → Enable C++ Exceptions* | `No` | No C++ exceptions in the kernel; `try`/`catch` cannot unwind across a driver boundary |
| *C/C++ → Language → C++ Language Standard* | `ISO C++17 (/std:c++17)` | Template default; do not lower it below C++14 |
| *C/C++ → Optimization → Basic Runtime Checks* | `Default` | `RTC1` is incompatible with kernel-mode code |
| *Driver Signing → General → Sign Mode* | `Off` for this walkthrough | Signing is done explicitly with `signtool` in a later step; `Test Sign` needs a certificate configured first |
| *Driver Settings → General → Run Inf2Cat* | `No` when the INF was removed | Inf2Cat with no INF in the package fails the build |

Check after: *Apply*, then *OK*, then open the `.vcxproj` and confirm the properties landed as XML
(`<SpectreMitigation>Spectre</SpectreMitigation>` and a `WindowsTargetPlatformVersion` value). A
property set only for the active configuration is the most common cause of "it builds in Debug but
not Release".

### Step 6 — Honor the `DriverEntry` and `DriverUnload` contract

```c
NTSTATUS
DriverEntry(
    _In_ PDRIVER_OBJECT  DriverObject,
    _In_ PUNICODE_STRING RegistryPath
    );

static VOID
HelloUnload(
    _In_ PDRIVER_OBJECT  DriverObject
    );
```

- Runnable once, at `PASSIVE_LEVEL`, in the context of the system thread that performs the load.
  Paging is legal, waits are legal, and the registry is open for business.
- **Return value is the contract.** `STATUS_SUCCESS` means "the driver owns everything it created".
  Any other value means the load fails: the image is unmapped, the service reports the failure, and
  **`DriverUnload` is not called**. Every partially built object must be destroyed inside
  `DriverEntry` on the failure path (`IoDeleteDevice` after a failed `IoCreateSymbolicLink`).
- `DriverObject->DriverUnload` is a `VOID` callback with the same `PDRIVER_OBJECT`. Set it before
  returning success; without it the driver cannot be stopped.
- `RegistryPath` is owned by the loader and is **only valid while `DriverEntry` runs**. Read it or
  copy it there; never store the pointer.
- Order inside `DriverEntry`: create the device, create the symbolic link, clear
  `DO_DEVICE_INITIALIZING` if the device was created outside `DriverEntry`, then set the dispatch
  table. Handles can be opened the moment the symbolic link exists, so the dispatch table must be
  complete before that point.

Check after: the driver compiles with `/W4 /WX` and no `UNREFERENCED_PARAMETER` omissions. An unused
`RegistryPath` parameter is a C4100 error at `/WX`, not a warning.

### Step 7 — Build from the command line, not only from the IDE

```powershell
$vsPath = & "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" `
    -latest -products * -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 `
    -property installationPath
Import-Module "$vsPath\Common7\Tools\Microsoft.VisualStudio.DevShell.dll"
Enter-VsDevShell -VsInstallPath $vsPath -SkipAutomaticLocation -DevCmdArguments '-arch=x64 -host_arch=x64'

msbuild.exe .\Hello\Hello.vcxproj /nologo /m /v:minimal `
    /p:Configuration=Debug /p:Platform=x64
```

Expected output ends with:

```
Build succeeded.
    0 Warning(s)
    0 Error(s)
```

and `Hello\x64\Debug\Hello.sys` exists. `Enter-VsDevShell` is what puts `msbuild`, `cl`, `link`,
`WindowsSdkDir` and `WDKContentRoot` into the session; a plain PowerShell window without it usually
fails with `MSB8020` or missing WDK property sheets.

Check after: `Get-Item .\Hello\x64\Debug\Hello.sys | Select-Object Length` returns a value around
10–30 KB. A 3 KB file usually means the linker produced an empty image because `DriverEntry` was not
found.

### Step 8 — Prepare the guest: checkpoint, Secure Boot, test signing

```powershell
# on the Hyper-V host
Checkpoint-VM -Name DevVM -SnapshotName BeforeFirstDriver
# on the guest, elevated
bcdedit /set testsigning on
bcdedit /enum "{current}"
```

Expected: the last command shows `testsigning             Yes` in the Windows Boot Loader block.
Reboot the guest. After the reboot the desktop shows a **Test Mode** watermark in the lower-right
corner; that watermark is the proof, not the command output.

Two failures are common:

- `An error has occurred setting the element data. The value is protected by Secure Boot policy and
  cannot be modified or deleted.` — Secure Boot is on. Turn it off in the VM firmware (Hyper-V Gen2:
  *VM Settings → Security → uncheck Enable Secure Boot*; VirtualBox: *System → Motherboard → EFI*
  with Secure Boot unchecked; VMware: firmware type UEFI with Secure Boot unchecked), then retry.
- `bcdedit` says the operation completed but the watermark never appears — the machine rebooted into
  a different boot entry, or a policy (Device Guard) re-applied the old value. Confirm with
  `bcdedit /enum {current} | Select-String testsigning` after the reboot.

Check after: watermark visible, `testsigning Yes` printed, and a fresh checkpoint named after the
working state.

### Step 9 — Deploy with the Service Control Manager and read the result

```powershell
# elevated, on the guest, after copying Hello.sys to C:\Test
sc.exe create hello type= kernel binPath= C:\Test\Hello.sys start= demand
sc.exe start hello
sc.exe query hello
```

Expected (the spacing is the SCM's, not yours):

```
[SC] CreateService SUCCESS

SERVICE_NAME: hello
        TYPE               : 1  KERNEL_DRIVER
        STATE              : 4  RUNNING
                                (STOPPABLE, NOT_PAUSABLE, IGNORES_SHUTDOWN)
        WIN32_EXIT_CODE    : 0  (0x0)
        SERVICE_EXIT_CODE  : 0  (0x0)
        CHECKPOINT         : 0x0
        WAIT_HINT          : 0x0
        PID                : 0
        FLAGS              :
```

`PID : 0` is normal: a driver is not a process. `STATE : 4 RUNNING` means the image is mapped and
`DriverEntry` returned success; a driver whose `DriverEntry` failed usually reports
`STATE : 1 STOPPED` plus a non-zero `WIN32_EXIT_CODE` (often 31 or 577).

Rules for `sc` syntax that cost an hour each time they are forgotten:

- `type=`, `binPath=` and `start=` need a space **after** the equals sign and none before it.
  `type=kernel` fails with `[SC] CreateService FAILED 87` (`ERROR_INVALID_PARAMETER`).
- `binPath=` takes the full path of the `.sys` file including the file name.
- `start= demand` writes `StartType = 3`; omit it and the service defaults to demand start anyway,
  while `start= system` (1) and `start= boot` (0) require a reboot to take effect.
- Deleting the service (`sc delete hello`) does not delete the file.

Check after: `sc query hello` shows `RUNNING`; `Get-CimInstance Win32_SystemDriver -Filter
"Name='hello'"` shows `State = Running` and `Started = True`.

### Step 10 — Make the trace visible

The debug-print filter is read at boot. Add it once, then reboot:

```powershell
$key = 'HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Debug Print Filter'
New-Item -Path $key -Force | Out-Null
New-ItemProperty -Path $key -Name DEFAULT   -PropertyType DWord -Value 0xF -Force | Out-Null
New-ItemProperty -Path $key -Name IHVDRIVER -PropertyType DWord -Value 0xF -Force | Out-Null
Restart-Computer
```

- `DEFAULT` is the mask used by plain `DbgPrint`/`KdPrint`. Any value whose bit 3 is set (`0x8` or
  higher) makes `DbgPrint` output visible; `0xF` enables levels 0–3.
- `IHVDRIVER` is the mask for `DbgPrintEx(DPFLTR_IHVDRIVER_ID, level, fmt, args)`. Without it, `INFO`
  output from that component can be filtered out even though `ERROR`-level output still appears.

Then, on the guest: run `Dbgview.exe` **elevated**, press `Ctrl+K` (*Capture → Capture Kernel*), and
disable *Capture Win32* so user-mode noise does not bury the driver output. Start the driver and
expect a line similar to:

```
[1234] FirstDriver: DriverEntry, device \Device\FirstDriver created
```

Check after: the message appears exactly once per `sc start`, and a second message appears on
`sc stop`. If nothing appears, the driver never ran (check `sc query`) or the filter is wrong
(re-check the registry value and reboot).

### Step 11 — Know the DPC latency baseline before adding DPCs

DPC latency is the delay between a hardware interrupt and the completion of the deferred work that
interrupt queued. It is a per-CPU property, it is what audio dropouts and USB timeouts look like from
the outside, and every driver that queues a DPC contributes to it.

- **Measure with a trace, not with a print.** `wpr -start CPU -start DiskIO -fileMode`, reproduce
  the workload, `wpr -stop C:\traces\hello.etl`, then open the ETL in Windows Performance Analyzer
  and read *DPC/ISR Duration by Module* and *DPC Duration by CPU*. A single driver dominating that
  graph is the culprit.
- **Measure inside the driver** with `KeQueryInterruptTime()`, which returns 100 ns units and is
  callable at any IRQL: capture it on DPC entry, capture it again on exit, keep the maximum in a
  global, print the maximum on unload. Anything approaching 100 µs is a DPC that should be a work
  item instead.
- **Do not block or touch paged memory in a DPC.** A DPC runs at `DISPATCH_LEVEL`; a long loop, a
  wait, or a paging access there raises latency for every device on that CPU and eventually trips the
  DPC watchdog (bugcheck `0x133 DPC_WATCHDOG_VIOLATION`). The hello driver has no DPC; `wk-dpcs`
  covers `KeInitializeDpc`, `KeInsertQueueDpc`, timer DPCs and unload safety when one is added.

Check after: a baseline trace taken on an idle guest, so later regressions can be compared against
something. Keep the ETL next to the checkpoint name.

### Build checklist

- [ ] `Target Platform Version` equals an installed SDK folder name, and the WDK matches that
      version family.
- [ ] `Platform` is `x64` (or `arm64`) in *Configuration Manager*, and the `.sys` is built for the
      guest's architecture.
- [ ] `/W4` and `/WX` are on for **all** configurations, including Release.
- [ ] `Spectre Mitigation` is `Enabled` and the Spectre libraries are installed.
- [ ] C++ exceptions are off; no `/EHsc`, no RTTI, no STL, no `new`/`delete`.
- [ ] `Sign Mode` is intentional: `Off` when signing with an explicit `signtool` command, `Test Sign`
      when a certificate is configured in the project.
- [ ] `Run Inf2Cat` is `No` when the package has no INF; `Yes` and correct when it has one.
- [ ] The build emits `Hello.sys` plus `Hello.pdb`; no `LNK2019` on `DriverEntry`.
- [ ] The `.sys` is copied to the guest, not run from a network share (the loader maps it from a
      local path in `System32\drivers` or wherever `ImagePath` points).

### Deploy checklist

- [ ] `sc start` returns `STATE : 4 RUNNING`.
- [ ] DebugView (elevated, Capture Kernel) shows the `DriverEntry` line.
- [ ] The client can open the device, and closing the last handle produces the unload trace.
- [ ] `sc stop` succeeds and `sc query` reports `STOPPED`; a driver with no `DriverUnload` cannot
      reach this state.

## Complete example

A complete minimal WDM driver. It builds with the settings above, creates one named device and one
symbolic link, accepts and completes create/close requests, traces every transition, and unwinds
correctly on every failure path. Save as `Hello.c` in the project.

```c
// Hello.c - minimal WDM driver: builds, loads, traces, unloads cleanly.
#include <ntddk.h>

#define DEVICE_NAME  L"\\Device\\Hello"
#define SYMLINK_NAME L"\\??\\Hello"

// Unload: undo DriverEntry in reverse order. The driver object is the only
// argument; the device object is reached through DriverObject->DeviceObject.
static VOID
HelloUnload(
    _In_ PDRIVER_OBJECT DriverObject
    )
{
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(SYMLINK_NAME);

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "Hello: DriverUnload, deleting device and symbolic link\n");

    IoDeleteSymbolicLink(&symLink);
    IoDeleteDevice(DriverObject->DeviceObject);
}

// Create and close share one routine: both only need to succeed.
static NTSTATUS
HelloCreateClose(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
    )
{
    UNREFERENCED_PARAMETER(DeviceObject);

    // The IRP must be completed with the same status that is returned.
    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

// Any request that is not create/close/cleanup is rejected explicitly.
static NTSTATUS
HelloUnsupported(
    _In_ PDEVICE_OBJECT DeviceObject,
    _In_ PIRP Irp
    )
{
    UNREFERENCED_PARAMETER(DeviceObject);

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
    UNICODE_STRING devName = RTL_CONSTANT_STRING(DEVICE_NAME);
    UNICODE_STRING symLink = RTL_CONSTANT_STRING(SYMLINK_NAME);

    // Valid only during this call: log it here, never store the pointer.
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "Hello: DriverEntry, registry path %wZ\n", RegistryPath);

    // 1. Dispatch table first, so the device is never reachable half-configured.
    for (index = 0; index <= IRP_MJ_MAXIMUM_FUNCTION; index++) {
        DriverObject->MajorFunction[index] = HelloUnsupported;
    }
    DriverObject->MajorFunction[IRP_MJ_CREATE] = HelloCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = HelloCreateClose;
    DriverObject->DriverUnload = HelloUnload;

    // 2. Named device object. No extension: this driver keeps no per-device state.
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

    // 3. Symbolic link, so user mode can reach the device as \\.\Hello.
    status = IoCreateSymbolicLink(&symLink, &devName);
    if (!NT_SUCCESS(status)) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_ERROR_LEVEL,
                   "Hello: IoCreateSymbolicLink failed 0x%08X\n", status);
        IoDeleteDevice(deviceObject);   // DriverUnload will not be called.
        return status;
    }

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "Hello: driver ready, open \\\\.\\Hello\n");
    return STATUS_SUCCESS;
}
```

Build it with the command from Step 7, deploy it with the commands from Step 9, and expect the
following sequence in DebugView for a start, one client connection, and a stop:

```
Hello: DriverEntry, registry path \REGISTRY\MACHINE\SYSTEM\ControlSet001\Services\hello
Hello: driver ready, open \\.\Hello
Hello: DriverUnload, deleting device and symbolic link
```

## Variations and advanced use

**KMDF instead of WDM.** *Empty KMDF Driver* produces a project that links `WdfLdr`/`Wdf01000` and
calls `WdfDriverCreate` instead of `IoCreateDevice`; deployment then requires a matching KMDF
co-installer (or the framework already present in the guest). Choose KMDF for new PnP drivers and
WDM for teaching or for code that must run without the framework. The project properties are the
same, plus `KMDF Version Major/Minor` under *Driver Settings*.

**ARM64.** Add the platform in *Configuration Manager*, set `Target Platform Version` again (the
property is per platform), and note that test signing works the same way but the guest must be ARM64.
Cross-building on an x64 host is supported; running is not.

**Repairing a broken install.** If the templates disappeared but the toolset is intact, rerun the
WDK installer and choose *Change → Install Windows Driver Kit Visual Studio extension*. If the
templates appear but MSBuild cannot import the WDK property sheets, the extension was installed while
Visual Studio was running: close VS, repair the extension, reopen.

## Failure modes and diagnosis

| Symptom | Likely cause | First command / fix |
|---|---|---|
| `[SC] CreateService FAILED 87` | `ERROR_INVALID_PARAMETER`: missing space after `type=`, `binPath=` or `start=` | `sc.exe create hello type= kernel binPath= C:\Test\Hello.sys` (space after each `=`) |
| `sc start` → `[SC] StartService FAILED 577` | `ERROR_INVALID_IMAGE_HASH`: the image has no signature the current policy accepts (unsigned, or test-signed with test signing off) | `bcdedit /enum {current}` → `testsigning`; `bcdedit /set testsigning on` and reboot with Secure Boot off |
| `sc start` → `[SC] StartService FAILED 1275` | `ERROR_DRIVER_BLOCKED`: a policy block list, memory integrity (HVCI) rejecting the image, or a security product vetoing it | `Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard` → `SecurityServicesRunning`; turn off *Memory integrity* in Windows Security |
| `sc start` → `FAILED 31` with `STATE : 1 STOPPED` | `DriverEntry` returned a failure status; the SCM can only report a generic failure | Add an `ERROR_LEVEL` trace on every failure path; `0xC0000035` means the device name already exists, `0xC0000023` a bad name |
| Device Manager *Code 31* ("cannot load the drivers required") | The service exists but the driver object failed to start, often a missing dependency or a KMDF version mismatch | `Get-CimInstance Win32_SystemDriver -Filter "Name='hello'"`; check the KMDF major/minor against the guest's `Wdf01000.sys` |
| Device Manager *Code 39* ("driver may be corrupted or missing") | `ImagePath` points at a file that is absent, renamed or built for another architecture | `Test-Path C:\Test\Hello.sys`; `dumpbin /headers Hello.sys` → machine `8664` for x64 |
| `0xC0000428` in an event log or device status | `STATUS_INVALID_IMAGE_HASH`, the status behind 577 and Device Manager code 52 | Treat as 577: fix the signing state, then re-check with `signtool verify /pa Hello.sys` |
| `sc start` → `FAILED 1058` | The service is disabled (`StartType = 4`, written by a failed PnP install or by policy) | `sc qc hello` → `START_TYPE`; `sc config hello start= demand` |
| `sc stop` fails, or the driver stays loaded | `DriverUnload` is `NULL`, or a client still holds a handle to the device | Set `DriverObject->DriverUnload`; close every handle before `sc stop` |
| `MSB8020: The build tools for v143 cannot be found` | The C++ x64 toolset is not installed, or PowerShell was not initialized with `Enter-VsDevShell` | Rerun the VS installer, then the `Enter-VsDevShell` block from Step 7 |
| `LNK2019: unresolved external symbol DriverEntry` | The entry point has C++ linkage (`.cpp` without `extern "C"`) or was renamed | Add `extern "C"`, or rename the file to `.c`; confirm with `dumpbin /exports Hello.sys` |
| `error C2220: the following warning is treated as an error` + `C4100` | `/WX` plus an unused parameter | `UNREFERENCED_PARAMETER(RegistryPath);` (or use the parameter) — do not disable `/WX` |
| `LNK2038: mismatch detected for 'SpectreMitigation'` | Spectre-mitigated libraries missing, or mixed `/Qspectre` settings between objects | Install the Spectre libraries; set the property for *All Configurations* |
| DebugView shows nothing while `sc query` says `RUNNING` | Filter mask, DebugView not elevated, *Capture Kernel* off, or the driver was loaded before the filter reboot | Re-check the `Debug Print Filter` values, reboot, run DbgView elevated with `Ctrl+K` |

## Safety, IRQL, and cleanup checklist

- [ ] `DriverEntry` runs at `PASSIVE_LEVEL`; every wait, registry read and file open there is legal,
      but the guest is already usable, so a hang in `DriverEntry` hangs the `sc start` call.
- [ ] A failure return from `DriverEntry` skips `DriverUnload`: delete every object created so far,
      in reverse order, before returning.
- [ ] `DriverUnload` runs at `PASSIVE_LEVEL`; it must delete the symbolic link **before** the device,
      and it must not touch a device pointer that could already be gone.
- [ ] `IoCompleteRequest` returns the IRP to the I/O manager: after it, the `IRP*` must never be
      dereferenced (returning `Irp->IoStatus.Status` after the call is a use-after-free that
      bugchecks).
- [ ] Never hold an open handle while stopping the driver: the object manager keeps a reference to
      the device and the unload cannot complete.
- [ ] Keep `DO_DEVICE_INITIALIZING` out of the running device: the I/O manager clears it after
      `DriverEntry`; clear it manually for devices created later.
- [ ] The registry path pointer dies with `DriverEntry`; copy it if it is needed after the return.
- [ ] Test signing is a lab-only state: turn it off (`bcdedit /set testsigning off`) before handing a
      machine to anyone else, and never ship a binary that only loads in Test Mode.

## Verification

Run in order; each step has a pass condition.

1. **Toolchain present.** `Test-Path "${env:ProgramFiles(x86)}\Windows Kits\10\bin\*\x64\inf2cat.exe"`
   returns `True`.
2. **Build clean.**
   `msbuild.exe .\Hello\Hello.vcxproj /p:Configuration=Debug /p:Platform=x64 /v:minimal` →
   `0 Warning(s)`, `0 Error(s)`.
3. **Image is a driver.** `dumpbin /headers .\Hello\x64\Debug\Hello.sys | Select-String machine` →
   `8664 machine (x64)`; `dumpbin /exports` lists `DriverEntry`.
4. **Guest trusts the boot policy.** `bcdedit /enum {current} | findstr /i testsigning` shows
   `Yes`, and the Test Mode watermark is on the desktop.
5. **Service starts.** `sc.exe query hello` → `STATE : 4 RUNNING`, `WIN32_EXIT_CODE : 0 (0x0)`.
6. **Code ran.** DebugView (elevated, Capture Kernel) shows `Hello: driver ready, open \\.\Hello`.
7. **Device is reachable.** `Get-Item \\.\Hello` or the client from `wk-hello-kernel-driver` opens
   the handle without `ERROR_FILE_NOT_FOUND (2)`.
8. **Unload is clean.** `sc.exe stop hello` → `STATE : 1 STOPPED`; DebugView shows the
   `DriverUnload` line; a second `sc start` succeeds, proving the device name was released.
9. **No leak across cycles.** Start and stop five times in a row; each cycle must produce exactly one
   entry line and one unload line. A `STATUS_OBJECT_NAME_COLLISION` on the second start means the
   unload path did not delete the device or the link.
