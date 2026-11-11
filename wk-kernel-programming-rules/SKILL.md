---
name: wk-kernel-programming-rules
description: Use when writing or reviewing any kernel-mode C or C++ code and you need the non-negotiable rules first: NTSTATUS encoding and NT_SUCCESS/NT_ERROR checks, IRQL legality, C++ runtime restrictions, Debug versus Release behavior, SAL annotations, and the DriverEntry and unload contracts. Provides a 20-status reference matrix and a pre-flight checklist for any WDK API call.
---

# Kernel programming rules: the contract before the first line of code

Kernel code runs with no safety net: an unhandled exception does not kill a process, it
stops the machine, and a leaked allocation survives until the next boot. The APIs look
familiar because they are C functions with C names, but every one of them carries a
contract about IRQL, paging, blocking, ownership, and failure that user-mode code never
had to satisfy. This skill states those contracts in the order they bite: execution
environment, status codes, language restrictions, build configuration, and the two
routines the loader calls. The rule that prevents most disasters: **check every status,
know your IRQL, and give every resource exactly one owner and one release path**.

## When to use

- You are about to write a new driver file, a new dispatch routine, or a helper that
  calls any `Ex*`, `Ke*`, `Io*`, `Mm*`, `Ob*`, `Ps*`, `Rtl*`, or `Zw*` routine.
- You are reviewing kernel code and need to justify a rejection: unchecked status,
  blocking at raised IRQL, a C++ feature that cannot link, missing unload cleanup.
- A build fails with C2712, C4530, LNK2019 on `DriverEntry`, or a code-analysis warning.
- A driver loads but returns the wrong error to `sc start`, or `DriverEntry` fails and
  the reason is invisible.
- You must decide between a C and a C++ source file, and what subset of C++ is legal.
- You are onboarding to an existing driver and need the invariant list before touching it.

Do not use this skill for API-specific mechanics. Use `wk-kernel-strings` for
`UNICODE_STRING`, `wk-kernel-memory-pools` for allocation, `wk-irql` for the full IRQL
model, and `wk-kernel-exceptions` for structured exception handling and probing.

## Prerequisites and related skills

- **Context:** kernel-mode code built with the WDK (Windows Driver Kit) for Windows
  10/11, x64 or ARM64, compiled as a driver project that produces a `.sys` image.
- **Headers and libs:** `#include <ntddk.h>` for WDM-style drivers, `<wdf.h>` with
  `<ntddk.h>` for KMDF, `<fltKernel.h>` for minifilters. `ntoskrnl.lib` is linked by the
  driver project; safe string routines additionally need `ntstrsafe.lib`, and
  `wdmsec.lib` is needed for `IoCreateDeviceSecure`. Never include `<windows.h>` or
  `<winternl.h>` in a driver.
- **Related skills:** `wk-driver-dev-environment` (toolchain and test VM),
  `wk-driver-anatomy` (complete driver with a real protocol), `wk-irql` (legal API per
  level), `wk-kernel-exceptions` (SEH and user-memory probing),
  `wk-kernel-libraries` (multi-file structure, CRT and linker settings).
- **C++ standard:** the C++ material here needs C++14; C++17 changes which `operator new`
  overloads the compiler demands, as noted below.

## User mode versus kernel mode

| Aspect | User mode | Kernel mode | What it forces on you |
|---|---|---|---|
| Unhandled exception | Process dies; system survives | **Bugcheck; whole machine dies** | Wrap risky accesses; never let an exception escape |
| Termination | Process exit frees all private memory and handles | Driver unload frees **nothing** | Every allocation, handle, reference, and lock needs an explicit release path |
| Return values | Sometimes ignored | **Almost never ignorable** | `NT_SUCCESS` on every call; propagate or handle |
| IRQL | Always `PASSIVE_LEVEL` (0) | 0 up to `HIGH_LEVEL` (15) | Know the level before choosing an API |
| Blast radius | One process | Whole system, all sessions | Assume one bug is a fleet-wide outage |
| Debugging | Debugger attaches to a process | Debugger must run on another machine; a breakpoint freezes everything | Plan a two-machine setup from day one |
| Libraries | Nearly all of the C/C++ ecosystem | `ntoskrnl` exports plus a few static libs | No STL, no CRT heap, no C++ runtime |
| Exception handling | C++ `try`/`catch`, SEH, `/EHsc` | SEH only (`__try`/`__except`) | No C++ exceptions anywhere |
| C++ runtime | Full runtime | None | No `new`/`delete` unless overloaded, no RTTI, no global constructors that run |
| Stack | 1 MB default per thread | 12 KB on x86, 24 KB (0x6000) on x64 | No large locals; allocate anything big from pool |
| Floating point | Free use | Bracket with `KeSaveFloatingPointState`/`KeRestoreFloatingPointState`, below `DISPATCH_LEVEL` | Keep FP out of the driver, or confine it to one helper |
| Failure visibility | Event log, exceptions, dumps | Bugcheck, dump, debugger, ETW | Build tracing in before you need it |

