---
name: wk-crash-dump-analysis
description: Use when a machine bugchecks, when a stop code must become a root cause, or when a system hangs without a crash. Covers bugcheck mechanics and parameters, CrashControl and bcdedit dump configuration, symbol setup, the !analyze -v workflow, stack/thread/IRP/pool commands, a stop-code table with first commands, hang triage with 0x133, 0x101, !stacks and !locks, capturing a dump from a hung VM, and verifying the fix.
---

# Crash dump analysis: turn a stop code into a root cause and a fix

A bugcheck is evidence, not a diagnosis: the parameters record what the kernel noticed at the instant it gave up, and the stack records where the code was, not necessarily what was wrong. The job is to reconstruct one causal chain that explains every argument, the IRQL, the stack, and the state of the memory involved — and then to change the code so the same chain cannot form again. Most wasted analysis comes from accepting the debugger's `Probably caused by` line without checking whether the blamed module could really have caused that fault from that context.

## When to use

- A test machine or VM stops with a bugcheck and a dump exists, or should have.
- A stop code must be explained before the next build: `0x1E`, `0x50`, `0xA`, `0xD1`, `0x133`, `0x101`, `0x9F`, `0xC2`, `0x139`.
- A machine is hung, unresponsive, or reboots with no dump, and the state must be captured before it is lost.
- Dump capture must be configured before a stress run so the next failure is diagnosable.
- A fix must be proven: same reproducer, same verifier flags, no new dump.
- Do not use for: attaching a live debugger and stepping (`wk-kernel-debugging`), WinDbg basics (`wk-windbg-fundamentals`), deciding which exception the driver should have caught (`wk-kernel-exceptions`), or making a routine IRQL-safe (`wk-irql`).

## Prerequisites and related skills

- WinDbg (the Store build or the Debugging Tools for Windows package) on the analysis machine; `kd` is equivalent for scripted work.
- The exact binaries and PDBs that were running when the dump was taken. A dump is only as useful as the symbol match.
- A dump configured and present. Without one, the first task is to configure capture and reproduce, not to guess.
- Sibling skills: `wk-kernel-debugging` (live kernel sessions, transports, breakpoints), `wk-windbg-fundamentals` (symbols, breakpoints, `dt`), `wk-irql` (why `0xA`/`0xD1`/`0x50` happen), `wk-dpcs` (the mechanism behind `0x133`), `wk-synchronization` (deadlock shapes behind `0x101`), `wk-driver-build-deploy` (installing the rebuilt driver), `wk-kernel-exceptions` (the exception codes inside `0x1E`/`0x3B`/`0x7E`/`0x8E`).

## Mental model

A bugcheck is a controlled stop, not a random failure. The kernel raises IRQL to `HIGH_LEVEL`, records the code and four parameters, freezes the other processors so the memory image is coherent, optionally breaks into an attached kernel debugger, and then writes the dump. It cannot use the normal storage stack — the file system, volume, and filter drivers may be exactly what failed — so it writes through a minimal crash-dump stack to the paging file on the boot volume. On the next boot the session manager notices the pending image, copies it to the configured dump path, and clears the paging file.

```text
  KeBugCheckEx(code, p1..p4)
        |
        +-- raise IRQL to HIGH_LEVEL, save code and parameters
        +-- freeze the other processors (IPI), keep their stacks
        +-- if a kernel debugger is attached: break in, stay stopped
        +-- write the image through the dump stack to the boot pagefile
        |      (only the disk miniport plus registered dump filters run)
        +-- display the stop screen, reboot if AutoReboot = 1
        |
  next boot: session manager copies the image to DumpFile / MinidumpDir
        |
        v
  a .dmp file that holds physical memory, kernel state, and the stacks
```

Two consequences shape every analysis. First, **what is in the file depends on the dump type**: a minidump has the faulting stack and the module list but no pool and no other processors' stacks, so a pool-corruption theory cannot be tested on one. Second, **the dump is physical memory**, so only what was resident was captured; a kernel dump has no user memory, which is why user-mode stacks and heap are missing from it.

Each documented stop code defines what its four parameters mean, and the same position means different things for different codes: for `0xD1` the second parameter is the IRQL and the fourth is the faulting instruction, while for `0x9F` the third is a triage-block pointer and the fourth is the blocked IRP. `!analyze -v` prints the documented meaning beside each argument, and `!analyze -show <code> <p1> <p2> <p3> <p4>` prints the same interpretation without a dump. Never read a parameter by position alone.

| Dump type | `CrashDumpEnabled` | Contains | Typical size | Paging-file requirement |
|---|---|---|---|---|
| None | `0` | Nothing | n/a | n/a |
| Complete | `1` | All physical memory | About installed RAM | RAM + 257 MB |
| Kernel | `2` | Kernel-mode memory only, no user memory | 200 MB – 2 GB | Large enough for kernel memory; a few GB is safe |
| Small (minidump) | `3` | Stop code, parameters, faulting thread stack, module list | 256 KB on Windows 8+, 64 KB earlier | 256 KB |
| Automatic | `7` | Kernel dump with boot-time paging-file sizing (Windows 8+) | Like kernel | Works only with a system-managed paging file |
| Active | (Windows 10+ option) | Complete minus pages known unused and minus guest VM memory | Smaller than complete | Like complete |

