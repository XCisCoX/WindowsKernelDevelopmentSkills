---
name: wk-windbg-fundamentals
description: Use when you must drive WinDbg, cdb, or kd before any live user-mode or kernel session: installing the Debugging Tools, setting _NT_SYMBOL_PATH and .symfix, reading lm and lmvm, setting bp, ba and conditional breakpoints, stepping with p, t, gu, pc and ph, catching exceptions with sxe and .exr, reading !analyze -v, writing .dump /ma files, and scripting with .scriptload and .logopen.
---

# WinDbg fundamentals: from installation to a stopped target

A debugger session is where a crashing program tells the truth, in the vocabulary of its
commands: modules, symbols, types, frames, registers, and breakpoints. This skill builds the
fluency to stand in front of a live kernel without wasting the session: installing the
debugger, proving that symbols are real, and reading memory, types, stacks, and exceptions
fast. Every kernel technique depends on those moves; `!process` and `!irp` stay unreadable
until `lm`, `dt`, and `kb` are reflexes.

The rule that prevents most wasted sessions: **never interpret a stack, a variable, or a
local before `lm` proves the module's symbols are loaded.** An export-symbol stack is
fiction, and a root cause drawn from it is worse than no answer.

## When to use

- You are about to attach a debugger to a process, a driver, or a dump and need the exact
  commands to get from "attached" to "I know the faulting line and the bad pointer".
- You are installing or repairing a debugging environment: the Debugging Tools package, the
  Store build of WinDbg, cdb on a server, or a debugger copied into an offline VM.
- Symbols are missing, mismatched, or slow, and `k` prints raw addresses.
- You must set a breakpoint that fires only for one value, one call site, or one access.
- You must capture evidence — a transcript with `.logopen`, a dump with `.dump /ma` — or
  rehearse the command set once, end to end, before a kernel session.

Do not use for: kernel transports and boot configuration (`bcdedit /debug`, net or serial
pipes) — that is `wk-kernel-debugging`. Stop-code triage is `wk-crash-dump-analysis`;
DbgPrint, WPP, and ETW are `wk-kernel-tracing`; build outputs are `wk-driver-build-deploy`.

## Prerequisites and related skills

- **Host**: Windows 10/11 or Server 2016+, x64 or ARM64, with the matching debugger build;
  admin rights for kernel attach and for attaching to elevated processes.
- **Target**: any running process, or a dump file. For kernel mode, a target booted with
  debugging enabled — that setup belongs to `wk-kernel-debugging`.
- **Binaries**: private symbols (`/Zi` for MSVC, `-g` for clang), the PDB next to the binary
  or on the symbol path, and the *same* build that is running. A PDB from an older build is
  worse than no PDB.
- **Network**: HTTPS egress to `https://msdl.microsoft.com/download/symbols`, or a
  pre-populated local store when egress is blocked.
- **Context/IRQL**: none for reading symbols or memory; in kernel mode every break freezes
  the machine, so read the safety checklist before attaching to anything that serves
  traffic.
- Related: `wk-kernel-debugging` (host/target setup, driver stepping), `wk-crash-dump-analysis`
  (bugchecks and `!analyze -v` in anger), `wk-irql` (what the numbers in a stack mean),
  `wk-driver-build-deploy` (build outputs and PDB placement), `wk-kernel-programming-rules`
  (why a bad pointer in kernel is a bugcheck, not an exception).

## Mental model

### One engine, several front ends

Every debugger in the package is a front end over the same engine, `dbgeng.dll`, and the
same extension DLLs. `windbg.exe` adds windows; `cdb.exe` and `ntsd.exe` are console
user-mode front ends; `kd.exe` is the console kernel front end. The commands are identical,
so a session recorded in cdb replays in WinDbg and vice versa.

```
+------------------+  transport: local, serial, network, USB, dump file  +-------------+
|    front end     | --------------------------------------------------> |   target    |
| windbg / cdb / kd| <-------------------------------------------------- | proc or OS  |
+--------+---------+    events, memory reads, symbol requests            +-------------+
         v
   dbgeng.dll -> extension DLLs: ext.dll, kdexts.dll, ntsdexts.dll, winext\*.dll
```

Consequences that matter:

- Memory is read lazily (`k` loads symbols, `dt` loads types) and extension commands live in
  DLLs, so "command not found" is a missing `.load`. A dump is the same interface over a file.

### Three command classes

| Class | Looks like | Operates on | Examples |
| --- | --- | --- | --- |
| Intrinsic | `k`, `bp`, `d`, `r` | the target | control flow, memory, breakpoints |
| Meta | leading `.` | the debugger session | `.symfix`, `.reload`, `.dump`, `.logopen` |
| Extension | leading `!` | the target, via a DLL | `!analyze`, `!process`, `!address`, `!heap` |