The stack difference is not cosmetic: a 4 KB local array in a routine called from a
dispatch routine at `DISPATCH_LEVEL` can overflow the stack and produce bugcheck 0x7F
(`UNEXPECTED_KERNEL_MODE_TRAP`) or silently corrupt the adjacent stack frame.

## Mental model

A driver is a set of callbacks registered into kernel subsystems. The kernel calls you;
you almost never call the kernel's own entry points on your own schedule. Every callback
arrives with a fixed execution environment that you cannot change, only respect.

```text
  user mode                          kernel mode
  ---------                          -----------
  app --CreateFile/DeviceIoControl--> +-------------------------------+
                                      | I/O manager builds an IRP     |
                                      +-------------------------------+
                                             | MajorFunction[major]
                                             v
                                      +-------------------------------+     +------------------+
                                      | your dispatch routine         |---->| kernel API       |
                                      | IRQL 0 .. DISPATCH_LEVEL      |     | Ex/Ke/Io/Mm/Rtl  |
                                      | arbitrary thread, maybe       |     | Zw/Ob/Ps/Se      |
                                      | arbitrary process context     |     +------------------+
                                      +-------------------------------+              |
                                             |                                     | may fault,
                                             | NTSTATUS                            | may block,
                                             v                                     | may raise
                                      IoCompleteRequest                             v
                                                                          bugcheck on misuse
```

### The four execution environments

1. **Passive, arbitrary thread** — most dispatch routines: paging, blocking waits, file
   and registry I/O, and probed user memory are all legal.
2. **Passive, system thread** — `DriverEntry`, `DriverUnload`, work items, system
   threads: there is no user process context, so user addresses are meaningless.
3. **`DISPATCH_LEVEL`** — DPCs, timer callbacks, completion routines, code after
   `KeAcquireSpinLock`: non-paged pool only, no waits, no paging, no user memory.
4. **Device IRQL (DIRQL)** — ISRs: only non-paged register work on data guarded by that
   interrupt's spinlock.

### IRQL in one screen

| Level | Constant | Value | Legal work |
|---|---|---|---|
| 0 | `PASSIVE_LEVEL` | 0 | Everything, including paging, waits, file and registry I/O, user memory |
| 1 | `APC_LEVEL` | 1 | No paging I/O that waits on APCs; no user memory; fast mutexes and guarded regions |
| 2 | `DISPATCH_LEVEL` | 2 | Non-paged memory only, spinlocks, `Interlocked*`, DPC queuing; no waits, no paging |
| 3-11 | device IRQL (DIRQL) | HAL-assigned | ISR code only: register access and DPC queuing; `CMCI_LEVEL` (5) is reserved inside this range |
| 13-14 | `CLOCK_LEVEL` (13), `IPI_LEVEL`/`POWER_LEVEL` (14) | 13-14 | Reserved for the clock and inter-processor interrupts |
| 15 | `HIGH_LEVEL` | 15 | Reserved; masks every interrupt |

Rules that follow from the model:

- **IRQL only rises inside a call chain, and you must restore it exactly.** A routine that
  raises must lower before returning; `KeAcquireSpinLock` saves the old IRQL and
  `KeReleaseSpinLock` restores it.
- **Never call `KeLowerIrql` with a value lower than the IRQL you entered with.** The
  kernel bugchecks (0x1E or 0xA) or runs with interrupts enabled inside a critical
  section.
- **`KeGetCurrentIrql()` is the only reliable answer** to "can I call this here?". Call
  it; do not infer the level from a routine's name.
- **Raising IRQL does not provide mutual exclusion.** It masks interrupts on the current
  processor only; use a spinlock for cross-processor exclusion.
- Full treatment: `wk-irql`.

## NTSTATUS: encoding, helpers, and the 20 statuses you will actually see

`NTSTATUS` is a signed 32-bit integer in which the bits carry meaning:

| Bits | Field | Meaning |
|---|---|---|
| 31-30 | Severity | `00` success, `01` informational, `10` warning, `11` error |
| 29 | Customer | Vendor-defined status; normally 0 |
| 27-16 | Facility | Subsystem that defined the code: `0x0` null, `0x1` RPC, `0x2` dispatch, `0x3` storage, `0x4` interface, `0x7` Win32, `0x8` Windows, `0x9` security, `0xA` control |
| 15-0 | Code | The specific value inside the facility |

Every severity-3 value has the top bit set and is negative as a `LONG`; every success,
informational, and warning value is non-negative. The helper macros are defined in
`ntdef.h` and mean exactly this:

```c
#define NT_SUCCESS(s)     (((NTSTATUS)(s)) >= 0)          /* severity 0..2 */
#define NT_INFORMATION(s) ((((ULONG)(s)) >> 30) == 1)
#define NT_WARNING(s)     ((((ULONG)(s)) >> 30) == 2)
#define NT_ERROR(s)       ((((ULONG)(s)) >> 30) == 3)     /* severity 3 only */
```

Consequences that catch experienced developers:

- **`NT_SUCCESS` is true for warnings and for `STATUS_PENDING`.**
  `STATUS_BUFFER_OVERFLOW` (0x80000005) and `STATUS_NO_MORE_ENTRIES` (0x8000001A) pass
  it, so compare against `STATUS_SUCCESS` when truncation matters, and return
  `STATUS_PENDING` only together with `IoMarkIrpPending`.
- **`NT_ERROR` is narrower than `!NT_SUCCESS`.** It excludes informational and warning
  codes, so `if (NT_ERROR(status))` silently accepts `STATUS_BUFFER_OVERFLOW`. Use
  `if (!NT_SUCCESS(status))` for failure paths and explicit compares for semantics.
- **Never return a Win32 error from a kernel routine.** `ERROR_ACCESS_DENIED` (5) is
  positive and reads as success; convert with `NTSTATUS_FROM_WIN32(5)` when a value
  genuinely originates as a Win32 error.
- **Return the status you were given** when a lower call fails; do not rewrite
  `STATUS_INSUFFICIENT_RESOURCES` into `STATUS_UNSUCCESSFUL`, because callers and the SCM
  map specific codes to specific user-visible errors.
- **Pick between the two common failures by cause:** bad input is
  `STATUS_INVALID_PARAMETER` (`STATUS_INVALID_BUFFER_SIZE` for a length mismatch), while
  failure to obtain memory, a mapping, or an object is `STATUS_INSUFFICIENT_RESOURCES`.
  Never use either for the other's cause.

| Status | Value | Meaning | Typical cause in a driver |
|---|---|---|---|
| `STATUS_SUCCESS` | 0x00000000 | Completed | Normal path |
| `STATUS_PENDING` | 0x00000103 | Queued for later completion | Dispatch returned after `IoMarkIrpPending` |
| `STATUS_TIMEOUT` | 0x00000102 | Wait timed out (success severity) | `KeWaitForSingleObject` relative timeout expired |
| `STATUS_NO_MORE_ENTRIES` | 0x8000001A | Enumeration finished (warning) | End of a `ZwEnumerateKey`/value loop |
| `STATUS_BUFFER_OVERFLOW` | 0x80000005 | Result truncated to fit (warning) | `RtlStringCbPrintfW` clipped; reply buffer too small |
| `STATUS_UNSUCCESSFUL` | 0xC0000001 | Generic failure | A helper failed and the specific code was discarded |
| `STATUS_INVALID_PARAMETER` | 0xC000000D | Bad argument value | Zero length, NULL required pointer, misaligned IOCTL input |
| `STATUS_INVALID_BUFFER_SIZE` | 0xC0000206 | Length inconsistent with the buffer or type | IOCTL length field disagrees with the payload |
| `STATUS_INVALID_DEVICE_REQUEST` | 0xC0000010 | Unsupported operation | Unhandled `IRP_MJ_*` reaching `IopInvalidDeviceRequest` |
| `STATUS_INSUFFICIENT_RESOURCES` | 0xC000009A | Cannot obtain a resource | `ExAllocatePool2` returned NULL |
| `STATUS_ACCESS_DENIED` | 0xC0000022 | Access check failed | Client lacks rights; device SDDL too tight; forced access check |
| `STATUS_PRIVILEGE_NOT_HELD` | 0xC0000061 | Required privilege missing | `SeSinglePrivilegeCheck` on `SeDebugPrivilege` failed |
| `STATUS_OBJECT_NAME_NOT_FOUND` | 0xC0000034 | Name does not resolve | Wrong path in `IoGetDeviceObjectPointer`; missing symlink |
| `STATUS_OBJECT_NAME_COLLISION` | 0xC0000035 | Name already exists | Second load creating `\Device\Name` again |
| `STATUS_INVALID_HANDLE` | 0xC0000008 | Handle is not valid | Closed handle reused; handle from another process used in kernel |
| `STATUS_BUFFER_TOO_SMALL` | 0xC0000023 | Output buffer too small | `OutputBufferLength` check in an IOCTL handler |
| `STATUS_INFO_LENGTH_MISMATCH` | 0xC0000004 | Query buffer wrong size | `ZwQuerySystemInformation` with an undersized buffer; grow and retry |
| `STATUS_NOT_SUPPORTED` | 0xC00000BB | Not implemented | Optional feature absent on this build or device |
| `STATUS_DELETE_PENDING` | 0xC0000056 | Object is being deleted | Opening a device while its driver is unloading |
| `STATUS_CANCELLED` | 0xC0000120 | Request was cancelled | `IoCancelIrp` won the race; complete with this status |
| `STATUS_END_OF_FILE` | 0xC0000011 | Read past end | Zero-length read at end of file in a file-oriented driver |

Other codes that appear constantly in kernel work: `STATUS_MORE_PROCESSING_REQUIRED`
(0xC0000016, returned from a completion routine to stop completion),
`STATUS_INVALID_BUFFER_SIZE` (0xC0000206), `STATUS_DEVICE_NOT_READY` (0xC00000A3), and
`STATUS_REVISION_MISMATCH` (0xC0000059).

## Key APIs and structures

### Driver entry and unload contracts

