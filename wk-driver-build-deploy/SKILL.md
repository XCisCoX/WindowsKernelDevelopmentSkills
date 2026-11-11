---
name: wk-driver-build-deploy
description: Use when a driver must build and install reproducibly, covering msbuild configuration and platform arguments, StampInf stamping, Inf2Cat catalog generation and the sign-binary-before-catalog order, SHA-256 signtool signing, a software-device INF with a ROOT hardware ID, pnputil and devcon installation, driver store inspection and removal, and load failures 577, 1275, 31, 39, 0xC0000428.
---

# Reproducible driver builds and repeatable installation

A driver that only installs when a human clicks through Device Manager is not deployable. The goal of
this skill is a pipeline with three idempotent commands: one builds and signs, one installs and
verifies, one removes everything without leaving an orphan in the driver store. The mental model is
that a driver package is **four artifacts that must agree with each other** — the `.sys`, the `.inf`,
the `.cat`, and the service configuration — and that the catalog is a set of hashes over the final
bytes of the other files. The one rule: **sign the binary first, generate the catalog second, sign
the catalog third, and never edit a packaged file afterwards.** Violate the order and installation
fails with a hash mismatch that looks like corruption but is really a stale catalog.

## When to use

- Building the same driver on a developer machine and on a build agent, with identical output.
- Turning a hand-run `signtool` invocation into a script with a certificate, a catalog and a
  verification step.