The class tells you what a failure means: `.` commands change your environment, `!` commands
can be missing or versioned, and bare commands always exist.

### Break state is the only state you can inspect

A debugger alternates between two states. **Running** means the target executes and the
debugger refuses commands ("Debuggee is running"). **Broken** means every thread (user
mode) or every processor (kernel mode) is frozen at an instruction boundary, and only then
are memory and stacks coherent.

- In user mode a break suspends all threads of the process. Other processes keep running.
- In kernel mode a break freezes the whole machine: all processors, all drivers, all
  interrupts below the debugger's own.
- `g` resumes; `Ctrl+Break` or the Break button interrupts a running target and lands in
  `ntdll!DbgBreakPoint` (user) or the kernel debugger's break routine.

Everything you read is a snapshot of one instant. When a value looks impossible, check that
you are still in the same break: a `g` and a re-break invalidate every address you wrote down.

### The same commands mean different things in the two modes

| Aspect | User-mode session | Kernel-mode session |
| --- | --- | --- |
| Prompt | `0:003>` = processor 0, thread 3 | `0: kd>` = processor 0 |
| `~` enumerates | threads of the process | processors of the machine |
| Context switch | `~3s` then `k` | `~1s` for a processor; `.thread` / `.process` for a context |
| Scope of `bp` | that process only, code bytes patched | whole machine, any context |
| Freeze | the process's threads | every processor |
| Symbols | EXE and DLLs from `lm` | `nt` (the kernel) and drivers; user symbols need `.reload /user` |
| First useful commands | `lm`, `bp`, `g` | `lm`, `!process 0 0`, `sxe ld:drv.sys` |

The most common beginner error in kernel work is typing `~` and expecting threads: in a
kernel session it lists processors, and threads come from `.thread` and `.process`.

### Windows are views over commands

Every window renders what a command prints: Locals is `dv`, Registers is `r`, Call Stack is
`k`, Disassembly is `u`, Memory is the `d` family, Breakpoints is `bl`; `F5`/`F10`/`F11` are
`g`/`p`/`t`, `Shift+F11` is `gu`, `Ctrl+Break` breaks in. The Command window stays the
authority, and a breakpoint typed in a dialog cannot carry a command string.

## Installation variants and launch lines

The package writes nothing to the registry: it is a directory of executables and DLLs, so it
can be copied into a VM, onto a USB stick, or next to a crash dump. Install it from the
Windows SDK setup (Debugging Tools for Windows) or the WDK, or use the Store build.

| Variant | Executable | Interface | Choose it when |
| --- | --- | --- | --- |
| Classic WinDbg | `windbg.exe` | GUI | you want the package with no Store dependency, or an offline VM copy |
| WinDbg (Store) / WinDbg Preview | `WinDbgX.exe` | GUI | current UI, auto-update, same engine and same commands |
| cdb | `cdb.exe` | console, user mode | scripted sessions, servers without a GUI, output you can paste |
| ntsd | `ntsd.exe` | console, user mode | legacy scripts; identical to cdb but always opens a new console |
| kd | `kd.exe` | console, kernel | kernel targets over serial or network with no GUI |
| Debugging Tools package | the folder itself | all of the above | the base install; also ships `gflags.exe`, `dumpchk.exe`, `kill.exe` |

Default location on a 64-bit host: `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64\`
with siblings `x86\` and `arm64\`. **Use the debugger that matches the target, not the
host's default shell**: `x64\cdb.exe` for 64-bit targets, `x86\cdb.exe` only for 32-bit
targets.

```bat
:: already-running process, by PID or by name
windbg -p 7412
windbg -pn notepad.exe

:: launch under the debugger, break at the first instruction
cdb crashlab.exe
cdb -c "sxe av; g" crashlab.exe