| Contract | `DriverEntry` | `DriverUnload` |
|---|---|---|
| Signature | `NTSTATUS DriverEntry(PDRIVER_OBJECT, PUNICODE_STRING)` | `VOID DriverUnload(PDRIVER_OBJECT)` |
| Typedef | `DRIVER_INITIALIZE` | `DRIVER_UNLOAD` |
| IRQL | `PASSIVE_LEVEL` | `PASSIVE_LEVEL` |
| Context | System thread, no user process | System thread, arbitrary process context |
| Called | Once by the loader, before any dispatch | Once, before the image is unmapped |
| `DriverObject` | Partially initialized; all `MajorFunction[]` entries point at `IopInvalidDeviceRequest`; `DriverUnload` is NULL | Same object, devices still alive |
| `RegistryPath` | `\REGISTRY\MACHINE\SYSTEM\CurrentControlSet\Services\<ServiceName>`; **freed after return, must be copied** | Not provided |
| Failure | Non-success return: the loader calls no unload routine, so `DriverEntry` must unwind everything itself | Cannot fail |
| Name | Must be `extern "C"` in C++ files, or the linker looks for a mangled name | Same |

Rules that are not obvious:

- **If `DriverEntry` fails, `DriverUnload` is never called.** Every allocation and object
  created before the failing call must be released inside `DriverEntry`.
- **`RegistryPath` is read-only and transient.** Copy it if it must outlive the call, and
  free the copy in unload.
- **A driver with `DriverUnload == NULL` cannot be stopped.** `sc stop` returns error 1052
  (`ERROR_SERVICE_CANNOT_ACCEPT_CTRL`) and the image stays until reboot.
- **After `DriverUnload` returns, the image can be unmapped.** Any thread, DPC, timer,
  work item, or callback still able to execute driver code is a use-after-free; the usual
  bugchecks are 0xCE, 0x9F, and 0x50.
- **`IRP_MJ_CREATE` and `IRP_MJ_CLOSE` are effectively mandatory** for any driver a client
  opens by name; everything else can stay at the default.

### SAL annotations that do real work

SAL is compile-time only (it expands to nothing in code generation), but it drives static
analysis and documents the contract at the call site.

| Annotation | Contract |
|---|---|
| `_In_` | Required input; caller provides and initializes it |
| `_Out_` | Required output; callee writes it completely on success |
| `_Inout_` | Caller initializes, callee reads and writes |
| `_In_opt_`, `_Out_opt_`, `_Inout_opt_` | The pointer itself may be NULL |
| `_In_reads_(n)` | Buffer of at least `n` elements, readable |
| `_Out_writes_(n)` | Buffer of at least `n` elements, callee fills it |
| `_IRQL_requires_(level)` | Routine must be called at exactly that IRQL |
| `_IRQL_requires_max_(level)` | Routine must be called at that IRQL or lower |
| `_Must_inspect_result_` | Caller must check the returned status (warning if not) |
| `_Success_(expr)` | On success, `expr` is true; analysis may trust the outputs |
| `_Use_decl_annotations_` | Apply the header's annotations to this definition |

Bold warning: **SAL does not make code safe.** A routine annotated `_In_reads_(bytes)`
still trusts the caller to pass a real buffer of that size; the annotation only lets the
analyzer prove the caller wrong when it can see both sides.

### What C++ cannot do in the kernel, and why

| Feature | Works? | Reason | Replacement |
|---|---|---|---|
| `try` / `catch` / `throw` | No (C4530 or compile error) | The unwinder lives in the C++ runtime, which is absent | SEH `__try`/`__except`; see `wk-kernel-exceptions` |
| RTTI (`typeid`, `dynamic_cast`) | No (`/GR-`) | Requires runtime type descriptors | Explicit type tags or virtual functions |
| STL containers and algorithms | No | They allocate through the CRT or assume exceptions and `<new>` | Intrusive lists (`wk-kernel-linked-lists`), `ktl` (`wk-ktl`) |
| `new` / `delete` | Only if overloaded | Default operators call the user-mode heap through the CRT | Overload globally; see `wk-kernel-memory-pools` |
| Global objects with non-trivial constructors | Not run | No CRT initializer loop; the loader never calls them | POD globals, or allocate in `DriverEntry` and delete in unload |
| Function-local statics needing dynamic init | No | Thread-safe static initialization calls a runtime helper | Initialize in `DriverEntry`, guard with an explicit flag |
| Templates, `constexpr`, lambdas, `nullptr`, `auto` | Yes | Pure compile-time features | Use freely |
| Virtual functions | Yes | The compiler emits the dispatch array; no run-time type data | Fine for interfaces, avoid in hot `DISPATCH_LEVEL` paths |

Compile settings that make this explicit:

- **`/EHs-c-`** disables C++ exception handling; the WDK kernel project default.
- **`/GR-`** disables RTTI; the WDK kernel project default.
- **`/Zc:alignedNew-`** stops the compiler from demanding the C++17 aligned `operator new`
  overloads, which need `<new>` and `std::align_val_t`.