`DumpFile` defaults to `%SystemRoot%\MEMORY.DMP`; small dumps go to `%SystemRoot%\Minidump` (`MinidumpDir`). The dump is written to the paging file on the **boot volume**, so a machine whose paging file lives elsewhere, or which has none, needs `DedicatedDumpFile` with `IgnorePagefileSize = 1`.

A dump is physical memory at one instant. It has no ordering information, so two events microseconds apart look simultaneous; I/O that was in flight appears only as driver state, never as a sequence; a kernel dump has no user memory; and a minidump has neither pool nor the other processors. Any theory that depends on timing has to be tested by reproduction, not by reading the file more carefully.

## Key APIs and structures

| Item | Purpose | Notes |
|---|---|---|
| `KeBugCheckEx(Code, P1, P2, P3, P4)` | Stop with parameters | Documented codes define what the four parameters mean; `KeBugCheck(Code)` is the no-parameter form |
| `!analyze -v` | Automatic triage | Prints the description, parameter meanings, faulting IP, stack, and a bucket ID |
| `!analyze -show <code> [p1..p4]` | Explain a code without a dump | Use it on the stop code from an event log |
| `!analyze -hang` | Hang analysis | Looks for blocked threads and lock owners |
| `.bugcheck` | Raw code and parameters | The ground truth before any interpretation |
| `.dump /f <file>` | Write a dump from a live kernel session | `/m` small, `/ka` active kernel plus user memory, `/k` kernel only |
| `.crash` | Force the target to bugcheck (`0xE2`) | Use only when a dump is configured and you want the crash path exercised |
| `!process 0 7` | All processes with thread stacks | Heavy on a complete dump |
| `!thread <ETHREAD>` | One thread: state, wait reason, stack | Follow `!stacks` addresses |
| `!stacks 2 [pattern]` | Every thread with its blocker | The fastest way to see one lock blocking everything |
| `!irp <addr>` | IRP stack locations, pending state | For `0x9F` use the IRP in arg4 |
| `!pool <addr>` | Owning pool allocation and tag | Needs a kernel or complete dump |
| `!locks` | ERESOURCE ownership and contention | Pair with `!thread` on the owner |
| `!cs -l` | All critical sections | `!cs <addr>` for one |

Configuration, verified rather than assumed:

| Setting | Where | Value to use on a test machine |
|---|---|---|
| Dump type | `HKLM\SYSTEM\CurrentControlSet\Control\CrashControl` → `CrashDumpEnabled` | `2` kernel or `7` automatic; `1` complete for corruption work |
| Dump path | Same key → `DumpFile`, `MinidumpDir` | A volume with room, on the boot disk |
| Overwrite | Same key → `Overwrite` | `1`; with `0`, a second crash leaves the old dump and writes nothing |
| Auto reboot | Same key → `AutoReboot` | `1` normally, `0` while watching a stop screen |
| All of the above by UI | `SystemPropertiesAdvanced.exe` → Advanced → Startup and Recovery → Settings | Same values as `CrashDumpEnabled`, `DumpFile`, `MinidumpDir`, `AutoReboot`, `Overwrite` |
| Dump filters | Same key → `DumpFilters` | Must include `dumpfve.sys` when the boot volume is BitLocker-encrypted |
| Enable capture | `bcdedit /set {default} crashdump on` | Inspect with `bcdedit /enum {current}` |
| Dedicated file | `DedicatedDumpFile`, `IgnorePagefileSize = 1` | When the paging file cannot hold a kernel dump |

Verification commands for the configuration itself:

```powershell
Get-CimInstance Win32_OSRecoveryConfiguration |
    Format-List DebugInfoType, DebugFilePath, MiniDumpDirectory, AutoReboot,
                OverwriteExistingDebugFile
Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl' |
    Select-Object CrashDumpEnabled, DumpFile, MinidumpDir, Overwrite, AutoReboot,
                  DumpFilters
Get-WinEvent -FilterHashtable @{LogName = 'System';
    ProviderName = 'Microsoft-Windows-WER-SystemErrorReporting'} -MaxEvents 5 |
    Format-List TimeCreated, Id, Message
```

`DebugInfoType` uses the same numbering as `CrashDumpEnabled`. Event ID 1001 from the WER system-error provider carries the stop code and parameters in its text, which is enough for a first triage when the dump is missing.

Symbol setup, in the order that avoids surprises:

```text
kd> .symfix
kd> .sympath+ C:\build\out
kd> .reload
kd> lmvm nt
kd> lmvm sensord
```

`_NT_SYMBOL_PATH=srv*C:\Symbols*https://msdl.microsoft.com/download/symbols` in the environment does the same for `.symfix`. Put the private PDB **and** the matching `.sys` in `C:\build\out`; `!sym noisy` shows exactly which path failed. `lmvm` must report the same build string that the dump header shows, and the same timestamp and image size as the binary you shipped.