:: a dump, and a kernel target over the network
windbg -z C:\dbg\crashlab.dmp
windbg -kl
kd -k net:port=50000,key=1.2.3.4
```

`-g` skips the initial breakpoint and `-G` the final one; both suit batch runs and both hide
the moment when you could still have set a filter. For diagnosis, start without them.

## Symbol configuration

Symbols are a lookup from (module, timestamp, image size, PDB signature) to a PDB file. Set
the path once, in the environment, and every tool on the machine benefits.

```bat
setx _NT_SYMBOL_PATH "srv*C:\symbols*https://msdl.microsoft.com/download/symbols"
```

The string has three parts separated by `*`: `srv` means "symbol server protocol", the
middle part is the local cache (created on demand, safe to put on a share), and the last part
is the upstream store. After `setx`, open a **new** shell; existing processes keep the old
environment.

Inside a session the same settings are reachable, and they are what you use when you cannot
change the machine's environment:

```
0:000> .symfix C:\symbols
0:000> .sympath
Symbol search path is: srv*
Expanded Symbol search path is: cache*c:\symbols;SRV*https://msdl.microsoft.com/download/symbols
0:000> .reload /f
```

- `.symfix` replaces the symbol path with a server path; `.symfix+` appends to what is
  already configured, which is how a private store gets searched first.
- The optional argument is the local cache directory; without it the debugger uses the `sym`
  subdirectory of its own installation.
- `.sympath+ C:\build\out\x64\Debug` puts your own build output ahead of the server, which is
  how you get private symbols for code you just compiled.
- `.reload` re-reads symbols for modules that still need them; `.reload /f` forces every
  module and `.reload /f name.exe` forces one. `.reload /user` loads user-mode symbols in a
  kernel session for the process selected with `.process /p`.
- `!sym noisy` prints every lookup — path tried, HTTP request, PDB found — and `!sym quiet`
  restores normal output. Turn it on whenever a module refuses to resolve.

### What symbol status looks like, and what it means

```
0:000> lm
start             end                 module name
00007ff6`9a2c0000 00007ff6`9a2c9000   crashlab   (private pdb symbols)  C:\lab\crashlab.pdb
00007ffb`1c1a0000 00007ffb`1c3b2000   KERNEL32   (deferred)
00007ffb`37d80000 00007ffb`37f2c000   ntdll      (pdb symbols)  C:\symbols\ntdll.pdb\3B92DED9\ntdll.pdb
```

| Status | Meaning | Action |
| --- | --- | --- |
| `deferred` | not needed yet; will load on first use | none — run `k` and re-check |
| `pdb symbols` | public symbols loaded | usable: names, globals, no locals |
| `private pdb symbols` | full symbols, locals and private types | what your own `/Zi` build must show |
| `export symbols` | only the export table; names are guesses | fix the symbol path, then `.reload /f` |
| `no symbols` | nothing found, not even exports | same; `!sym noisy` to see why |
| `Mismatched PDB` | the PDB belongs to another build | rebuild, or fetch the build's own PDB |

Every module you examine must show `pdb symbols` or `private pdb symbols`; a stack whose
frames all carry four-hex-digit offsets (`ntdll!RtlAllocateHeap+0x1a3c`) is export fiction.

### Common symbol errors and their causes

| Message | Real cause | Fix |
| --- | --- | --- |
| `*** ERROR: Symbol file could not be found. Defaulted to export symbols for ntdll.dll` | no path entry holds the PDB, or the server is unreachable | `.symfix`, `.reload /f ntdll`, `!sym noisy` |
| `*** ERROR: Module load completed but symbols could not be loaded for mydrv.sys` | driver PDB not on the path; debug directory stripped | `.sympath+` the build output, `.reload /f mydrv.sys` |
| `*** WARNING: Unable to verify checksum for mydrv.sys` | binary and PDB are from different builds | rebuild both, copy the PDB beside the driver |
| `Mismatched PDB` or a signature error | stale PDB with the same name | delete the cached copy, rebuild, re-`reload` |
| HTTP 403 / 404 lines under `!sym noisy` | egress filtered, or the module is not on the public server | allow the host, or use a private store |
| Symbols load but locals say `<unavailable>` | optimized build, or the PDB is for another build | rebuild `/Od /Zi`, confirm `lmvm` timestamps |
| Every lookup takes minutes the first time | cold cache, `ntoskrnl` PDB is large | wait once; the cache serves later sessions |

## Command reference: the debugger's API surface

### Session and environment

| Command | Does | Notes |
| --- | --- | --- |
| `.symfix` / `.sympath` | set or display the symbol path | `.symfix+` appends; `.sympath+` adds a directory |
| `.reload` | reload symbols (`/f`, `/f module`, `/user`) | `/user` needs `.process /p` first |
| `!sym noisy` | verbose symbol lookup log | `!sym quiet` turns it off |
| `.logopen` / `.logclose` | record the command window to a file | `/t` appends a timestamp to the name |
| `.dump` | write a dump of the target | `/ma` for user mode, `/f` for a kernel complete dump |
| `.detach` | disconnect, leave the target running | `q` terminates a user-mode target |
| `.effmach` | switch effective machine (`x86`, `native`) | needed for WOW64 targets and 32-bit dumps |

### Modules, symbols, types, locals

| Command | Does | Notes |
| --- | --- | --- |
| `lm` | loaded modules with symbol status | `lmv m name` is verbose for one module, `lmf` shows full paths |
| `lmvm name` | one module: path, timestamp, checksum, PDB | the fastest way to spot a stale build |
| `x module!pattern` | find symbols by wildcard | `x mydrv!*Dispatch*` before setting a breakpoint |
| `dt module!TYPE [addr]` | print a structure layout or an instance | `dt nt!_IRP` shows offsets; always use the underscore |
| `dx expr` | data-model expression, navigable objects | `dx -r2 @$curprocess`; understands pointers and arrays |
| `?? expr` | C++ expression, resolves locals and casts | `?? found == 0`; the tool for comparisons |
| `? expr` | MASM evaluator, number conversion | `? 0n34636` prints the hex; `0x`/`0n`/`0y` prefixes |
| `dv /t /v` | locals of the current frame, with types and addresses | empty means wrong frame or no private symbols |

### Memory

| Command | Does | Notes |
| --- | --- | --- |
| `db`/`dw`/`dd`/`dq` | dump bytes, words, dwords, qwords | append `L20` for a count: `dq @rsp L8` |
| `da`/`du`/`dc` | ASCII, Unicode, dword+char view | `du @rcx` is the way to read a `UNICODE_STRING` buffer |
| `eb`/`ew`/`ed`/`eq` | write memory | live only; a code patch is undone by a restart, data writes are not |
| `s -a`/`s -u`/`s -b` | search strings or bytes in a range | `s -a 0 L?7fffffff "needle"` |
| `!address addr` | region containing an address: usage, state, protect | `!address` alone summarizes the whole space |
| `!heap -a addr` | heap block containing an address | `-p` needs full heap tracking from `gflags /p /enable app.exe /full` |

### Registers, stacks, contexts

| Command | Does | Notes |
| --- | --- | --- |
| `r` | all registers; `r rax` one; `r @rax=0` sets | `r $t0=5` are pseudo-registers for scripts |
| `k` | call stack of the current context | the frame number column is what `.frame` takes |
| `kb` | `k` plus the first arguments per frame | the fastest way to see a NULL argument |
| `kv` | `k` plus frame-pointer and calling-convention detail | use when frames look shifted |
| `~` | list threads (user) or processors (kernel) | `.` marks current, `#` marks the last event |
| `.thread` | set the current thread by ETHREAD address | kernel sessions; needed to read a thread's stack |
| `.process` | set the process context (`/p` for implicit only, `/i` invasive) | `/p` plus `.reload /user` to read user memory |
| `!teb` / `!peb` | thread and process environment blocks | `!teb` shows LastErrorValue and the stack range |
| `!handle 0 0` / `!handle h f` | list handles, or detail one handle | kernel: set `.process` context first |