- **`extern "C"`** goes on every routine the kernel calls by name; callbacks registered by
  address may stay C++ as long as the signature matches the typedef exactly.

C2712 ("cannot use `__try` in functions that require object unwinding") appears when a
function holds a C++ object with a destructor and also uses `__try`. Fix it by moving the
`__try` block into a small helper that owns no objects, and call that helper from the
object-owning function.

### Debug versus Release

| Item | Debug (Checked) | Release (Free) |
|---|---|---|
| Preprocessor | `DBG` defined as 1 | `DBG` undefined |
| `KdPrint` | Calls `DbgPrint` | Compiles to nothing; arguments are not evaluated |
| `DbgPrint` | Emitted subject to the filter mask | Still emitted, still subject to the filter mask |
| `ASSERT` / `NT_ASSERT` | Evaluates the condition and breaks | Compiled out; side effects disappear with it |
| Optimization | `/Od`, easy stepping, variables visible | `/O2`, inlining, reordered memory access, variables in registers |
| Races | Hidden by slow code and debugger stops | Exposed; timing is completely different |
| `PAGED_CODE()` | Asserts the IRQL is below `DISPATCH_LEVEL` | Compiles to nothing |
| Driver Verifier | Complements a debug build | Required for release validation |

- **`DbgPrint` is not removed in release**, but its output is filtered by the component
  mask at `DPFLTR_INFO_LEVEL`. `DbgPrintEx` with `DPFLTR_ERROR_LEVEL`
  is the form that reliably appears; see `wk-kernel-tracing`.
- **Never put a required side effect inside an assert.** `NT_ASSERT(++calls > 0);` does
  nothing in a free build.
- **Optimization changes correctness, not just speed.** Reordered writes to a shared
  structure, a loop variable cached in a register, or a missing barrier are release-only
  bugs; use `KeMemoryBarrier`, `Interlocked*`, and `volatile` only where the memory model
  requires them. Build and test both configurations, always.

## Step-by-step

1. **Decide the file language.** Pure C (`.c`) removes every C++ question and is the
   default for small drivers. Choose `.cpp` only for RAII wrappers, and then put
   `extern "C"` on `DriverEntry`. Check: the linker resolves `DriverEntry` unmangled.
2. **List every resource `DriverEntry` will acquire** — pool, device object, symbolic
   link, handle, object reference, lock, callback registration — and write the release
   order before the acquire order. Check: each resource has one owner and one free site.
3. **Write `DriverEntry` with one failure block per step** that undoes only what was done.
   Check: a failure at the last step frees everything from the earlier steps.
4. **Register only the `MajorFunction[]` entries you implement.** Check: an unimplemented
   IOCTL returns `STATUS_INVALID_DEVICE_REQUEST`, not a crash.
5. **Annotate every routine you write** with `_In_`, `_Out_`, `_IRQL_requires_`, and
   `_Must_inspect_result_`. Check: `/analyze` reports no new warnings.
6. **Set `DriverUnload` and implement the exact inverse of step 2**, unregistering
   callbacks first and waiting for in-flight work. Check: 100 load/unload cycles with a
   steady `!poolused` count.
7. **Build Debug x64 at `/W4`** and fix every warning before adding features. Check: zero
   warnings and zero C281xx analysis warnings.
8. **Build Release x64 and repeat the functional test**, including failure paths driven by
   a fault-injection switch. Check: identical behavior, no assert-dependent logic.
9. **Enable Driver Verifier for your image only** with special pool, pool tracking, and
   forced IRQL checking (`verifier /standard /driver YourDriver.sys`). Check: no 0xC4.
10. **Record the IRQL and context of every routine** in a header comment. Check: no routine
    claims `PASSIVE_LEVEL` while a DPC or completion routine can call it.

## Complete example

A complete WDM driver that demonstrates the rules: status propagation, one cleanup path, a
private copy of `RegistryPath`, checked IRQL assumptions, and a symmetric unload.