The commands that answer the questions analysis actually asks:

| Question | Command |
|---|---|
| What stopped and with which parameters? | `.bugcheck`, `!analyze -v` |
| Is the blamed binary the one I built? | `lmvm <module>`, then compare timestamp, size, and PDB path |
| What was each processor doing? | `!running -t`, `~<n>s` to switch, `kv` there |
| Who was running when it died? | `!thread`, `!process 0 0` |
| Who is blocked on what? | `!stacks 2`, `!locks`, `!cs -l` |
| What IRP was stuck? | `!irp <addr>`, `!irpfind` |
| Which device stack is involved? | `!devobj <addr>`, `!drvobj <driver> 2`, `!devstack <addr>` |
| What memory was it? | `!pool <addr>`, `!poolused 2`, `!vm` |
| Did verifier catch it first? | `!verifier 3 <driver>`, `!verifier` |
| What does this code mean in general? | `!analyze -show <code> <params>` |

## Stop codes and first commands

The codes below cover most driver work. Read the arguments before the stack: they tell you what class of fault it is, and the stack tells you where.

| Code | Name | Arguments and meaning | First commands |
|---|---|---|---|
| `0x1E` | `KMODE_EXCEPTION_NOT_HANDLED` | Arg1 exception code, arg2 faulting address; unhandled kernel exception | `!analyze -v`, `kv`, `!pool <arg2>` |
| `0x3B` | `SYSTEM_SERVICE_EXCEPTION` | Arg1 exception code, arg2 address; fault during a system call | `!analyze -v`, `kv`, `!thread` |
| `0x7E` | `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | Arg1 exception code; a system thread faulted | `!analyze -v`, `!thread`, `kv` |
| `0x8E` | `KERNEL_MODE_EXCEPTION_NOT_HANDLED` | Arg1 exception code; no handler anywhere | `!analyze -v`, `kv` |
| `0x50` | `PAGE_FAULT_IN_NONPAGED_AREA` | Arg1 referenced address, arg2 `0` read/`1` write; often freed pool | `!pool <arg1>`, `!analyze -v` |
| `0xA` | `IRQL_NOT_LESS_OR_EQUAL` | Arg1 address, arg2 IRQL, arg3 read/write, arg4 instruction | `!analyze -v`, `!irql`, `kv` |
| `0xD1` | `DRIVER_IRQL_NOT_LESS_OR_EQUAL` | Same four arguments, attributed to a driver | `!analyze -v`, `lmvm <module>`, `kv` |
| `0xC1` | `SPECIAL_POOL_DETECTED_MEMORY_CORRUPTION` | Arg1 address; overrun or underrun of a special-pool block | `!analyze -v`, `!pool <arg1>` |
| `0xC2` | `BAD_POOL_CALLER` | Arg1 violation subcode, arg2 address, arg3 tag; double free or bad free | `!analyze -v`, `!pool <arg2>`, `!poolused 2` |
| `0xC5` | `DRIVER_CORRUPTED_EXPOOL` | Arg1 address, arg2 IRQL, arg3 read/write; paged pool at high IRQL | `!analyze -v`, `!pool <arg1>` |
| `0x9F` | `DRIVER_POWER_STATE_FAILURE` | Arg1 subcode, arg2 PDO, arg3 triage block, arg4 blocked IRP | `!irp <arg4>`, `!devstack <arg2>`, `!poaction` |
| `0xCE` | `DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | Arg1 address, arg2 IRQL; code ran after unload | `!analyze -v`, `lmvm <module>`, `!irpfind` |
| `0xD6` | `DRIVER_PAGE_FAULT_BEYOND_END_OF_ALLOCATION` | Arg1 address past the end of an allocation | `!analyze -v`, `!pool <arg1>` |
| `0x1A` | `MEMORY_MANAGEMENT` | Arg1 subcode; PTE or page-frame corruption | `!analyze -v`, `!vm`, `!pte` |
| `0x139` | `KERNEL_SECURITY_CHECK_FAILURE` | Arg1 class; `3` is a corrupted `LIST_ENTRY` | `!analyze -v`, `kv`, `!pool` the link address |
| `0x13A` | `KERNEL_MODE_HEAP_CORRUPTION` | Heap block corruption | `!analyze -v`, `!heap -p`, `!pool` |
| `0x133` | `DPC_WATCHDOG_VIOLATION` | Arg1 `0` single DPC, arg2 count, arg3 allotment; arg1 `1` cumulative | `!analyze -v`, `!dpcs`, `!running -t` |
| `0x101` | `CLOCK_WATCHDOG_TIMEOUT` | Arg3 PRCB of the stuck processor, arg4 its index | `~<arg4>s`, `kv`, `!irql`, `!prcb <arg4>` |
| `0xEF` | `CRITICAL_PROCESS_DIED` | A process the system cannot live without terminated | `!analyze -v`, `!process`, event log |
| `0x109` | `CRITICAL_STRUCTURE_CORRUPTION` | Kernel code or critical data was modified | `!analyze -v`, then identify the named region and the most recent driver loads |
| `0x7F` | `UNEXPECTED_KERNEL_MODE_TRAP` | Arg1 trap number; `8` is a double fault, usually stack exhaustion | `!analyze -v`, `kv`, `!thread` |
| `0xE2` | `MANUALLY_INITIATED_CRASH` | Deliberate crash from the keyboard or `.crash` | Expected during capture; not a bug to fix |