### Breakpoints and execution control

| Command | Does | Notes |
| --- | --- | --- |
| `bp addr` | software breakpoint at an address or `module!symbol` | patches code bytes; lost when the image is reloaded |
| `bu module!symbol` | breakpoint resolved when the module loads | the right tool for a driver that is not loaded yet |
| `bm module!pattern` | breakpoints on every matching symbol | `bm mydrv!*Dispatch*` |
| `ba r4 addr` | processor breakpoint: `r`, `w`, `e`, or `x`, plus size | 4 per processor, needs alignment; `ba w8 &g_state` |
| `bl` / `bc` / `bd` / `be` | list, clear, disable, enable | `bc *` before handing a session to anyone else |
| `g` | continue to the next event | `gc` continues from a command string |
| `gu` | run until the current function returns | stops in the caller; confused by tail calls |
| `p` | step over one instruction or source line | a `syscall` executes completely |
| `t` | step into | follows a call |
| `pc` | step to the next `call` in this function | good for skipping loop bodies |
| `ph` | step to the next branch (jump, call, or return) | good for leaving a loop |

### Exceptions, dumps, analysis

| Command | Does | Notes |
| --- | --- | --- |
| `sxe av` / `sxd av` / `sxi av` | break on first chance, second chance, or never | `sxe -c "kb; g" av` logs and continues |
| `sxe ld:mydrv.sys` | break when a module loads | the standard way to catch a driver early |
| `.exr -1` | most recent exception record | read the code and the two parameters |
| `.cxr addr` / `.ecxr` | switch to a saved context / the exception's context | `.ecxr` is how you get back to the faulting frame |
| `!analyze -v` | automated triage: faulting IP, stack, module | start here on any dump, then verify by hand |
| `.dump /ma file` | full-memory user dump | `/o` overwrites an existing file |

### Scripting and automation