```c
/* RulesDemo.c - build as an Empty WDM Driver, x64. */
#include <ntddk.h>

#define RULES_TAG           'elRD'      /* reads as "DRle" in a pool dump */
#define RULES_DEVICE_NAME   L"\\Device\\RulesDemo"
#define RULES_SYMLINK_NAME  L"\\DosDevices\\RulesDemo"
#define RULES_DEVICE_TYPE   0x8333
#define RULES_MAX_PATH      512

#define RULES_IOCTL_HITS \
    CTL_CODE(RULES_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

typedef struct _RULES_CONTEXT {
    UNICODE_STRING ServicePath;   /* private copy of the DriverEntry argument */
    ULONG          Hits;
} RULES_CONTEXT, *PRULES_CONTEXT;

DRIVER_INITIALIZE DriverEntry;
DRIVER_UNLOAD RulesUnload;
DRIVER_DISPATCH RulesCreateClose;
DRIVER_DISPATCH RulesDeviceControl;

#ifdef ALLOC_PRAGMA
#pragma alloc_text(INIT, DriverEntry)
#endif

static NTSTATUS
RulesCopyServicePath(_In_ PCUNICODE_STRING Source, _Out_ PUNICODE_STRING Copy)
{
    USHORT bytes;

    RtlZeroMemory(Copy, sizeof(*Copy));
    if (Source == NULL || Source->Buffer == NULL || Source->Length == 0) {
        return STATUS_INVALID_PARAMETER;
    }
    if ((Source->Length & 1) != 0) {          /* UTF-16 needs an even byte count */
        return STATUS_INVALID_PARAMETER;
    }
    if (Source->Length > RULES_MAX_PATH || Source->Length > 0xFFFF - sizeof(WCHAR)) {
        return STATUS_INVALID_PARAMETER;
    }

    bytes = (USHORT)(Source->Length + sizeof(WCHAR));
    Copy->Buffer = (PWCH)ExAllocatePool2(POOL_FLAG_PAGED, bytes, RULES_TAG);
    if (Copy->Buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    RtlCopyMemory(Copy->Buffer, Source->Buffer, Source->Length);
    Copy->Buffer[Source->Length / sizeof(WCHAR)] = L'\0';
    Copy->Length = Source->Length;
    Copy->MaximumLength = bytes;
    return STATUS_SUCCESS;
}

static VOID
RulesFreeServicePath(_Inout_ PUNICODE_STRING Path)
{
    if (Path->Buffer != NULL) {
        ExFreePoolWithTag(Path->Buffer, RULES_TAG);
    }
    RtlZeroMemory(Path, sizeof(*Path));
}

static NTSTATUS
RulesCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);

    Irp->IoStatus.Status = STATUS_SUCCESS;
    Irp->IoStatus.Information = 0;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return STATUS_SUCCESS;
}

static NTSTATUS
RulesDeviceControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    PRULES_CONTEXT context = (PRULES_CONTEXT)DeviceObject->DeviceExtension;
    NTSTATUS status;

    Irp->IoStatus.Information = 0;

    if (stack->Parameters.DeviceIoControl.IoControlCode != RULES_IOCTL_HITS) {
        status = STATUS_INVALID_DEVICE_REQUEST;
    } else if (stack->Parameters.DeviceIoControl.OutputBufferLength < sizeof(ULONG)) {
        status = STATUS_BUFFER_TOO_SMALL;
    } else if (Irp->AssociatedIrp.SystemBuffer == NULL) {
        status = STATUS_INVALID_PARAMETER;
    } else {
        ULONG hits = (ULONG)InterlockedIncrement((LONG*)&context->Hits);
        *(ULONG*)Irp->AssociatedIrp.SystemBuffer = hits;
        Irp->IoStatus.Information = sizeof(ULONG);
        status = STATUS_SUCCESS;
    }

    Irp->IoStatus.Status = status;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}

NTSTATUS
DriverEntry(_In_ PDRIVER_OBJECT DriverObject, _In_ PUNICODE_STRING RegistryPath)
{
    NTSTATUS status;
    PDEVICE_OBJECT device = NULL;
    PRULES_CONTEXT context;
    UNICODE_STRING servicePath;
    UNICODE_STRING deviceName;
    UNICODE_STRING symlinkName;

    /* DriverEntry is always passive; assert it so a refactor cannot move it. */
    NT_ASSERT(KeGetCurrentIrql() == PASSIVE_LEVEL);
    RtlZeroMemory(&servicePath, sizeof(servicePath));

    status = RulesCopyServicePath(RegistryPath, &servicePath);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    RtlInitUnicodeString(&deviceName, RULES_DEVICE_NAME);
    status = IoCreateDevice(DriverObject, sizeof(RULES_CONTEXT), &deviceName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        RulesFreeServicePath(&servicePath);
        return status;
    }

    context = (PRULES_CONTEXT)device->DeviceExtension;
    RtlZeroMemory(context, sizeof(*context));
    context->ServicePath = servicePath;

    RtlInitUnicodeString(&symlinkName, RULES_SYMLINK_NAME);
    status = IoCreateSymbolicLink(&symlinkName, &deviceName);
    if (!NT_SUCCESS(status)) {
        RulesFreeServicePath(&context->ServicePath);
        IoDeleteDevice(device);
        return status;
    }

    DriverObject->MajorFunction[IRP_MJ_CREATE] = RulesCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = RulesCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = RulesDeviceControl;
    DriverObject->DriverUnload = RulesUnload;

    device->Flags &= ~DO_DEVICE_INITIALIZING;

    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "RulesDemo: loaded, service key %wZ\n", &context->ServicePath);
    return STATUS_SUCCESS;
}

static VOID
RulesUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT device = DriverObject->DeviceObject;
    UNICODE_STRING symlinkName;

    if (device == NULL) {
        return;
    }

    RtlInitUnicodeString(&symlinkName, RULES_SYMLINK_NAME);
    IoDeleteSymbolicLink(&symlinkName);

    RulesFreeServicePath(&((PRULES_CONTEXT)device->DeviceExtension)->ServicePath);
    IoDeleteDevice(device);
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL, "RulesDemo: unloaded\n");
}
```