## Step-by-step

1. **Establish what you have.** Confirm the file exists with a fresh timestamp (`%SystemRoot%\MEMORY.DMP`, `%SystemRoot%\Minidump\*.dmp`), and read event 1001 for the code and parameters. Check: is the file newer than the failure, and is it the type you need? A minidump cannot answer a pool question.
2. **Open it with symbols already configured.** Launch with `windbg -z <file> -y "srv*C:\Symbols*https://msdl.microsoft.com/download/symbols"`, or run `.symfix` and `.reload`. Check: `lmvm nt` resolves with a full path and a build string; unresolved `nt` makes everything else fiction.
3. **Run `.bugcheck` and `!analyze -v`.** Read the output in this order:
    - **Code and description** — the class of fault, before any speculation.
    - **The four arguments** — their documented meaning from the stop-code table.
    - **`CURRENT_IRQL`** — whether this path was allowed to page, block, or wait.
    - **`FAULTING_IP`** — the faulting instruction; arg4 usually repeats it for `0xA` and `0xD1`.
    - **`PROCESS_NAME`** — the running process, which for a DPC is only whatever thread was interrupted.
    - **`STACK_TEXT`** — where the code was and which context it ran in.
    - **`SYMBOL_NAME` / `MODULE_NAME` / `IMAGE_NAME`** — the debugger's best guess, not a verdict.
    - **`DUMP_TYPE`** — what evidence the file can support.

    Check: do the arguments agree with the stack? A `0xD1` whose arg2 is `2` must have a `DISPATCH_LEVEL` path on the stack.
4. **Verify the blamed module against your build.** `lmvm <module>` and compare the timestamp, image size, and PDB with the binary in your output directory. Check: a mismatch means the symbols are wrong and `!analyze` is guessing.
5. **Walk the stack outward from the fault.** `kv` for frame numbers and arguments, `!thread` for the current thread, `!running -t` for every processor, `~<n>s` to switch. Check: is the fault in a dispatch routine, a DPC, a completion routine, a worker thread, or an arbitrary thread?
6. **Test the memory story.** For a bad address: `!pool <addr>` for the tag and state, `!poolused 2` for a tag that is growing, `!vm` for overall pressure. For a stuck request: `!irp <addr>`, `!devobj`, `!drvobj <driver> 2`, `!devstack`. Check: freed, paged out, overrun, or never valid — these are four different bugs with four different fixes.
7. **Write the causal chain in one paragraph.** It must explain the code, all four arguments, the IRQL, the stack, and why the memory was in that state. If the paragraph needs "somehow", the analysis is not finished.
8. **Fix the cause, not the symptom, then re-verify** by rerunning the same reproducer with the verifier flags that would have caught it (see Verification).

### Hang triage

1. **Capture before you touch anything.** A hung machine holds the only copy of the evidence. Use the debugger, the keyboard crash, an NMI, or a live dump (see the next section); do not power it off.
2. **Run `!analyze -hang` first.** It looks for threads blocked on locks whose owners are not running, and prints the lock and the blocker when it can identify them.
3. **Group the threads by blocker.** `!stacks 2` lists every thread with the kernel routine it is blocked in. Many threads in one routine, or one routine repeated as the blocker, points at a single resource.
4. **Find the owner.** `!locks` shows `ERESOURCE` owners and waiters; `!cs -l` lists critical sections; `!deadlock` works when Driver Verifier deadlock detection was enabled. Then `!thread <owner>` and ask what *that* thread is waiting for.
5. **Close the cycle.** Follow owner to waiter until a thread repeats; that cycle is the deadlock. If there is no cycle, the holder is simply not being scheduled: check its priority, its processor affinity, and whether a processor is spinning at high IRQL (`!irql` on each).
6. **Classify the hang.** One lock held forever (`!locks`), a lock-order inversion between two drivers, a DPC that never returns (`0x133`), or a processor that never takes the clock (`0x101`).
7. **For `0x101`, start from the named processor.** Arg4 is its index and arg3 its PRCB: switch with `~<arg4>s`, read the stack with `kv`, and run `!irql` to see the level it is stuck at. A processor that never takes the clock is spinning with interrupts masked, not waiting on a lock, so lock commands will show nothing.