| Command | Does | Notes |
| --- | --- | --- |
| `aS name "commands"` | define an alias; `al` lists, `ad` deletes | expand with `name` or `${name}` next to other text |
| `${/v:name}` | treat the alias name stored in a variable | how loops create aliases dynamically |
| `$$< file` | run a file as one command per line | keep one command per line; no control flow across lines |
| `.scriptload file.js` | load a JavaScript provider script | call functions with `dx @$scriptContents.fn(arg)` |
| `.scriptrun file.js` | load, initialise, and invoke in one step | what a breakpoint command string usually calls |
| `.foreach (v {cmd}) { .echo v }` | iterate over command output | pair with `aS` to build addresses |

## Step-by-step: bring up a session you can trust

1. **Match the debugger to the target.** The banner says `AMD64` or `x86`, and a 32-bit
   debugger cannot usefully open a 64-bit target, so switch to the `x64` directory.
2. **Set the symbol path in the environment** with the `setx` line above and open a new
   shell; running processes keep the old environment.
3. **Start the debugger** with `-p`, `-pn`, an executable name, or `-z`. Check: the prompt
   is at a break and the banner names the target and its architecture.
4. **Prove the path inside the session** with `.sympath`: the expanded path must list the
   cache and the server. An empty expansion means the variable never reached the process.
5. **Load and verify symbols** with `.reload`, `lm`, and `lmvm`. The module must show
   `pdb symbols` or `private pdb symbols` and a timestamp matching your build.
6. **Verify types** with `x module!*pattern*` and `dt module!TYPE`; if `dt` reports an
   unknown type, the PDB is missing or the module name is wrong.
7. **Set one breakpoint** with `bp`, then confirm with `bl`: enabled, resolved address, hit
   count `0001 (0001)`.
8. **Reproduce, then inspect in a fixed order**: `k` (where), `r` (state), `dv` (locals),
   `dt` (layout), `d` (memory). Every frame must have a name and a small offset.
9. **Capture and leave cleanly**: `.logopen` before the run, `.dump /ma` while stopped,
   `.logclose`, then `bc *` and `.detach` or `q`.

## Complete example: the null-dereference lab

The lab is small on purpose: one C file, one missing check, one access violation whose
faulting address is not zero. Reproducing it builds the habit of reading the exception record.

### The program

```c
/* crashlab.c - build: cl /nologo /Zi /Od /W4 crashlab.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

typedef struct _ITEM {
    unsigned long Id;          /* +0x00 */
    char *Name;                /* +0x08 */
    struct _ITEM *Next;        /* +0x10 */
} ITEM, *PITEM;

static PITEM g_head = NULL;

static PITEM Insert(unsigned long id, const char *name)
{
    PITEM item = (PITEM)malloc(sizeof(ITEM));
    if (item == NULL) {
        return NULL;
    }
    item->Name = _strdup(name);
    if (item->Name == NULL) {
        free(item);
        return NULL;
    }
    item->Id = id;
    item->Next = g_head;
    g_head = item;
    return item;
}

static PITEM Find(unsigned long id)
{
    PITEM it;

    for (it = g_head; it != NULL; it = it->Next) {
        if (it->Id == id) {
            return it;
        }
    }
    return NULL;               /* not found: the caller must check */
}

int main(void)
{
    PITEM found;

    if (Insert(1, "alpha") == NULL || Insert(2, "beta") == NULL) {
        return 1;
    }

    found = Find(3);                           /* 3 was never inserted */
    printf("id 3 length = %zu\n", strlen(found->Name));
    return 0;
}
```

Build: `crashlab.exe`, `crashlab.pdb`, `crashlab.obj`. Run it from a shell first — it exits
with `0xc0000005`, which is all the user sees; everything else comes from the debugger.

### Reproduce under the debugger

```
C:\lab> cdb crashlab.exe
Microsoft (R) Windows Debugger Version 10.0.26100.1 AMD64

0:000> sxe av
0:000> g
(1f4c.2a10): Access violation - code c0000005 (first chance)
First chance exceptions are reported before any exception handling.
This exception may be expected and handled.
crashlab!main+0x38:
00007ff6`9a2c10b8 488b4108        mov     rax,qword ptr [rcx+8] ds:00000000`00000008=????????????????
```

Three facts on that screen each matter:

- `(first chance)` means the debugger stopped at the fault, before any handler could swallow
  it; `sxe av` guarantees this, and without it the break may come at the second chance.
- `main+0x38` puts the fault inside `main`, not inside `strlen`: frame 0 is where the
  instruction is, not where the logic is.
- `ds:00000000`00000008` is the *effective address*: `0x8`, not `0x0`, because the base
  pointer is NULL and the member offset is 8. That one number says the base was NULL.

### Inspect the faulting frame

```
0:000> k
 # Child-SP          RetAddr               Call Site