The C++ version differs in three places: the file is `.cpp`, `DriverEntry` is declared
`extern "C"` with the same two parameters, and objects with destructors are created with
an overloaded `operator new` in `DriverEntry` and destroyed with `delete` in
`RulesUnload`, never as globals. See `wk-kernel-memory-pools` for the overload set.

## Variations and advanced use

- **Pure C, one exit per failure.** Preferred for drivers under review: no destructors, no
  compiler switches to argue about, and every cleanup visible in source order. The cost is
  discipline; a missed `goto` is a leak.
- **C++ RAII, no exceptions.** Wrap each owned resource in a move-only class whose
  destructor releases it (pool, object reference, spinlock, event). The gain is real when
  a function has five failure points; the cost is C2712 when SEH is needed nearby.
  `wk-ktl` documents a ready-made wrapper library.
- **C with a per-request context object.** Allocate one structure per request, put every
  acquired resource in it, and free the whole structure in one place. This scales better
  than five independent locals and keeps cleanup order explicit.
- **`/analyze` and SDV.** Static analysis finds null dereferences and IRQL mismatches
  before the machine does; SDV finds protocol violations in WDM/KMDF callbacks. Neither
  replaces Driver Verifier.

## Before you call any kernel API

Answer all of these before the call, not after the crash:

1. **What IRQL am I at?** `KeGetCurrentIrql()`, or the documented contract of the routine
   I am inside. Does the callee allow it? `_IRQL_requires_max_` is the answer.
2. **Am I in a pageable routine?** Is `PAGED_CODE()` legal here, and is the callee legal
   at this IRQL if my code can be paged out?
3. **Can the callee block or wait?** If yes, it is illegal at `DISPATCH_LEVEL` and above.
4. **Can the callee touch user memory?** Only below `DISPATCH_LEVEL`, and only after
   probing and inside SEH; better, use buffered I/O and never see the user pointer.
5. **Can the callee raise an exception?** Quota allocation, `ProbeForWrite`, and
   `ExRaiseStatus` can. If yes, is it inside `__try` with a cleanup path?
6. **What does it return, and what do I do with each value?** Success, warning, error,
   pending, and "not supported" all need a branch.
7. **Does it hand me a reference, handle, or allocation I must release?** Who releases it,
   with which routine, from which IRQL?
8. **What context does it require?** A user process, a system thread, a file object?
   `PsGetCurrentProcess` is meaningless in a DPC.
9. **Can it fail under low memory, and is that failure handled without leaking?**
10. **Is it available on the oldest OS build I support?** A missing import fails driver
    load with `STATUS_ENTRYPOINT_NOT_FOUND` (0xC0000139), not at the first call.
11. **Does it need a lock, and does that lock's IRQL fit the caller?**
12. **Is the buffer size I pass derived from arithmetic that can overflow?** Use
    `RtlSizeTAdd`/`RtlSizeTMult` and reject `STATUS_INTEGER_OVERFLOW`.

## Failure modes and diagnosis

| Symptom | Likely cause | First command or fix |
|---|---|---|
| Bugcheck 0x1E `KMODE_EXCEPTION_NOT_HANDLED` | Unhandled SEH exception or AV in a driver frame | `!analyze -v`, `kv`, `lmvm <driver>`; fix the faulting instruction |
| Bugcheck 0x3B `SYSTEM_SERVICE_EXCEPTION` | AV in a routine reached from a system call, driver frame on stack | `!analyze -v`, `.frame`, inspect driver locals |
| Bugcheck 0x7E `SYSTEM_THREAD_EXCEPTION_NOT_HANDLED` | Exception in a system thread: `DriverEntry`, worker, system thread | `!thread`, `kv`; audit the thread routine's error paths |
| Bugcheck 0x8E `KERNEL_MODE_EXCEPTION_NOT_HANDLED` | Same class, different accounting | `!analyze -v`; parameter 2 is the exception record |
| Bugcheck 0x50 `PAGE_FAULT_IN_NONPAGED_AREA` | Freed or non-resident memory touched at `DISPATCH_LEVEL` | `!pool <addr>`, `!analyze -v`; find the free site |
| Bugcheck 0xA `IRQL_NOT_LESS_OR_EQUAL` | Paged or user memory touched at raised IRQL | `!irql`, `kv`; move the work to `PASSIVE_LEVEL` |
| Bugcheck 0xD1 `DRIVER_IRQL_NOT_LESS_OR_EQUAL` | Same as 0xA, attributed to a driver | `!analyze -v` names the driver; check each callee's IRQL limit |
| Bugcheck 0xC2 `BAD_POOL_CALLER` | Double free, wrong tag, or freeing non-pool memory | `!analyze -v`; parameter 1 gives the subcode |
| Bugcheck 0xCE `DRIVER_UNLOADED_WITHOUT_CANCELLING_PENDING_OPERATIONS` | Pending IRP or thread alive at unload | `!irpfind`, `!stacks 2`; drain before returning from unload |
| Bugcheck 0x9F `DRIVER_POWER_STATE_FAILURE` | IRP held across a power transition | `!irp`, `!devstack`; complete or cancel before the power IRP |
| Bugcheck 0xC4 (verifier) | Verifier caught an IRQL, pool, or DDI violation | `!verifier 3`; read the subcode in parameter 1 |
| `sc start` returns error 87 | `DriverEntry` returned `STATUS_INVALID_PARAMETER` (mapped to Win32) | Fix the parameter check; log the failing value |
| `sc start` returns error 577 or 1275 | Signature enforcement blocked the image | Signing policy; see `wk-driver-signing` |
| `sc stop` returns error 1052 | `DriverUnload` is NULL, so the service rejects the stop control | Provide an unload routine |
| Build error C2712 | `__try` in a function with C++ objects needing unwinding | Move SEH into a helper with no objects |
| Link error LNK2019 on `DriverEntry` | Missing `extern "C"`, or a changed entry point | Add `extern "C"`, verify `/ENTRY` |
| No output in DebugView | Default print filter suppresses `DPFLTR_INFO_LEVEL` | Use `DPFLTR_ERROR_LEVEL` or set the `Debug print filter` registry value |