| Observation | Reading | Next command |
|---|---|---|
| Many threads blocked in the same routine, one owner | A single lock or resource serializes everything | `!locks`, `!thread <owner>` |
| Two threads each waiting on a lock the other owns | Lock-order inversion | `!locks`, `!cs -l`, `!deadlock` |
| The owner is runnable but never scheduled | Priority or affinity starvation | `!thread <owner>`, `!running` |
| Every processor at `DISPATCH_LEVEL` with one driver repeated | A lock held far too long, or a spin | `!irql`, `kv` on each processor |
| One processor never takes the clock interrupt (`0x101`) | A loop with interrupts masked | `~<n>s`, `kv`, `!prcb <n>` |
| A DPC ran past its allotment (`0x133`) | Unbounded work at `DISPATCH_LEVEL` | `!dpcs`, `kv` |

### Reading a stop screen with no dump

- The screen shows the code, the four parameters, and a `What failed:` image name. That name is where the instruction was; for corruption it is often the victim, not the culprit.
- Event 1001 repeats the same text, so `Get-WinEvent` recovers it after the reboot.
- `!analyze -show <code> <p1> <p2> <p3> <p4>` in any kernel session prints the same interpretation `!analyze -v` would print.
- Without a dump there is no stack, no pool, and no other processor, so the screen selects what to capture next; it does not diagnose anything by itself.

## Complete example

The session below is a representative walkthrough, abbreviated and with addresses from one run; the reasoning, not the hex, is the reusable part. A server VM rebooted overnight and event 1001 reported `0xD1`.

```text
kd> .symfix
kd> .reload
kd> .bugcheck
Bugcheck code 000000D1
Arguments ffffb10a`3c2e8a40 00000000`00000002 00000000`00000000 fffff805`11a61530

kd> !analyze -v
*******************************************************************************
*                        Bugcheck Analysis                                    *
*******************************************************************************
DRIVER_IRQL_NOT_LESS_OR_EQUAL (d1)
An attempt was made to access a pageable (or completely invalid) address at an
interrupt request level (IRQL) that is too high.
Arguments:
Arg1: ffffb10a3c2e8a40, memory referenced
Arg2: 0000000000000002, IRQL
Arg3: 0000000000000000, value 0 = read operation, 1 = write operation
Arg4: fffff80511a61530, address which referenced memory

CURRENT_IRQL:  2
FAULTING_IP:
sensord+1530
fffff805`11a61530 8b03            mov     eax,dword ptr [rbx]
IMAGE_NAME:  sensord.sys
PROCESS_NAME:  System
DUMP_TYPE:  2

kd> lmvm sensord
Browse full module list
start             end                 module name
fffff805`11a50000 fffff805`11a6a000   sensord    (private pdb symbols)
    Loaded symbol image file: sensord.sys
    Image path: \SystemRoot\System32\drivers\sensord.sys
    Image name: sensord.sys
    Timestamp:        Thu Nov  6 09:14:22 2025
    CheckSum:         0001B7A4
    ImageSize:        0001A000

kd> !pool ffffb10a3c2e8a40
 ffffb10a3c2e8a40 size:  110 previous size:    0  (Allocated)  Snsr

kd> kn
 # Child-SP          RetAddr           Call Site
00 fffff805`0e4b7980 fffff805`11a61e2d sensord+0x1530
01 fffff805`0e4b79b0 fffff805`0fcb0f88 sensord!SensorDpc+0x5d
02 fffff805`0e4b7a00 fffff805`0fcb2a10 nt!KiExecuteDpc+0x1f8
03 fffff805`0e4b7b00 fffff805`0fcab3da nt!KiRetireDpcList+0x4aa
```

Reading it in order:

- **Arg2 = 2** fixes the context: this is `DISPATCH_LEVEL`, so it is a DPC, not a thread. The stack confirms it: `SensorDpc` sits directly on the DPC dispatcher with no thread frames below it.
- **Arg3 = 0** means a read. `FAULTING_IP` is `sensord+0x1530`, a `mov eax,[rbx]`; `rbx` holds the referenced address `ffffb10a3c2e8a40`.
- **The address is paged pool** (`!pool` reports a `Paged pool` region, tag `Snsr`, state `Allocated`; the region line is abbreviated above). It is not freed and not corrupt: the allocation is intact. The fault exists because the memory is not resident and the memory manager may not fault it in at `DISPATCH_LEVEL`.
- **`lmvm` matches the shipped build**, so the source line is meaningful and symbols are trustworthy.
- **`DUMP_TYPE: 2`** means a kernel dump was captured, which is why the pool question could be answered at all; a minidump would have shown the stack and nothing about the memory.

Root cause, stated as a chain: `SensorDpc` calls the driver's logging helper, which formats `ext->DeviceName`; `DeviceName.Buffer` was allocated from paged pool in `DriverEntry` and is only read when a sample completes. Under memory pressure the allocation was paged out, and the first DPC to touch it after that faulted at `DISPATCH_LEVEL`. The defect is not the faulting instruction — it is the allocation site, which chose paged pool for data reachable from a DPC. The bug was latent for weeks because the buffer normally stayed resident.

Fix: make the name a fixed `WCHAR DeviceName[64]` inside the non-paged device extension (or allocate it with `POOL_FLAG_NON_PAGED`), remove the formatting call from the DPC entirely and log from a work item, and add `NT_ASSERT(KeGetCurrentIrql() <= DISPATCH_LEVEL)` at the top of any helper that may touch paged memory.

The second failure did bugcheck, but only because the watchdog caught it. `sensord` had a DPC that waited for a device that had stopped answering, and a DPC that runs too long is exactly what `0x133` exists to interrupt:

```text
kd> !analyze -v
DPC_WATCHDOG_VIOLATION (133)
The DPC watchdog detected a prolonged run time at an IRQL of DISPATCH_LEVEL
or above.
Arguments:
Arg1: 0000000000000000, A single DPC or ISR exceeded its time allotment.
Arg2: 0000000000000501, The DPC time count (in ticks).
Arg3: 0000000000000500, The DPC time allotment (in ticks).
Arg4: 0000000000000000
CURRENT_IRQL:  2
IMAGE_NAME:  sensord.sys