00 000000d1`a63af8e8 00007ff6`9a2c10a5     crashlab!main+0x38
01 000000d1`a63af9e0 00007ff6`9a2c117f     crashlab!invoke_main+0x22
02 000000d1`a63afa30 00007ffb`1c1ce8d7     crashlab!__scrt_common_main_seh+0x10f
03 000000d1`a63afa70 00007ffb`37dc7374     KERNEL32!BaseThreadInitThunk+0x14
04 000000d1`a63afaa0 00000000`00000000     ntdll!RtlUserThreadStart+0x21

0:000> r
rax=000000d1a63afa40 rbx=0000000000000000 rcx=0000000000000000
rdx=000000d1a63afa40 rsi=0000000000000000 rdi=0000000000000000
rip=00007ff69a2c10b8 rsp=000000d1a63af8e8 rbp=0000000000000000

0:000> dv /t /v
0x000000d1`a63af928  struct _ITEM * found = 0x00000000`00000000
```

The stack is named with small offsets, so symbols are real. `rcx` is zero and the faulting
instruction reads `[rcx+8]`, so the base is NULL. `dv` names the local: `found` is zero, yet
the code goes on to read the name of record 3 — a lookup that failed and was never checked.
Confirm the layout:

```
0:000> dt crashlab!ITEM
   +0x000 Id               : Uint4B
   +0x008 Name             : Ptr64 Char
   +0x010 Next             : Ptr64 _ITEM

0:000> ?? found == 0
bool true
```

`Name` is at `+0x008`, exactly the faulting address. Two ways to see this earlier, with less
noise:

```
0:000> u @rip L3
crashlab!main+0x38:
00007ff6`9a2c10b8 488b4108        mov     rax,qword ptr [rcx+8]
00007ff6`9a2c10bd 488bc8          mov     rcx,rax
00007ff6`9a2c10c0 e83bfaffff      call    qword ptr [crashlab!_imp_strlen (00007ff6`9a2c2100)]

0:000> bp crashlab!Find ".if (poi(@rcx) == 3) { .echo --- lookup of missing id 3 ---; kb } .else { gc }"
0:000> bl
     0 e Disable Clear  00007ff6`9a2c1050     0001 (0001)  0:**** crashlab!Find
0:000> bc *
```

The conditional breakpoint costs nothing until the interesting call, then prints the stack and
stops. `@rcx` is the first argument under the x64 convention and `poi()` reads the
pointer-sized value at that address; when the condition is false the string runs `gc`, so a
loop of a thousand calls produces one stop.

### Fix and confirm

Add the missing check in `main`, keeping the same build flags:

```c
    found = Find(3);
    if (found == NULL) {
        printf("id 3 not found\n");
        return 2;
    }
    printf("id 3 length = %zu\n", strlen(found->Name));
```

Rebuild, then re-run the same sequence: `cdb crashlab.exe`, `sxe av`, `bp crashlab!Find` with
the same condition, `g`. The breakpoint never fires, the program prints `id 3 not found`, and
the session ends with an exit event. The sequence that faulted now takes the guarded path.

### Turn the crash into a reusable dump

```
0:000> .logopen C:\dbg\crashlab.log
0:000> .dump /ma C:\dbg\crashlab.dmp
0:000> .logclose

:: later, with no target attached
C:\lab> windbg -z C:\dbg\crashlab.dmp
0:000> .symfix C:\symbols
0:000> .reload
0:000> !analyze -v
FAULTING_IP:
crashlab!main+0x38
00007ff6`9a2c10b8 488b4108        mov     rax,qword ptr [rcx+8]
```

`!analyze -v` is a first pass, not a verdict: confirm its fields by hand with `k`, `r`, and
`dt`. Then `.ecxr` followed by `kb` — a dump usually opens inside the exception dispatcher,
and without `.ecxr` the stack shows the unwinding machinery instead of `crashlab!main+0x38`.

## Complete example: attach to a live process, break, inspect, step

This is the rehearsal for the same moves on a kernel target: attach, break on a call,
inspect its arguments, step, and read a failure status instead of a crash.

```
0:000> ~
   0  Id: 3f18.1b04 Suspend: 1 Teb: 0000002c`0f0d7000 Unfrozen
   1  Id: 3f18.2c88 Suspend: 1 Teb: 0000002c`0f0e1000 Unfrozen