## Safety, IRQL, and cleanup checklist

- [ ] Every `NTSTATUS` is tested with `NT_SUCCESS` or compared explicitly, including
      `IoCreateSymbolicLink`, `IoDeleteSymbolicLink`, and `ObReferenceObjectByHandle`.
- [ ] Every failure path releases exactly what the successful path acquired before it, in
      reverse order, with no partially initialized structure left reachable.
- [ ] `DriverEntry` cleans up after itself on failure, because `DriverUnload` will not run.
- [ ] `DriverUnload` unregisters callbacks, cancels timers and DPCs, drains queues, waits
      for threads, closes handles, deletes the symbolic link, deletes devices, and frees
      pool, in that order.
- [ ] No pageable code or data is touched at `DISPATCH_LEVEL` or above; `PAGED_CODE()` is
      present in every pageable routine and absent from every non-pageable one.
- [ ] No blocking call (`KeWaitForSingleObject`, `Zw*` file or registry call, mutex
      acquisition) at `DISPATCH_LEVEL` or above.
- [ ] IRQL is raised and lowered symmetrically; the acquire's saved IRQL is the value
      passed to the release.
- [ ] No C++ exceptions, RTTI, STL, or unoverloaded `new`/`delete` in the driver.
- [ ] No global object with a non-trivial constructor; no function-local static that needs
      dynamic initialization.
- [ ] Floating point is absent, or bracketed by `KeSaveFloatingPointState` and
      `KeRestoreFloatingPointState` below `DISPATCH_LEVEL`.
- [ ] Every user-controlled length is validated against a policy maximum before use and
      before any allocation size is computed.
- [ ] Assertions contain no side effects; Release was compiled and tested.

## Verification

```text
:: Debug and Release builds, from a WDK developer prompt
msbuild RulesDemo.vcxproj /p:Configuration=Debug /p:Platform=x64 /warnaserror
msbuild RulesDemo.vcxproj /p:Configuration=Release /p:Platform=x64 /warnaserror
:: Static analysis (or Analyze > Run Code Analysis in Visual Studio)
msbuild RulesDemo.vcxproj /p:Configuration=Release /p:Platform=x64 /p:RunCodeAnalysis=true

:: Install, start, stop (error 1052 means DriverUnload was NULL)
sc create RulesDemo type= kernel binPath= C:\drivers\RulesDemo.sys
sc start RulesDemo
sc stop RulesDemo

:: Verifier for this image only, then reboot
verifier /standard /driver RulesDemo.sys
verifier /querysettings
```

In a kernel debugger attached to the target:

```text
kd> !drvobj RulesDemo 2
Driver object (ffffa20b`1c2d3e40) is for:
 \Driver\RulesDemo
DriverEntry:   fffff805`12341000 RulesDemo!DriverEntry
DriverUnload:  fffff805`12341120 RulesDemo!RulesUnload
Dispatch array: MajorFunction[0..31]
   IRP_MJ_CREATE         RulesDemo!RulesCreateClose
   IRP_MJ_CLOSE          RulesDemo!RulesCreateClose
   IRP_MJ_DEVICE_CONTROL RulesDemo!RulesDeviceControl

kd> !poolused 4 RulesDemo
kd> !verifier 3
```

What to look for: `DriverUnload` is not null, the dispatch array contains only the handlers
you wrote plus `nt!IopInvalidDeviceRequest`, and `!poolused` shows a steady tag count across
repeated load/unload cycles. The `DRle` counter may be non-zero while the driver is loaded,
but it must return to the same baseline after every unload. After an unload cycle,
`!poolfind DRle` returning no hits proves the release path is complete.

Expected user-visible behavior: `sc start` succeeds, opening `\\.\RulesDemo` succeeds, the
IOCTL returns an incrementing `ULONG`, `sc stop` succeeds, and DebugView shows exactly two
lines per cycle, the first containing the service key path printed through `%wZ`.