kd> kn
 # Child-SP          RetAddr           Call Site
00 fffff805`0e5b7a40 fffff805`11a61f02 sensord!SensorWaitReady+0x22
01 fffff805`0e5b7a70 fffff805`0fcb0f88 sensord!SensorDpc+0x71
02 fffff805`0e5b7aa8 fffff805`0fcb2a10 nt!KiExecuteDpc+0x1f8
03 fffff805`0e5b7b00 fffff805`0fcab3da nt!KiRetireDpcList+0x4aa

kd> !dpcs
CPU Type      KDPC       Function
 0: Normal  : 0xfffff8050feb4290 0xfffff8050fc15630 nt!PpmPerfAction
 2: Threaded: 0xfffff8050feb43d8 0xfffff8050fcb7d70 nt!KiDpcWatchdog
```

Interpretation: arg1 `0` means one DPC ran past its allotment (`0x501` ticks against `0x500`), so the watchdog fired on that single DPC. `kv` puts the loop in `SensorWaitReady`, called from the DPC; that helper spins on a status register waiting for a device that had stopped answering. The watchdog is why a dump exists at all — without it the machine would simply stop responding, with no stop code and no file. The device fault is real, but the driver turned it into a system hang. Fix: bound the wait, never stall for more than a few microseconds at `DISPATCH_LEVEL`, retry through a timer or move the wait to a system thread, and return the DPC immediately when the device is not ready.

The third case is a power transition that never finished, shown here abridged. The machine neither hung nor crashed; it failed a sleep transition and bugchecked `0x9F` when the power manager's timeout expired.

```text
kd> !analyze -v
DRIVER_POWER_STATE_FAILURE (9f)
A driver has failed to complete a power IRP within a specific time.
Arguments:
Arg1: 0000000000000003, A device object has been blocking an Irp for too long a time
Arg2: ffffd20b1b3c5060, Physical Device Object of the stack
Arg3: ffffd20b1c2a5b40, nt!_TRIAGE_9F_POWER
Arg4: ffffd20b1c2a5010, The blocked IRP

kd> !irp ffffd20b1c2a5010
Irp is active with 4 stacks 3 is current (= 0xffffd20b1c2a5130)
 No Mdl: No System Buffer: Thread ffff94820a3c4080:  Irp stack trace.
     cmd  flg cl Device   File     Completion-Context
 [IRP_MJ_POWER(16), IRP_MN_SET_POWER(2)]
            0  1 ffffd20b1b3c8050 00000000 00000000-00000000    pending
	       \Driver\sensord
			Args: 00000001 00000001 00000003 00000000

kd> !devstack ffffd20b1b3c5060
  !DevObj           !DrvObj            !DevExt           ObjectName
  ffffd20b1b3c8050  \Driver\sensord    ffffd20b1b3c81a0