- Creating a software device (a `ROOT\`-enumerated device with an INF), rather than a bare service.
- Installing a driver package with `pnputil`, `devcon` or `sc` and knowing exactly which one creates
  the device node.
- A package installs but the device shows a yellow mark: code 31, code 39, code 52, or status
  `0xC0000428`.
- `pnputil /add-driver` succeeds but nothing appears in Device Manager.
- Removing a driver cleanly: service, device node, driver store entry, and files.
- A build must continue to work after the developer machine is patched to a new SDK.

Do not use for: the first-ever toolchain install and first load (`wk-driver-dev-environment`), the
guided end-to-end lab with expected output at every step (`wk-hello-kernel-driver`), or the driver's
own IRP protocol (`wk-driver-anatomy`).

## Prerequisites and related skills

- **Toolchain**: Visual Studio 2022 with the C++ workload, a Windows SDK and a WDK of the same
  version family, and the Spectre-mitigated libraries; `wk-driver-dev-environment` covers the install
  order and the exact version checks.
- **Rights and target state**: local Administrator on the build machine (certificate stores,
  `signtool`) and on the target (service creation, driver store writes); test signing on and Secure
  Boot off for test-signed packages, or a production certificate chaining to a Microsoft-trusted root.
- **Tools**: `inf2cat.exe`, `stampinf.exe`, `signtool.exe` and `makecab.exe` from
  `Windows Kits\10\bin\<version>\x64`; `pnputil.exe` and `sc.exe` from `System32`; `devcon.exe` is
  **not** part of Windows and must be obtained separately (WDK samples) to create a root device node.
- **Format**: the INF is plain text with CRLF endings, and a UTF-16 or UTF-8-with-BOM INF is rejected
  by Inf2Cat. The driver needs `<ntddk.h>`; a client needs `<windows.h>`.

Siblings: `wk-driver-dev-environment` produces the project this pipeline builds;
`wk-hello-kernel-driver` validates the pipeline end to end; `wk-device-objects` explains what the
INF's device node becomes in the object namespace; `wk-driver-anatomy` is the driver code that the
service entry loads.

## Mental model

A driver package is a directory, not a file. Windows installs the directory as a unit: the INF
describes the device and the service, the catalog carries hashes of every file the INF names, and the
driver store keeps a private copy under a hashed directory name. Four stages, in this order:

```
 1. BUILD       hello.vcxproj --msbuild--> hello.sys  (+ hello.pdb)
                StampInf rewrites DriverVer and the version resource if the INF/sources
                contain $STAMPED$ style variables, so the date comes from the build

 2. SIGN BIN    hello.sys --signtool sign--> signature appended to the PE image
                (the file changes here: any hash computed earlier is now wrong)

 3. CATALOG     inf2cat /driver:<dir> /os:10_X64 --> hello.cat
                (hashes of every file the INF names, taken from the signed bytes)

 4. SIGN CAT    hello.cat --signtool sign--> catalog signature
                then: copy to target, install, start, verify, inspect, remove
```

Why the order cannot be rearranged:

- `Inf2Cat` reads the files named by `CopyFiles` and stores their hashes. Signing the `.sys` after the
  catalog is generated changes those bytes, so the catalog no longer describes the file. Setup then
  fails with *"the hash for the file is not present in the specified catalog file"* and the device
  lands on code 52 or code 39.
- The catalog signature covers the catalog file itself, so it must be the **last** write to the
  package. Any later edit — a version resource, an INF string, a re-link — invalidates the catalog,
  which is why the pipeline regenerates both artifacts on every build instead of caching them.

Installation is a separate contract from building:

| Model | Who creates the device node | Command | Use when |
|---|---|---|---|
| Legacy service | Nobody; there is no device node, the driver creates its own device in `DriverEntry` | `sc create` with `type= kernel` | Non-PnP software drivers, quick lab work, drivers with no hardware ID |
| PnP software device | The INF plus an installer that creates a `ROOT\` node | `pnputil /add-driver x.inf /install` for staging, `devcon install x.inf ROOT\Hello` to create the node | A driver that must participate in PnP (power, surprise removal, driver store versioning) |

`pnputil /add-driver` **stages** a package into the driver store. Adding `/install` applies it to
devices that already match. Neither creates a new `ROOT\`-enumerated node: a root device exists only
after something asks the root enumerator for it, which is what `devcon install` does. Confusing these
two behaviors is the most common reason "the INF installed but there is no device".

## Key APIs and structures

Tools, with the exact location and the flag that matters:

| Tool | Path | Purpose and critical flag |
|---|---|---|
| `msbuild.exe` | resolved through `Enter-VsDevShell` | Compiles and links; `/p:Configuration`, `/p:Platform`, `/p:OutDir` |
| `stampinf.exe` | `Windows Kits\10\bin\<ver>\x64` | Rewrites `DriverVer` and the `.sys` version resource from build variables; `-d` sets the date, `-v` the version |
| `inf2cat.exe` | same directory | Builds the catalog; `/driver:<dir>` (no space after the colon), `/os:<list>` (comma separated, no spaces), `/verbose` |
| `signtool.exe` | same directory | `sign /fd SHA256 /s My /sha1 <thumb>`; `verify /pa` for a driver package, `/kp` for kernel policy |
| `makecab.exe` | same directory | Packs the package into a `.cab` for distribution; not needed for local installation |
| `pnputil.exe` | `C:\Windows\System32` | `/add-driver <inf> [/install]`, `/enum-drivers`, `/delete-driver <oemN.inf> /uninstall /force`, `/enum-devices /class System` |
| `devcon.exe` | WDK samples, not in `System32` | `install <inf> <hwid>` creates a root node; `remove <hwid>`; `update <inf> <hwid>` |
| `sc.exe` | `C:\Windows\System32` | `create`, `start`, `stop`, `delete`, `qc` for the legacy model |
| `Get-WindowsDriver` | DISM PowerShell module | `-Online` lists the driver store as objects, with `Driver` = `oemN.inf` and `OriginalFileName` |
| `sigcheck.exe` | Sysinternals | `-a -i` shows the signature and the catalog for a file, without SetupAPI |

INF directives that change behavior, and the values that are not negotiable:

| Directive | Section | Meaning and required value |
|---|---|---|
| `Signature` | `[Version]` | `"$WINDOWS NT$"` — anything else fails Inf2Cat signability |
| `CatalogFile` | `[Version]` | The `.cat` name; without it Inf2Cat writes no catalog and the package cannot be PnP-installed |
| `DriverVer` | `[Version]` | `MM/DD/YYYY,version`. The date must not precede the target OS release date, or Inf2Cat fails with error `22.9.7` |
| `Class` / `ClassGuid` | `[Version]` | `System` / `{4D36E97D-E325-11CE-BFC1-08002BE10318}` for a software device; the pair must match |
| `PnpLockdown` | `[Version]` | `1` makes the driver store own the files, so edits to `System32\drivers` do not affect the installed package |
| `[Manufacturer]` | — | `%Name% = ModelsSection, NTamd64` (or `NTarm64`); the decoration selects the per-architecture models section |
| `CopyFiles` | `[DDInstall]` | Names the file list section; `%12%` is `System32\drivers` |
| `AddService` | `[DDInstall.Services]` | `AddService = hello, 0x00000002, Hello_Service`; `0x2` is `SPSVCINST_ASSOCSERVICE`, marking it the function driver |
| `ServiceType` | service section | `1` = `SERVICE_KERNEL_DRIVER` |
| `StartType` | service section | `0` boot, `1` system, `2` auto, `3` demand, `4` disabled |
| `ErrorControl` | service section | `0` ignore, `1` normal, `2` severe, `3` critical (a failure to load bugchecks during boot) |
| `ServiceBinary` | service section | `%12%\hello.sys` |

## Step-by-step

### Step 1 — Pin the build inputs and record them

```powershell
$kitRoot = Join-Path ${env:ProgramFiles(x86)} 'Windows Kits\10'
Get-ChildItem (Join-Path $kitRoot 'bin') -Directory | Select-Object Name
& "${env:ProgramFiles(x86)}\Microsoft Visual Studio\Installer\vswhere.exe" -latest -products * `
    -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationVersion
```

Check after: exactly one SDK `bin` version matches the project's `Target Platform Version`, and
`inf2cat.exe`, `signtool.exe` and `stampinf.exe` all exist under that one `bin\<version>\x64`.

### Step 2 — Choose the installation model before writing the INF

- The driver creates its own device in `DriverEntry` and needs no power or PnP management: **legacy
  service**. No INF is needed; deploy with `sc`. This is what `wk-hello-kernel-driver` does.
- The driver must be a PnP function driver, appear in Device Manager, and be versioned in the driver
  store: **PnP software device** with a `ROOT\` hardware ID.

A legacy driver cannot be updated through the driver store, and a PnP driver whose node was never
created never receives `IRP_MN_START_DEVICE`, which looks like a load failure.

### Step 3 — Write the INF and validate it before building anything

Write `hello.inf` as shown in *Complete example*, then check the three things Inf2Cat is strict
about:

1. **The file list is complete.** Every file named in `CopyFiles` must exist in the package
   directory; Inf2Cat reports a numbered error naming the missing file.
2. **The date is not stale.** Inf2Cat compares `DriverVer` against the release date of each `/os`
   target and fails with `22.9.7: DriverVer set to incorrect date`.
3. **The decoration matches.** `NTamd64` in both the `[Manufacturer]` line and the models section for
   x64; `NTarm64` and `[Models.NTarm64]` for ARM64. A mismatch installs nothing, silently.

Check after: `InfVerif /k hello.inf` (ships with the WDK) reports no violations; anything it flags
would also fail a Windows Hardware Compatibility submission.

### Step 4 — Build with explicit properties

```powershell
$outDir = Join-Path $PSScriptRoot 'out\x64\Release'
msbuild.exe .\Hello\Hello.vcxproj /nologo /m /v:minimal `
    /p:Configuration=Release /p:Platform=x64 `
    "/p:OutDir=$outDir\" "/p:IntDir=$outDir\obj\"
```

Expected: `Build succeeded.`, `0 Warning(s)`, `0 Error(s)`, and `hello.sys` in `$outDir`. Setting
`OutDir` explicitly is what makes the rest of the pipeline path-stable; relying on the default
`$(SolutionDir)$(Platform)\$(Configuration)\` gives a different directory when the project is built
outside its solution.

Check after: `Get-ChildItem $outDir` shows `hello.sys` and `hello.inf`. If the INF is missing, copy
it from the project directory; Inf2Cat only looks at the directory it is pointed at.

### Step 5 — Create or locate the signing certificate, then sign the binary

`build.ps1` creates and trusts the test certificate on its first run; the conditions it satisfies are
the ones test signing checks. The certificate must carry the code signing EKU `1.3.6.1.5.5.7.3.3`
(set by `New-SelfSignedCertificate -Type CodeSigningCert`), it must have a private key in a store
`signtool` can read, and its root must be in `LocalMachine\Root` **and**
`LocalMachine\TrustedPublisher` on the target — otherwise a correctly signed driver still fails to
load with 577.

Signing by hand once the certificate exists, with `$cert` fetched from `Cert:\LocalMachine\My`:

```powershell
& signtool.exe sign /fd SHA256 /s My /sha1 $cert.Thumbprint /v .\out\x64\Release\hello.sys
```

Expected: `Successfully signed:` followed by the full path, then `Number of errors: 0`. Add
`/tr http://timestamp.digicert.com /td SHA256` only when the machine has internet access and the
signature must outlive the certificate; an isolated test network should skip timestamping rather than
fail on it.

Check after: `signtool verify /pa /v hello.sys` prints `Successfully verified`. A chain error here is
a certificate-trust problem that becomes a load failure 577 later.

### Step 6 — Generate the catalog from the signed binary

```powershell
& inf2cat.exe /driver:"C:\dev\Hello\out\x64\Release" /os:10_X64 /verbose
```

Success means exit code 0 and `hello.cat` next to the INF. Failure prints a header and numbered rows:

```
Signability test failed.
Errors:
22.9.7: DriverVer set to incorrect date (must be postdated to 4/21/2009 for newest OS) in \hello.inf
```

Read the number, not the prose: `22.9.x` is an INF syntax or date rule, and a message about a missing
file means the directory lacks a file that `CopyFiles` promises. For ARM64 the `/os` token is
`10_VB_ARM64`, and the accepted token list changes between WDK releases: confirm with `inf2cat /?`.

Check after: `hello.cat` exists and is not zero length, and double-clicking it lists `hello.sys` —
that file list is the proof that the catalog covers the right bytes.

### Step 7 — Sign the catalog, then verify the pair

```powershell
& signtool.exe sign /fd SHA256 /s My /sha1 $cert.Thumbprint /v .\out\x64\Release\hello.cat
& signtool.exe verify /pa /v .\out\x64\Release\hello.sys
& signtool.exe verify /pa /v .\out\x64\Release\hello.cat
```

Expected: `Successfully verified` for both files. Both must be signed: the catalog signature is what
PnP checks during installation, and the embedded signature is what the loader checks when the image is
mapped (mandatory for boot-start drivers, and required for x64 in general).

Check after: `sigcheck -a -i hello.sys` shows `Catalog: hello.cat` and a valid signature.

### Step 8 — Deploy: copy, install, start

Legacy service (no INF, no device node created by SetupAPI):

```powershell
New-Item -ItemType Directory -Path C:\Test\Hello -Force | Out-Null
Copy-Item .\out\x64\Release\hello.sys C:\Test\Hello\ -Force
sc.exe create hello type= kernel binPath= C:\Test\Hello\hello.sys start= demand
sc.exe start hello
sc.exe query hello
```

PnP software device:

```powershell
New-Item -ItemType Directory -Path C:\Test\Hello -Force | Out-Null
Copy-Item .\out\x64\Release\* C:\Test\Hello\ -Force
pnputil.exe /add-driver C:\Test\Hello\hello.inf /install
devcon.exe install C:\Test\Hello\hello.inf "ROOT\Hello"
```

Expected from `pnputil` (abridged; the trailing counts are omitted):

```
Microsoft PnP Utility

Adding driver package:  hello.inf
Driver package added successfully.
Published Name:         oem12.inf
```

`Published Name` is the driver store identity (`oemN.inf`) and the handle used for removal; wording
varies between Windows 10 and 11, so capture the value, not the sentence.

Check after: `Get-CimInstance Win32_SystemDriver -Filter "Name='hello'"` returns the service with
`State = Running` (legacy) and `pnputil /enum-devices /class System` lists
`Hello Test Device` with problem code 0 (PnP).

### Step 9 — Inspect the driver store instead of guessing

```powershell
pnputil.exe /enum-drivers
Get-WindowsDriver -Online | Where-Object { $_.OriginalFileName -like '*hello.inf' } |
    Select-Object Driver, OriginalFileName, ProviderName, ClassName, Version, Date
Get-ChildItem C:\Windows\System32\DriverStore\FileRepository -Filter 'hello.inf_*' -Directory
pnputil.exe /enum-devices /class System
```

Expected: exactly one `hello.inf` entry, one `oemN.inf` published name, and one
`hello.inf_amd64_<hex>` folder in `FileRepository`; the `Driver` property from `Get-WindowsDriver` is
the value `pnputil /delete-driver` needs.

Check after: a second staged copy of the same INF means an earlier install never removed its package;
each copy consumes a new `oemN.inf` number and the oldest can resurface on the next hardware
detection.

### Step 10 — Remove in reverse order, and prove the store is clean

```powershell
sc.exe stop hello
sc.exe delete hello
$package = Get-WindowsDriver -Online | Where-Object { $_.OriginalFileName -like '*hello.inf' }
foreach ($p in $package) { pnputil.exe /delete-driver $p.Driver /uninstall /force }
```

Order matters: stop the service before deleting it, remove the device node before the package (use
`devcon remove "ROOT\Hello"` or Device Manager → *Uninstall device*), and delete the store entry
last. `pnputil /delete-driver` refuses to remove a package that a device is still using unless
`/force` is supplied, and `/force` leaves that device without a driver.

Check after: `Get-WindowsDriver -Online | Where-Object OriginalFileName -like '*hello.inf'` returns
nothing, `sc.exe query hello` reports `1060` (service does not exist), and
`Test-Path C:\Windows\System32\drivers\hello.sys` is `False` for a `%12%`-based INF.

## Complete example

Three files, in one directory that also becomes the package directory. The INF is x64; for ARM64
duplicate the models section as `[Contoso.NTarm64]` and add `NTarm64` to the `[Manufacturer]` line.

```ini
; hello.inf - root-enumerated software device with one kernel service.
[Version]
Signature       = "$WINDOWS NT$"
Class           = System
ClassGuid       = {4D36E97D-E325-11CE-BFC1-08002BE10318}
Provider        = %ProviderName%
CatalogFile     = hello.cat
DriverVer       = 11/11/2026,1.0.0.0
PnpLockdown     = 1

[DestinationDirs]
DefaultDestDir  = 12

[Manufacturer]
%ProviderName%  = Contoso, NTamd64

[Contoso.NTamd64]
%DeviceName%    = Hello_Install, ROOT\Hello

[Hello_Install]
CopyFiles       = Hello_CopyFiles

[Hello_Install.Services]
AddService      = hello, 0x00000002, Hello_Service

[Hello_Service]
DisplayName     = %ServiceDisplayName%
ServiceType     = 1
StartType       = 3
ErrorControl    = 1
ServiceBinary   = %12%\hello.sys

[Hello_CopyFiles]
hello.sys

[SourceDisksNames]
1 = %DiskName%,,,""

[SourceDisksFiles]
hello.sys = 1,,

[Strings]
ProviderName        = "Contoso Labs"
DeviceName          = "Hello Test Device"
ServiceDisplayName  = "Hello Test Driver"
DiskName            = "Hello Driver Package"
```

`build.ps1` builds, signs the binary, catalogs, signs the catalog and verifies both signatures:

```powershell
# build.ps1 - compile, embed-sign, catalog, sign the catalog, verify.
[CmdletBinding()]
param(
    [ValidateSet('Debug', 'Release')][string]$Configuration = 'Release',
    [ValidateSet('x64', 'ARM64')][string]$Platform = 'x64',
    [string]$Project = (Join-Path $PSScriptRoot 'Hello\Hello.vcxproj'),
    [string]$CertSubject = 'CN=Contoso Test Driver Signing',
    [string]$TimestampUrl = ''
)

$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$kitRoot = Join-Path ${env:ProgramFiles(x86)} 'Windows Kits\10'
$outDir  = Join-Path $PSScriptRoot "out\$Platform\$Configuration"

function Get-KitTool {
    param([Parameter(Mandatory)][string]$Name)
    $bin = Join-Path $kitRoot 'bin'
    $newest = Get-ChildItem $bin -Directory |
        Where-Object { $_.Name -match '^10\.\d+\.\d+\.\d+$' } |
        Sort-Object { [version]$_.Name } | Select-Object -Last 1
    if (-not $newest) { throw "No SDK bin directory under $bin." }
    $tool = Join-Path $newest.FullName "x64\$Name"
    if (-not (Test-Path $tool)) { throw "$Name is missing from $($newest.FullName)." }
    return $tool
}

# 0. Test certificate: create once, trust its root locally.
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

# 1. Visual Studio environment: msbuild, cl, link, WindowsSdkDir, WDKContentRoot.
$vswhere = Join-Path ${env:ProgramFiles(x86)} 'Microsoft Visual Studio\Installer\vswhere.exe'
if (-not (Test-Path $vswhere)) { throw 'vswhere.exe not found; install Visual Studio 2022.' }
$vsPath = & $vswhere -latest -products * `
    -requires Microsoft.VisualStudio.Component.VC.Tools.x86.x64 -property installationPath
if (-not $vsPath) { throw 'No Visual Studio instance with the C++ x64 toolset.' }
Import-Module (Join-Path $vsPath 'Common7\Tools\Microsoft.VisualStudio.DevShell.dll')
Enter-VsDevShell -VsInstallPath $vsPath -SkipAutomaticLocation `
    -DevCmdArguments "-arch=$($Platform.ToLower()) -host_arch=x64" | Out-Null

# 2. Compile and link into a stable directory.
& msbuild.exe $Project /nologo /m /v:minimal `
    "/p:Configuration=$Configuration" "/p:Platform=$Platform" `
    "/p:OutDir=$outDir\" "/p:IntDir=$outDir\obj\"
if ($LASTEXITCODE -ne 0) { throw "msbuild failed with exit code $LASTEXITCODE." }

$sys = Join-Path $outDir 'hello.sys'
$inf = Join-Path $outDir 'hello.inf'
if (-not (Test-Path $inf)) {
    Copy-Item -Path (Join-Path $PSScriptRoot 'Hello\hello.inf') -Destination $inf
}
foreach ($file in @($sys, $inf)) { if (-not (Test-Path $file)) { throw "Missing output: $file" } }

# 3. Sign the binary before cataloging: the signature changes the bytes.
$signtool = Get-KitTool 'signtool.exe'
$signArgs = @('sign', '/fd', 'SHA256', '/s', 'My', '/sha1', $cert.Thumbprint, '/v')
if ($TimestampUrl) { $signArgs += @('/tr', $TimestampUrl, '/td', 'SHA256') }
& $signtool @signArgs $sys
if ($LASTEXITCODE -ne 0) { throw "signtool failed on $sys (exit $LASTEXITCODE)." }

# 4. Catalog every file the INF names, using the now-final bytes.
$inf2cat = Get-KitTool 'inf2cat.exe'
$osList = if ($Platform -eq 'x64') { '10_X64' } else { '10_VB_ARM64' }
& $inf2cat "/driver:$outDir" "/os:$osList" /verbose
if ($LASTEXITCODE -ne 0) { throw "inf2cat failed (exit $LASTEXITCODE); fix the numbered errors." }

$cat = Join-Path $outDir 'hello.cat'
if (-not (Test-Path $cat)) { throw 'No catalog produced. Does [Version] declare CatalogFile=hello.cat?' }

# 5. Sign the catalog last, then prove both signatures.
& $signtool @signArgs $cat
if ($LASTEXITCODE -ne 0) { throw "signtool failed on $cat (exit $LASTEXITCODE)." }
& $signtool verify /pa /v $sys
if ($LASTEXITCODE -ne 0) { throw 'Embedded signature does not verify.' }
& $signtool verify /pa /v $cat
if ($LASTEXITCODE -ne 0) { throw 'Catalog signature does not verify.' }
Write-Host "Package ready: $outDir" -ForegroundColor Green
```

`deploy.ps1` copies the package, installs it with either model, starts it and verifies both the
service and the device symlink:

```powershell
# deploy.ps1 - copy, install, start, verify.
[CmdletBinding()]
param(
    [ValidateSet('Debug', 'Release')][string]$Configuration = 'Release',
    [ValidateSet('x64', 'ARM64')][string]$Platform = 'x64',
    [ValidateSet('Legacy', 'PnP')][string]$Method = 'Legacy',
    [string]$TargetDir = 'C:\Test\Hello',
    [string]$ServiceName = 'hello',
    [string]$DevicePath = '\\.\Hello'
)

$ErrorActionPreference = 'Stop'
Set-StrictMode -Version Latest

$principal = [Security.Principal.WindowsPrincipal]::new(
    [Security.Principal.WindowsIdentity]::GetCurrent())
if (-not $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    throw 'Run this script from an elevated PowerShell console.'
}

function Test-DeviceOpen {
    param([string]$Path)
    if (-not ('DeployProbe.Native' -as [type])) {
        Add-Type -Namespace DeployProbe -Name Native -MemberDefinition @'
[System.Runtime.InteropServices.DllImport("kernel32.dll", CharSet = System.Runtime.InteropServices.CharSet.Unicode, SetLastError = true)]
public static extern System.IntPtr CreateFileW(string name, uint access, uint share,
    System.IntPtr attributes, uint disposition, uint flags, System.IntPtr template);
[System.Runtime.InteropServices.DllImport("kernel32.dll", SetLastError = true)]
public static extern bool CloseHandle(System.IntPtr handle);
'@
    }
    $handle = [DeployProbe.Native]::CreateFileW($Path, 0x80000000, 0,
        [System.IntPtr]::Zero, 3, 0, [System.IntPtr]::Zero)
    if ($handle.ToInt64() -eq -1) {
        $code = [Runtime.InteropServices.Marshal]::GetLastWin32Error()
        return [pscustomobject]@{ Open = $false; LastError = $code }
    }
    [void][DeployProbe.Native]::CloseHandle($handle)
    return [pscustomobject]@{ Open = $true; LastError = 0 }
}

$package = Join-Path $PSScriptRoot "out\$Platform\$Configuration"
foreach ($name in 'hello.sys', 'hello.cat', 'hello.inf') {
    $source = Join-Path $package $name
    if (-not (Test-Path $source)) { throw "Package incomplete: $source. Run build.ps1 first." }
}

New-Item -ItemType Directory -Path $TargetDir -Force | Out-Null
Copy-Item -Path (Join-Path $package '*') -Destination $TargetDir -Force
$sysPath = Join-Path $TargetDir 'hello.sys'
$infPath = Join-Path $TargetDir 'hello.inf'

if ($Method -eq 'Legacy') {
    $existing = Get-CimInstance Win32_SystemDriver -Filter "Name='$ServiceName'" -ErrorAction SilentlyContinue
    if ($existing) {
        if ($existing.State -eq 'Running') {
            & sc.exe stop $ServiceName | Out-Null
            Start-Sleep -Seconds 1
        }
        & sc.exe delete $ServiceName | Out-Null
    }
    & sc.exe create $ServiceName type= kernel binPath= $sysPath start= demand
    if ($LASTEXITCODE -ne 0) { throw "sc create failed with $LASTEXITCODE." }
    & sc.exe start $ServiceName
    if ($LASTEXITCODE -ne 0) { throw "sc start failed with $LASTEXITCODE (577 unsigned, 1275 blocked)." }
}
else {
    & pnputil.exe /add-driver $infPath /install
    if ($LASTEXITCODE -ne 0) { throw "pnputil failed with $LASTEXITCODE." }
    $devcon = (Get-Command devcon.exe -ErrorAction SilentlyContinue).Source
    if (-not $devcon) { throw 'devcon.exe not found: package staged, no ROOT device created.' }
    & $devcon install $infPath 'ROOT\Hello'
    if ($LASTEXITCODE -ne 0) { throw "devcon install failed with $LASTEXITCODE." }
}

Start-Sleep -Seconds 1
$driver = Get-CimInstance Win32_SystemDriver -Filter "Name='$ServiceName'" -ErrorAction SilentlyContinue
if (-not $driver) { throw "Service $ServiceName is not registered. Check the install step." }
if ($driver.State -ne 'Running') {
    throw "Service state is $($driver.State), expected Running. Check event log and setupapi.dev.log."
}

$probe = Test-DeviceOpen -Path $DevicePath
if (-not $probe.Open) {
    $text = [ComponentModel.Win32Exception]::new($probe.LastError).Message
    throw "CreateFileW($DevicePath) failed with $($probe.LastError) ($text)."
}
Write-Host "OK: $ServiceName $($driver.State); $DevicePath opened and closed." -ForegroundColor Green
```

Removal is the installation sequence in reverse: `sc stop`, `sc delete`, remove the device node
(`devcon remove "ROOT\Hello"` or Device Manager), `pnputil /delete-driver <oemN.inf> /uninstall
/force`, then delete `C:\Test\Hello`. The `oemN.inf` value comes from
`Get-WindowsDriver -Online | Where-Object { $_.OriginalFileName -like '*hello.inf' }`, whose
`Driver` property is exactly what `pnputil` wants. A package that refuses to delete is still in use:
reboot and retry before reaching for `/force`, which leaves the device driverless.

## Variations and advanced use

**StampInf for version management.** Keep a literal `DriverVer` date in the INF and bump it before a
release, or let `stampinf.exe` rewrite it during the build when the INF carries `;$MODIFIED$` markers
and the project supplies `/p:DriverVer`. Either way, the date in the INF is the date that ends up in
the driver store listing.

**Distribution package.** `makecab` with a directive file packs `hello.inf`, `hello.sys` and
`hello.cat` into `hello.cab`; `pnputil /add-driver hello.cab` stages from a cab. A cab is the only
form that guarantees the three files stay together in transit, because SetupAPI unpacks it rather
than trusting a directory listing.

**Production signing.** Replace the self-signed certificate with an EV or standard code-signing
certificate from a CA, sign with SHA-256 and a timestamp, then submit the package for Microsoft
attestation signing: since Windows 10 1607, a driver on a Secure Boot machine must carry a Microsoft
signature in addition to the publisher's. Never leave a test certificate's root installed on a
production machine.

**CI without a desktop session.** A CI agent needs `Enter-VsDevShell`, the pinned SDK and WDK, and
`Windows Kits\10\bin`. Without the signing key it can still build and catalog, then hand the unsigned
`.sys` and `.cat` to a signing step on the machine that holds the key — which is also the correct way
to keep a production key off the build agent.

## Failure modes and diagnosis

| Symptom | Likely cause | First command / fix |
|---|---|---|
| `sc start` → `FAILED 577`, event log mentions the image hash | `ERROR_INVALID_IMAGE_HASH`: unsigned image, or test-signed while test signing is off | `bcdedit /enum {current}` for `testsigning`; `signtool verify /pa hello.sys` |
| `sc start` → `FAILED 1275` | `ERROR_DRIVER_BLOCKED`: block list, memory integrity (HVCI), or a security product | `Get-CimInstance -Namespace root\Microsoft\Windows\DeviceGuard -ClassName Win32_DeviceGuard`; disable *Memory integrity* and retry |
| Device Manager code 31, "cannot load the drivers required" | The service exists but the driver object failed to start; a `DriverEntry` failure or a KMDF version mismatch | `sc qc hello`; `Select-String -Path C:\Windows\INF\setupapi.dev.log -Pattern 'hello' -Context 3,3` |
| Device Manager code 39, "driver may be corrupted or missing" | `ImagePath`/`ServiceBinary` points at a missing file, or the binary is the wrong architecture | `Test-Path C:\Windows\System32\drivers\hello.sys`; `dumpbin /headers hello.sys` shows `8664` for x64, `AA64` for ARM64 |
| Device status shows `0xC0000428` | `STATUS_INVALID_IMAGE_HASH` — the loader rejected the signature; the status behind 577 and code 52 | Fix the signing state; `signtool verify /pa /kp hello.sys` |
| `sc start` → `FAILED 52F` / 1327 `ERROR_ACCOUNT_RESTRICTION` | The start request was refused by the caller's token or a policy, not by the image loader | Run from an elevated console of a full administrator; `whoami /groups` to find a restricting group; a signature problem reports 577 or `0xC0000428` |
| `pnputil`: "the hash for the file is not present in the specified catalog file" | The catalog was generated before the `.sys` was signed, or a packaged file changed afterwards | Rerun `build.ps1`; never sign or edit a file after Inf2Cat |
| Inf2Cat: `22.9.7: DriverVer set to incorrect date` | `DriverVer` predates the target OS release | Set a current date in `[Version]` and rebuild the catalog |
| Inf2Cat: "Signability test failed" with a missing-file row | A file named in `CopyFiles` is not in the `/driver` directory | `Get-ChildItem` the package directory; copy the INF next to the `.sys` |
| `pnputil /add-driver` succeeds, no device appears | The package is staged but no `ROOT\` node exists | `devcon install hello.inf ROOT\Hello`, or add the device manually |
| `sc create` → `FAILED 1073` | The service already exists | `sc delete hello`, or pick a new name; the error is `ERROR_SERVICE_EXISTS` |
| `sc delete` → `FAILED 1072` | The service is marked for deletion because a handle is still open | Close Device Manager and any handle to the device, then retry or reboot |
| Copy to `System32\drivers` → "file in use" (32) | The image is loaded; Windows locks a mapped driver image | `sc stop hello` first; if it cannot stop, schedule the copy for boot with `MoveFileEx` |
| `bcdedit /set testsigning on` → "protected by Secure Boot policy" | Secure Boot is on | Turn off Secure Boot in firmware, then retry and reboot |
| `pnputil /delete-driver` → access denied or "in use" | A device still uses the package | Remove the device node first (`devcon remove "ROOT\Hello"`), then `/uninstall /force` |
| Device installs but the client gets `ERROR_FILE_NOT_FOUND` (2) | The driver never created the symbolic link, or created it under a different name | `WinObj` → `\??\Hello`; check `IoCreateSymbolicLink` status in the trace |

## Safety, IRQL, and cleanup checklist

- [ ] The catalog is regenerated on every build, after signing the binary and before signing the
      catalog. Cached catalogs are the source of "hash not present" failures.
- [ ] Test signing is a lab state: `bcdedit /set testsigning off` and remove the test root from
      `LocalMachine\Root` before a machine leaves the lab.
- [ ] A test private key stays out of source control. `New-SelfSignedCertificate` puts the key in the
      machine store; export it only to a protected location.
- [ ] `StartType = 0` (boot) or `1` (system) plus `ErrorControl = 3` turns a driver bug into a boot
      bugcheck; keep `3`/`1` for lab packages.
- [ ] Removal order is stop → delete service → remove device node → delete store package → delete
      files. Skipping the device node leaves a phantom device that reinstalls the package.
- [ ] `pnputil /delete-driver /force` removes a package a device is using; the device is left without
      a driver and may require a reboot to recover.
- [ ] The service name in the INF (`AddService = hello`) and the client's device path (`\\.\Hello`)
      are two different identifiers; renaming one without the other breaks the client, not the load.
- [ ] Checkpoints before installing: an installed driver that bugchecks at boot can make a VM
      unbootable until the checkpoint is restored.
- [ ] Keep the `.pdb` next to the `.sys` you keep for debugging; the driver store does not preserve
      symbols.

## Verification

1. **Build reproducible.** Run `build.ps1` twice from a clean `out` directory; both runs end with
   `Package ready` and a `hello.sys` that differs only in the signature timestamp.
2. **Signatures valid and paired.** `signtool verify /pa /v hello.sys` and `signtool verify /pa /v
   hello.cat` both print `Successfully verified`, and `sigcheck -a -i hello.sys` shows the catalog.
3. **Staged once.** `pnputil /enum-drivers` lists exactly one `hello.inf` entry and one `oemN.inf`
   published name.
4. **Installed and running.** `deploy.ps1` prints `OK: hello Running; \\.\Hello opened and closed.`
5. **PnP health.** `pnputil /enum-devices /class System` shows the device with problem code 0; any
   other code means SetupAPI has a diagnostic in `C:\Windows\INF\setupapi.dev.log`.
6. **Idempotent redeploy.** Run `deploy.ps1` three times; the third run succeeds with the same output
   and does not create a second `oemN.inf` (legacy) or a second device node (PnP).
7. **Clean removal.** Follow the removal sequence; `Get-WindowsDriver -Online` no longer lists the
   package, `sc query hello` returns `1060`, and the target directory is gone.