.  2  Id: 3f18.3f10 Suspend: 1 Teb: 0000002c`0f0f3000 Unfrozen
0:000> bp KERNEL32!CreateFileW
0:000> g
Breakpoint 0 hit
KERNEL32!CreateFileW:
00007ffb`1c1e6b60 ff25aa4e0c00    jmp     qword ptr [KERNEL32!_imp_CreateFileW (00007ffb`1c2aba10)]
0:000> k
 # Child-SP          RetAddr               Call Site
00 0000002c`0f0ff6a8 00007ffb`1c1c7c3e     KERNEL32!CreateFileW
01 0000002c`0f0ff6b0 00007ff6`9a2c1092     KERNELBASE!CreateFileInternal+0x2f6
02 0000002c`0f0ff6f0 00007ff6`9a2c1140     toolapp!wmain+0x12
03 0000002c`0f0ff730 00007ffb`1c1ce8d7     toolapp!invoke_main+0x22
04 0000002c`0f0ff770 00000000`00000000     KERNEL32!BaseThreadInitThunk+0x14
0:000> r rcx
rcx=0000002c0f0ff7a8
0:000> du @rcx
0000002c`0f0ff7a8  "C:\work\input.txt"
```

What mattered: the first argument is in `rcx`, it is a `PCWSTR`, and `du` decoded it in place.
That is the inspection pattern for every API: find the parameter, read the register, then read
memory as bytes (`db`), dwords (`dd`), ANSI (`da`), or Unicode (`du`). Stepping from here:

```
0:000> p
KERNEL32!CreateFileW+0x7:
00007ffb`1c1e6b67 488d0dd2bb0b00  lea     rcx,[KERNEL32!_imp_CreateFileW (00007ffb`1c2aba10)]
0:000> gu
KERNELBASE!CreateFileW+0x66:
00007ffb`1c1c7c3e 488b4c2448      mov     rcx,qword ptr [rsp+48h]
0:000> r eax
eax=c0000034
0:000> !error @eax
Error code: (NTSTATUS) 0xc0000034 (3221225524) - Object Name not found.
```

`gu` is the workhorse: rather than tracing every instruction of `CreateFileW`, run until it
returns and read the result. The return value is in `eax`/`rax`; `!error` turns it into text,
and `0xc0000034` means the path did not exist, a different bug from a permissions failure.

An exception is caught the same way, with the filter set before the run; the lab binary is the
subject here, so the frames name `crashlab`:

```
0:000> sxe av
0:000> g
(1f4c.2a10): Access violation - code c0000005 (first chance)
crashlab!main+0x38:
00007ff6`9a2c10b8 488b4108        mov     rax,qword ptr [rcx+8] ds:00000000`00000008=????????????????
0:000> .exr -1
ExceptionAddress: 00007ff69a2c10b8 (crashlab!main+0x0000000000000038)
   ExceptionCode: c0000005 (Access violation)
NumberParameters: 2
   Parameter[0]: 0000000000000000
   Parameter[1]: 0000000000000008
Attempt to read from address 0000000000000008
```

`Parameter[0]` is the operation — 0 read, 1 write, 8 execute — and `Parameter[1]` is the
address touched. Read those two numbers before reading source: they separate a NULL
dereference from a freed pointer and a read from a buffer overflow.

## Variations and advanced use

- **Dump triage without a live target.** `windbg -z file.dmp`, `.symfix`, `!analyze -v`, then
  `kv`, `.ecxr`, `dt nt!_IRP`. A `/ma` dump keeps every committed region, so object state
  survives; a minidump keeps stacks and module list, enough for a fault and not for a leak.
- **Kernel session, same commands.** `lm`, `!process 0 0`, `.process /p <EPROCESS>`, `.reload
  /user`, `!thread`, `!drvobj mydrv 2`, `!pool <addr>`, and `sxe ld:mydrv.sys` to stop when
  the driver loads. Remember that `~` lists processors here.
- **JavaScript for logic a command string cannot express.** `.scriptload` the file, then call
  it with `dx @$scriptContents.WalkList(0xADDRESS, 8)`:

```js
// listwalk.js
"use strict";

function WalkList(headAddress, max)
{
    var node = headAddress;
    for (var count = 0; node != 0 && count < max; count++) {
        var id = host.memory.readMemoryValues(node, 1, 4)[0];        // ITEM.Id
        var pair = host.memory.readMemoryValues(node.add(8), 2, 8);  // ITEM.Name, ITEM.Next
        host.diagnostics.debugLog("item ", count, ": id=", id,
                                  " name=", host.memory.readString(pair[0]), "\n");
        node = pair[1];
    }
    return count;
}
```

- **Aliases as a personal command set.** `aS dtp "dt crashlab!ITEM poi(crashlab!g_head)"` then
  `dtp`; `al` lists, `ad dtp` removes, and `${dtp}` expands when glued to other text.

## Failure modes and diagnosis