> ffffd20b1b3c5060  \Driver\ACPI       ffffd20b1b3c51b0
```

Interpretation: arg1 `3` means a device object held an IRP past the timeout, and arg4 is that IRP. `!irp` shows a pending `IRP_MN_SET_POWER` owned by `\Driver\sensord`, and `!devstack` places that device directly above the ACPI PDO, so the pending IRP is the driver's own power dispatch. That dispatch marked the IRP pending and deferred completion to the sampling DPC, which only runs when samples are being requested; at the moment of the sleep transition the queue was empty, so nothing completed it. Fix: complete `IRP_MN_SET_POWER` in the power dispatch after programming the device, and never tie a power IRP's completion to device I/O activity.

The fourth case never bugchecked. The machine stopped responding with no stop code and no dump; a kernel debugger was already attached, so the state was captured live and abridged to the client threads:

```text
kd> .dump /f C:\dumps\hang-01.dmp
kd> .reload
kd> !stacks 2
Proc.Thread  .Thread  Ticks  ThreadState  Blocker
[ffff94820a3c8000 client.exe]
1a2c.0012a0 ffff94820b1c4080 0000a2c Blocked nt!KeWaitForSingleObject+0x1a2
1a2c.0012a4 ffff94820b1c5080 0000a2c Blocked nt!KeWaitForSingleObject+0x1a2
1a2c.0012a8 ffff94820b1c6080 0000a2c Blocked nt!KeWaitForSingleObject+0x1a2
```

Interpretation: three client threads share one blocker, which is the signature of a single resource. `!thread ffff94820b1c4080` shows the wait inside `sensord!AcquireSampleLock`; `!locks` reports that resource owned by the worker thread `ffff94820b0c3080`; and that thread's own stack is in `IoCallDriver`, waiting on a lower filter that waits on the same resource. That is a lock-order inversion between two drivers, and no amount of waiting resolves it. Fix: impose one acquisition order on the two locks (the lower filter's lock before the driver's), or stop holding the driver's lock across the call down the stack — release it before `IoCallDriver` and reacquire afterwards.

## Variations and advanced use

### Capturing a dump from a hung VM

Pick the method from what still works, not from what is convenient:

1. **Kernel debugger already attached (best).** Break in, then `.dump /f C:\dumps\hang.dmp` for a complete dump, `.dump /ka` for active kernel plus user memory, or `.dump /m` for a small one. `.dump` writes on the target without crashing it, so the VM can be resumed with `g` afterwards. Confirm the directory exists inside the guest.
2. **No debugger, keyboard still alive, configured in advance.** Set `CrashOnCtrlScroll` (`REG_DWORD` = `1`) under the `Parameters` subkey of the keyboard service — `i8042prt` for PS/2, `kbdhid` for USB, `hyperkbd` for a Hyper-V keyboard — then reboot. Hold the **rightmost** `Ctrl` and press `Scroll Lock` twice: the kernel bugs with `0xE2` and writes the configured dump on the next boot. Limitation: it fails if the machine is stuck at high IRQL or the input stack is dead.
3. **NMI.** An NMI reaches the processor even with interrupts masked, which is exactly the hang where the keyboard path is dead. Inject it from the hypervisor (QEMU performs it with the monitor `nmi` command; other hypervisors expose an NMI control on some builds) or press the NMI button on server hardware; Windows converts it to `0x80` (`NMI_HARDWARE_FAILURE`) and writes the configured dump. It needs a dump to be configured beforehand.
4. **Sysinternals NotMyFault.** When the guest still runs well enough to launch a process, load `myfault.sys` and press Crash with the default crash type; it produces a `0xD1` at `DISPATCH_LEVEL` and therefore a dump. Use it to prove capture works, not to diagnose a real bug.
5. **Live dump.** Sysinternals `livekd -o C:\dumps\live.dmp` captures kernel state from a running machine without a bugcheck (Windows 8.1+ uses the live-dump API). Best for near-hangs where a crash would lose the evidence.
6. **Nothing works.** A machine powered off while hung yields no dump: the image only exists once the kernel has written it. Take hypervisor snapshots during a stress run, or keep a debugger attached, so the next hang is capturable.

### Minidumps, kernel dumps, and what each cannot answer

A minidump answers "which module was executing and what was the exception" and nothing about pool, other processors, or user memory. A kernel dump adds pool, every processor's stack, and driver state, but no user memory, so `!process` cannot show user stacks and there is no user heap or image data to inspect. A complete dump adds user memory at the cost of RAM-sized files. Choose before reproducing: capturing a minidump for a corruption bug wastes the reproduction.

### Making the captured image trustworthy

- Keep `Overwrite = 1`; with `0`, a second crash leaves the first dump in place and writes nothing, which looks exactly like "the bug stopped happening".
- Keep the paging file on the boot volume and large enough; with BitLocker, `DumpFilters` must include `dumpfve.sys`.
- Third-party storage filter and encryption drivers that are not dump-aware can prevent the write; the dump stack bypasses the normal stack and loads only the miniport plus registered filters.
- A crash before the dump stack is initialized, or while the disk miniport itself is failing, produces no file.
- Copy the dump off the machine before rebooting again; `MEMORY.DMP` is overwritten by the next crash.

### Choosing verifier flags before reproducing

The verifier turns a class of bug into a bugcheck at the moment it happens, which is worth far more than a dump taken after the damage spread. Enable the flags that match the suspect, reboot, reproduce, then read `!verifier 3 <driver>`.

| Suspected bug | Flags | What it catches |
|---|---|---|
| Overrun, underrun, use-after-free | `0x1` special pool | Any access outside the allocation boundary |
| Leak or wrong pool tag | `0x2` pool tracking | Blocks still allocated when the driver unloads |
| Paged access or a wait at high IRQL | `0x4` IRQL checking | `0xC4` at the violation instead of `0xD1` later |
| Deadlock between threads | `0x10` deadlock detection | Lock cycles, and enables `!deadlock` |
| Handle or security misuse | `0x80` security checks | Invalid handle operations and access checks |

```text
verifier /flags 0x1 /driver sensord.sys
verifier /querysettings
verifier /reset
```

### Batch triage

The first pass is the same on every dump, so script it and keep the log:

```text
$$ triage.txt - first-pass facts for any kernel dump
$$ windbg -z C:\dumps\crash.dmp -c "$$<C:\dbg\triage.txt"
.logopen /t C:\dbg\triage.log
.symfix
.reload
.bugcheck
!analyze -v
lm
!running -t
!stacks 2
.logclose
```

The log then holds the reason, the module list with timestamps, every processor's stack, and the blocker for every thread. Read that before running anything else.

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `MEMORY.DMP` has an old timestamp after a new crash | `Overwrite = 0`, or the paging file is too small | `Get-ItemProperty 'HKLM:\SYSTEM\CurrentControlSet\Control\CrashControl'`; set `Overwrite = 1` and enlarge the paging file |
| No dump at all, no event 1001 | Capture disabled, or the crash happened before the dump stack loaded | `bcdedit /enum {current}`; check `CrashDumpEnabled`; enable `crashdump on` |
| Dump loads but `nt` symbols are `export` only or unresolved | Symbol path wrong or offline | `.symfix`, `!sym noisy`, `.reload -f` |
| `WARNING: Unable to verify timestamp for <module>` | The PDB does not match the binary in the dump | Put the exact `.sys` and `.pdb` on the path, `.reload -f <module>` |
| `Probably caused by ntoskrnl` with no driver on the stack | Corruption happened earlier; the kernel is the victim | `!poolused 2`, special pool, `!verifier 3`, look for a driver that wrote out of bounds |
| `!pool` fails with "not a valid pool address" | Minidump, or the address is not pool | Check `DUMP_TYPE`; capture a kernel dump and reproduce |
| `0xA`/`0xD1` with an address that looks valid | Freed or paged-out memory | `!pool <addr>`; if paged, move the allocation to non-paged |
| `!irp` shows the IRP pending forever (`0x9F`) | A driver below never completes it | `!irp <arg4>`, `!devstack`, then the owning driver's power path |
| Hang with no bugcheck and no dump | Nobody forced the crash | `.dump /f`, keyboard crash, NMI, or `livekd` before resetting |
| `!analyze -hang` reports nothing | The hang is at high IRQL, not in a wait | `!running -t`, `!irql` per processor, `!dpcs` |
| New dump every boot | A driver fails during boot and the machine auto-reboots | Disable auto-reboot, then `!analyze -v` on the first dump |
| Verifier is enabled on a production machine | Left over from a test | `verifier /reset`, then reboot |

## Safety, IRQL, and cleanup checklist

- [ ] Dump capture is configured **before** the stress run, and the paging file is on the boot volume with enough room.
- [ ] `Overwrite = 1` and `AutoReboot = 0` while actively debugging, so the stop screen and dump survive.
- [ ] The analysis machine holds the exact `.sys` and `.pdb` for every build under test, keyed by timestamp.
- [ ] Dumps are treated as sensitive: a complete dump contains user memory, credentials, and keys. Delete them after analysis.
- [ ] A hung machine is never powered off before a dump is captured; take a VM snapshot first.
- [ ] `verifier /reset` is run after any verifier session, and the reboot is confirmed.
- [ ] `crashdump` is not left disabled on a test VM.
- [ ] Every claim in the causal chain is backed by one command's output, not by the `Probably caused by` line alone.
- [ ] The fix names the allocation, locking, or lifetime decision that was wrong, not just the faulting instruction.
- [ ] The rebuilt binary's timestamp and hash match what is installed before declaring the fix verified.

## Verification

1. **Configuration.** `Get-CimInstance Win32_OSRecoveryConfiguration` reports `DebugInfoType` `2` or `7` and a `DebugFilePath` on a volume with room; `bcdedit /enum {current}` shows `crashdump On`.
2. **Capture works.** On a scratch VM, force a crash (NotMyFault default, or the keyboard sequence) and confirm a new file with a fresh timestamp plus event 1001. Load it and confirm `!analyze -v` resolves `nt` and the faulting module.
3. **Symbols are right.** `lmvm <module>` shows the timestamp, image size, and PDB path of the build under test; `!sym noisy` shows no failed lookups for that module.
4. **The fix is in the running binary.** Compare `Get-FileHash` of the installed `.sys` with the build output, and confirm `lmvm` in a fresh dump matches.
5. **The reproducer no longer fails.** Re-run the same stress with verifier flags that would catch the class of bug — `verifier /flags 0x1` special pool, `0x2` pool tracking, `0x4` IRQL checking, `0x10` deadlock detection — for the same duration that produced the original dump. Expected: no bugcheck, and `!verifier 3 <driver>` clean after the run.
6. **No silent recurrence.** `Get-WinEvent` for provider `Microsoft-Windows-WER-SystemErrorReporting` shows no new event 1001 with the same code, and `%SystemRoot%\Minidump` gains no file with that code.
7. **The specific invariant holds.** For a paged-allocation-at-DISPATCH bug, the checked build's `NT_ASSERT` never fires and `!poolused 2` shows the tag's allocation in non-paged pool; for a hang, a forced capture of the same workload shows no thread blocked in the driver's wait routine.