| Symptom | Likely cause | First command |
| --- | --- | --- |
| `k` shows only addresses, or `module+0x2f3a` everywhere | no symbols, or export symbols only | `lm` then `.symfix` and `.reload /f` |
| `*** ERROR: Module load completed but symbols could not be loaded` | PDB not on the path | `.sympath+ C:\build\out`, `.reload /f mydrv.sys` |
| Breakpoint set, never hit | code never reached; wrong module; inlined or optimized away; module loaded after `bp` | `bl`, `lm`, then `bu module!symbol` |
| Breakpoint hits in the wrong place after a rebuild | `bp` was set on an address from the previous build | `bc *`, restart the target, re-`bp` by symbol |
| `bp` reports "cannot set breakpoint" | address not executable or not present | check `lm`/`x`; use `ba e1` or wait for load |
| `dv` prints nothing, locals are `<unavailable>` | optimized build, or stopped in a frame with no locals | rebuild `/Od /Zi`; use `kb` and `dt` on the pointer |
| Stepping jumps around, frames look shifted | inlining and tail calls in an optimized build | `kv`, `uf module!func`, break on a stable symbol |
| Wrong architecture errors; `lm` shows no modules | 32-bit debugger on a 64-bit target, or WOW64 process | use `x64\cdb.exe`; for WOW64 use `.effmach x86` |
| `Source file not found`, source window empty | PDB has a source path from another machine | `.srcpath+ C:\src`, `l+s` to allow loading |
| `q` killed the process you were debugging | `q` terminates a user-mode target | `.detach` leaves it running |
| Commands rejected: "Debuggee is running" | target resumed | `Ctrl+Break` or the Break button |
| Kernel target does not stop | interrupts disabled at a high IRQL, or transport broken | wait, then check the transport; do not assume the session is dead |
| Attach fails with access denied | target is elevated or protected | run the debugger elevated; protected processes are off limits |
| Dump opens but `k` starts in the exception dispatcher | dump captured at the handler, not the fault | `.ecxr` then `k` |

## Safety, IRQL, and cleanup checklist

- **A kernel break stops the machine.** Every processor, driver, and timer: a target that
  serves traffic drops it, a cluster loses quorum, a watchdog may fire. Never break production.
- **Elevation is required** for kernel attach, local kernel debugging, and attaching to
  elevated processes; Secure Boot blocks local kernel debugging unless disabled in firmware.
- **Snapshot the VM before the first session**: a driver that fails at boot is hard to remove.
- **Local kernel debugging cannot set breakpoints.** It reads the running machine, so values
  change under you between commands; anything that needs a stop needs a real target.
- **Know the mode before you read a number.** In kernel mode `~` is processors, `k` is the
  current thread's kernel stack, and user addresses are meaningless until `.process /p`.
- **Kernel breakpoints are global and persist for the session.** Clear them with `bc *` before
  returning a machine to anyone, and never leave one in a boot path without a recovery plan.
- **`bp` patches code; `e` writes memory.** Restarting undoes code patches but not data
  writes; never "fix" a bug by editing a global and call it verified.
- **Dumps are sensitive.** A process dump contains tokens and passwords; a kernel dump
  contains other processes' memory. Keep them in a protected directory, delete them after.
- **Symbol caches grow** to tens of gigabytes; prune the cache or point it at a share
  instead of filling a system disk.
- **Leave the target as you found it:** `bc *`, `.logclose`, then `.detach` (keeps it
  running) or `q` (terminates it) deliberately.

## Verification

A session is ready for real work when all of these hold; stop at the first failure.

1. **Banner and prompt** name the expected target and architecture (`AMD64`, `x86`) and sit
   at a break (`0:000>` / `0: kd>`), not at "Debuggee is running".
2. **Symbol path**: `.sympath` shows an expanded path with a cache and
   `https://msdl.microsoft.com/download/symbols`.
3. **Module proof**: `lm` shows the module as `pdb symbols` or `private pdb symbols`, and
   `lmvm` prints an image timestamp matching the binary you built.
4. **Type proof**: `dt <module>!_<STRUCT>` prints offsets such as `+0x008 Name`, and
   `x <module>!*<name>*` resolves the functions you plan to break on.
5. **Stack proof**: `k` or `kb` prints named frames with offsets below `0x1000`, and frame 0
   is in the module you expect.
6. **Breakpoint proof**: `bp <module>!<symbol>` returns silently, `bl` lists it as enabled
   with a resolved address, and one `g` hits it once at the expected instruction.
7. **Lab proof**: before the fix the run stops with `Access violation - code c0000005`,
   `.exr -1` reports `Parameter[1]: 0000000000000008`, and `dv` shows the pointer as zero;
   after the fix the same sequence prints the guarded message and no violation.
8. **Capture proof**: the `.logopen` file exists and holds the commands and output, and
   `windbg -z` on the `.dump /ma` file reproduces the same frame after `.ecxr`.
