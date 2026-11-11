---
name: wk-kernel-strings
description: Use when a driver must store, copy, compare, convert, format, or print text: UNICODE_STRING byte-versus-character semantics, RtlInitUnicodeString and its literal-buffer trap, RtlCopyUnicodeString and append/compare behavior, ntstrsafe RtlStringCb/RtlStringCch return values, ANSI and integer conversions, and DbgPrint %wZ, %Z and %ws specifiers. Includes a complete safe user-string copy routine.
---

# Kernel strings: counted buffers, not C strings

The kernel's native text type is `UNICODE_STRING`, a counted UTF-16 buffer with a byte
length, a byte capacity, and a pointer. Almost every mistake with it comes from treating
those two lengths as interchangeable, treating the buffer as NUL-terminated, or writing
through a pointer that was never meant to be written. The mental model is a slice: an
owner, an offset zero, a byte count, and a capacity that belongs to somebody else. The
one rule that prevents the worst bugs: **`Length` is bytes, excludes the terminator, and
is not a guarantee that the buffer can be written; `MaximumLength` and the allocation
owner decide that.**

## When to use

- You are defining an IOCTL, registry value, device name, or log message that carries text
  between user mode and a driver.
- You must copy a caller-supplied `UNICODE_STRING` and keep it beyond the call.
- A string is truncated, garbled, or replaced with `?` characters, or `%wZ` prints
  nothing.
- You are converting between wide and ANSI, or between text and integers, in either
  direction.
- You need to know the exact return value of an `ntstrsafe` routine and whether truncation
  counts as success.
- You are reviewing code that calls `wcscpy`, `wcscat`, `wcslen`, `strcpy`, or `sprintf`,
  or that writes into a string initialized from a wide string literal.

Do not use this skill for IRP buffer mechanics; see `wk-irp-buffering` for buffered and
direct I/O, and `wk-kernel-exceptions` for the exception model behind probe and catch.

## Prerequisites and related skills

- **Headers:** `<ntddk.h>` (or `<fltKernel.h>`) declares `UNICODE_STRING` and the `Rtl*`
  string routines. Safe string routines need `<ntstrsafe.h>` and the `ntstrsafe.lib`
  library, which the WDK driver project does not link by default.
- **IRQL:** all allocation and conversion routines here run at `PASSIVE_LEVEL` or
  `APC_LEVEL`. `DbgPrint`, `RtlCompareUnicodeString`, and `RtlCopyMemory` are safe up to
  `DISPATCH_LEVEL` **only if every buffer involved is non-paged**.
- **Related skills:** `wk-kernel-programming-rules` (status handling),
  `wk-kernel-memory-pools` (allocation and tags), `wk-irp-buffering` (input/output buffer
  methods), `wk-kernel-exceptions` (`ProbeForRead` and SEH), `wk-kernel-tracing`
  (`DbgPrintEx` levels and filters).

## Mental model

```text
  UNICODE_STRING
  +----------------+     Length         = 10 bytes  (5 UTF-16 code units, no terminator)
  | Length     (2) |     MaximumLength  = 12 bytes  (capacity, includes terminator room)
  | MaximumLength  |     Buffer         -> +---+---+---+---+---+----+
  | Buffer     (8) |                      | H | e | l | l | o | \0 |
  +----------------+                      +---+---+---+---+---+----+
                                           0   2   4   6   8   10   <- byte offsets
```

Invariants that hold for every well-formed string:

- **`Length <= MaximumLength`, and both are byte counts.** A 5-character string has
  `Length == 10` on Windows, not 5.
- **`Length` is even for valid UTF-16** (surrogate pairs are two code units, still even
  bytes). An odd `Length` means corruption or a hostile caller; reject it.
- **`Length` excludes the terminator; `MaximumLength` is the capacity of `Buffer`.** A
  terminator may or may not exist, and may or may not fit.
- **`Length == 0` does not imply `Buffer != NULL`.** An empty string can be a NULL buffer;
  `MaximumLength` is then meaningless.
- **A `UNICODE_STRING` does not own `Buffer`.** Whoever allocated the buffer frees it, and
  a string built by `RtlInitUnicodeString` never owns anything.
- **`MaximumLength` is a `USHORT`.** The largest capacity value is 65535 bytes;
  the largest usable even capacity with terminator room is 65534 (`0xFFFE`), which allows
  at most 65532 bytes (`0xFFFC`) of content.

Three consequences drive the rest of this skill:

1. **Capacity arithmetic must be done before writing**, because the copy routines that
   return `VOID` truncate silently.
2. **The terminator is a convention, not a property.** Functions that need one (`wcscmp`,
   `%ws`, any `CHAR`-based API) must be given a buffer you know is terminated.
3. **Allocation and ownership are separate from representation.** `RtlFreeUnicodeString`
   frees only what the `Rtl*` conversion routines allocated; it must never see a buffer
   from `ExAllocatePool2`.

### Which family to use

| Need | Family | Header / lib | Failure signal |
|---|---|---|---|
| Initialize a view over an existing literal or buffer | `RtlInitUnicodeString`, `RtlInitEmptyUnicodeString` | `ntddk.h` | none (void) |
| Copy, append, compare counted strings | `RtlCopyUnicodeString`, `RtlAppend*`, `RtlCompare*` | `ntddk.h` | `NTSTATUS` or `BOOLEAN`, except copy |
| Build or copy text into a fixed buffer with bounds | `RtlStringCb*W`, `RtlStringCch*W` | `ntstrsafe.h` + `ntstrsafe.lib` | `STATUS_BUFFER_OVERFLOW` |
| Convert between ANSI, Unicode, and integers | `Rtl*StringTo*`, `Rtl*ToUnicodeString` | `ntddk.h` | `NTSTATUS` |
| Print to the debugger | `DbgPrint`, `DbgPrintEx` | `ntddk.h` | `ULONG` count, usually ignored |

## Key APIs and structures

### Initialization, copy, append, compare

| Routine | Signature essentials | Returns | Contract and trap |
|---|---|---|---|
| `RtlInitUnicodeString` | `(PUNICODE_STRING Dest, PCWSTR Source)` | `VOID` | Points `Dest->Buffer` at `Source` (no copy), sets `Length` to `wcslen(Source)*2` and `MaximumLength` to `Length + sizeof(WCHAR)`. **Never write through the result**, and never let it outlive `Source`. |
| `RtlInitEmptyUnicodeString` | `(PUNICODE_STRING Dest, PWCH Buffer, USHORT MaximumLength)` | `VOID` | Sets `Length = 0`, stores your buffer and capacity. The correct way to prepare a destination you will fill. |
| `RtlInitAnsiString` | `(PANSI_STRING Dest, PCSZ Source)` | `VOID` | Same pointing semantics for `ANSI_STRING` (`Length`, `MaximumLength`, `Buffer`). |
| `RtlCopyUnicodeString` | `(PUNICODE_STRING Dest, PCUNICODE_STRING Source)` | `VOID` | Copies at most `Dest->MaximumLength` bytes and sets `Dest->Length`. **Truncates silently and does not terminate**; compare lengths yourself. |
| `RtlAppendUnicodeStringToString` | `(PUNICODE_STRING Dest, PCUNICODE_STRING Source)` | `NTSTATUS` | `STATUS_SUCCESS` or `STATUS_BUFFER_TOO_SMALL`; needs free capacity in `Dest->MaximumLength` and allocates nothing. |
| `RtlAppendUnicodeToString` | `(PUNICODE_STRING Dest, PCWSTR Source)` | `NTSTATUS` | Same as above for a NUL-terminated source. |
| `RtlCompareUnicodeString` | `(PCUNICODE_STRING A, PCUNICODE_STRING B, BOOLEAN CaseInsensitive)` | `LONG` | Lexicographic result (`<0`, `0`, `>0`), **not** an `NTSTATUS`; `TRUE` for case-insensitive. |
| `RtlEqualUnicodeString` | `(PCUNICODE_STRING A, PCUNICODE_STRING B, BOOLEAN CaseInsensitive)` | `BOOLEAN` | Equality only; the argument order does not matter. |
| `RtlPrefixUnicodeString` | `(PCUNICODE_STRING Prefix, PCUNICODE_STRING String, BOOLEAN CaseInsensitive)` | `BOOLEAN` | `TRUE` when `Prefix` is a prefix of `String` (including equal); order matters. |
| `RtlUpcaseUnicodeString` | `(PUNICODE_STRING Dest, PCUNICODE_STRING Source, BOOLEAN AllocateDest)` | `NTSTATUS` | With `AllocateDest = TRUE` it allocates (paged pool) and the result must be freed with `RtlFreeUnicodeString`; with `FALSE`, `Dest` must already have capacity. |
| `RtlDowncaseUnicodeString` | same as above | `NTSTATUS` | Same ownership rule. |
| `RtlFreeUnicodeString` | `(PUNICODE_STRING String)` | `VOID` | **Only** for buffers allocated by the `Rtl*` allocate/conversion routines. Sets `Buffer` to NULL. |
| `RtlDuplicateUnicodeString` | `(ULONG Flags, PCUNICODE_STRING Source, PUNICODE_STRING Dest)` | `NTSTATUS` | Allocates a private copy; flags `RTL_DUPLICATE_UNICODE_STRING_NULL_TERMINATE` (1) and `RTL_DUPLICATE_UNICODE_STRING_ALLOCATE_NULL_STRING` (2). Free with `RtlFreeUnicodeString`. Windows 8 and later. |

### Safe string functions with their exact return values

All of these live in `<ntstrsafe.h>` and require `ntstrsafe.lib`. They never overrun the
destination, and they always NUL-terminate what they produce, even when truncating.

| Routine | Signature essentials | `STATUS_SUCCESS` | `STATUS_BUFFER_OVERFLOW` (0x80000005) | `STATUS_INVALID_PARAMETER` (0xC000000D) |
|---|---|---|---|---|
| `RtlStringCbCopyW` | `(PWSTR Dest, size_t cbDest, PCWSTR Src)` | Whole source copied and terminated | Destination filled, truncated, terminated | NULL `Dest`/`Src`, `cbDest == 0`, or `cbDest` above the internal maximum |
| `RtlStringCchCopyW` | `(PWSTR Dest, size_t cchDest, PCWSTR Src)` | Whole source copied | Truncated to `cchDest - 1` characters | Bad arguments, `cchDest == 0` |
| `RtlStringCbCatW` | `(PWSTR Dest, size_t cbDest, PCWSTR Src)` | Concatenated | Truncated at the old terminator | Destination not terminated within `cbDest` |
| `RtlStringCchCatW` | `(PWSTR Dest, size_t cchDest, PCWSTR Src)` | Concatenated | Truncated | Bad arguments |
| `RtlStringCbPrintfW` | `(PWSTR Dest, size_t cbDest, PCWSTR Fmt, printf arguments)` | Formatted and terminated | Formatted prefix, truncated, terminated | Bad arguments or unsupported conversion |
| `RtlStringCbVPrintfW` | `(PWSTR Dest, size_t cbDest, PCWSTR Fmt, va_list Args)` | Same as above | Same as above | Same as above |
| `RtlStringCbLengthW` | `(PCWSTR Str, size_t cbMax, size_t *pcbLength)` | Terminator found; bytes returned **excluding** it | Never | No terminator within `cbMax`, or `cbMax`/`Str` invalid |
| `RtlStringCbCopyUnicodeString` | `(PUNICODE_STRING Dest, PCUNICODE_STRING Src)` | Fits; `Dest->Length` updated | Truncated into `Dest->MaximumLength` | Bad arguments or `Dest->MaximumLength == 0` |

**The trap:** `STATUS_BUFFER_OVERFLOW` has warning severity, so `NT_SUCCESS(status)` is
`TRUE`. A handler that checks only `NT_SUCCESS` will return truncated data to the caller
while claiming success. Compare against `STATUS_SUCCESS` whenever truncation is a failure,
and check `Information`/`Length` against the caller's request.

Also useful: `RtlStringCbCopyNW(Dest, cbDest, Src, cbSrc)` copies at most `cbSrc` bytes
from a source that need not be terminated, and is the right call for a slice of a larger
buffer.

### Conversions

| Routine | Signature essentials | Returns | Ownership and notes |
|---|---|---|---|
| `RtlUnicodeStringToAnsiString` | `(PANSI_STRING Dest, PCUNICODE_STRING Src, BOOLEAN Allocate)` | `NTSTATUS` | If `Allocate`, free with `RtlFreeAnsiString`. Uses the system ANSI code page; unmappable characters become `?`, and a string too large for `ANSI_STRING` returns `STATUS_BUFFER_OVERFLOW`. |
| `RtlAnsiStringToUnicodeString` | `(PUNICODE_STRING Dest, PCANSI_STRING Src, BOOLEAN Allocate)` | `NTSTATUS` | If `Allocate`, free with `RtlFreeUnicodeString`. |
| `RtlFreeAnsiString` / `RtlFreeOemString` | `(PANSI_STRING String)` | `VOID` | Matching free for the two conversion destinations. |
| `RtlIntegerToUnicodeString` | `(ULONG Value, ULONG Base, PUNICODE_STRING Dest)` | `NTSTATUS` | `Base` 0 means decimal, 2-36 accepted; `Dest` needs capacity for 33 characters plus terminator in the worst case (base 2, 32-bit value). |
| `RtlUnicodeStringToInteger` | `(PCUNICODE_STRING Src, ULONG Base, PULONG Value)` | `NTSTATUS` | `Base` 0 auto-detects a `0x` prefix; leading signs are handled. `Value` is undefined on failure, so check the status before use. |
| `RtlInt64ToUnicodeString` | `(ULONGLONG Value, ULONG Base, PUNICODE_STRING Dest)` | `NTSTATUS` | Same capacity rule for 64-bit values (65 characters in base 2). |
| `RtlUnicodeToUTF8N` / `RtlUTF8ToUnicodeN` | declared in `ntifs.h`; take explicit byte counts and an output length | `NTSTATUS` | Use these when the protocol is UTF-8; the ANSI conversions are locale-dependent and lossy. |

### Printf specifiers: debugger output versus ntstrsafe output

| Specifier | `DbgPrint` / `DbgPrintEx` expects | Meaning |
|---|---|---|
| `%s` | `PCSTR` | ANSI string, read until NUL |
| `%ws` | `PCWSTR` | Wide string, read until NUL |
| `%wZ` | `PCUNICODE_STRING` | Counted wide string, prints exactly `Length` bytes; no terminator needed |
| `%Z` | `PCANSI_STRING` | Counted ANSI string |
| `%p` | `PVOID` | Pointer |
| `%08X` | `ULONG` | Status or flags in fixed-width hex |
| `%I64u`, `%I64d` | `ULONGLONG`, `LONGLONG` | 64-bit decimal |
| `%u`, `%d`, `%x` | `ULONG`, `LONG` | 32-bit values; cast `SIZE_T` explicitly |

Rules for both families:

- **`%wZ` takes a pointer.** Passing the `UNICODE_STRING` by value makes the printer read
  a struct as a pointer and fault or print garbage.
- **`%wZ` needs no terminator**, which is exactly why it is the right way to log a counted
  string; `%ws` needs a terminated `WCHAR*`.
- **In the `RtlStringCbPrintfW` family, `%s` means a wide string**, the opposite of
  `DbgPrint`. Passing a `CHAR*` to `%s` there produces mojibake or a fault.
- **`%wZ` is a debugger specifier only.** The ntstrsafe printf family has no
  `UNICODE_STRING` conversion; terminate the string and pass `%s`, or append it with
  `RtlAppendUnicodeStringToString`.
- **The format string must match the arguments exactly.** The kernel printer walks the
  argument list by the format; a mismatch reads the stack as data and can bugcheck.
- **`DbgPrint` output is capped at 512 bytes per call** and adds no newline, so include
  `\n` and split long messages.

## Step-by-step

1. **Decide who owns the bytes.** Three options: a view over an existing buffer
   (`RtlInitUnicodeString`), a caller-provided destination
   (`RtlInitEmptyUnicodeString`), or a private allocation (`ExAllocatePool2` plus a copy).
   Check: every path that can outlive the call uses a private allocation.
2. **Compute capacity in bytes with a policy cap.** Content `<= 65532` bytes, capacity
   `content + sizeof(WCHAR)`, both `USHORT`-safe. Check: no `USHORT` cast happens before
   the range test.
3. **Allocate and zero the terminator.** Allocate the capacity, copy `Length` bytes, write
   `L'\0'` at `Buffer[Length / sizeof(WCHAR)]`. Check: the terminator index is a character
   index, not a byte offset.
4. **Set the three fields consistently:** `Buffer`, `Length` (content bytes), `MaximumLength`
   (capacity bytes). Check: `Length <= MaximumLength - sizeof(WCHAR)` for terminated strings.
5. **Compare and search with counted routines**, not `wcscmp`, `wcsstr`, or `wcslen`. Check:
   no C runtime string call appears in the file except on buffers you just terminated.
6. **Convert at the boundary, not in the core.** Keep one representation internally; convert
   to ANSI only for a legacy consumer, and check the conversion status. Check: `?`
   characters in output are explained by an encoding mismatch, not by a bug.
7. **Print with `%wZ` for counted strings and `%ws` only for terminated ones.** Check:
   DebugView shows the expected text and no character after it.
8. **Free with the matching allocator.** `ExFreePoolWithTag` for pool, `RtlFreeUnicodeString`
   or `RtlFreeAnsiString` for `Rtl*` allocations, nothing for views. Check: `!poolfind`
   shows no leftover tag after unload.

## Complete example

A self-contained string module: bounded duplication, a user-mode capture routine with probe
and catch, a formatting helper, a trace helper, and an IOCTL handler that uses all of them.

```c
/* KspStrings.c - string handling that cannot overrun or leak. */
#include <ntddk.h>
#include <ntstrsafe.h>

#define KSP_STRING_TAG      'srSK'      /* reads as "KSrs" in a pool dump */
#define KSP_MAX_CONTENT     0xFFFC      /* bytes of UTF-16, leaves terminator room */
#define KSP_DEVICE_TYPE     0x8334

#define KSP_IOCTL_SET_NAME \
    CTL_CODE(KSP_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)

/* Input: a header followed by ByteCount bytes of UTF-16, terminator optional. */
typedef struct _KSP_NAME_INPUT {
    ULONG ByteCount;
    ULONG Reserved;
    WCHAR Text[1];
} KSP_NAME_INPUT, *PKSP_NAME_INPUT;

/* Output: the same shape, filled by the driver. */
typedef struct _KSP_NAME_OUTPUT {
    ULONG ByteCount;
    ULONG Reserved;
    WCHAR Text[1];
} KSP_NAME_OUTPUT, *PKSP_NAME_OUTPUT;

/* Allocate a private buffer of ContentBytes plus a terminator. */
static NTSTATUS
KspAllocateBuffer(_Out_ PUNICODE_STRING String, _In_ ULONG ContentBytes)
{
    USHORT capacity;

    RtlZeroMemory(String, sizeof(*String));
    if (ContentBytes > KSP_MAX_CONTENT) {
        return STATUS_INVALID_PARAMETER;
    }
    capacity = (USHORT)(ContentBytes + sizeof(WCHAR));
    String->Buffer = (PWCH)ExAllocatePool2(POOL_FLAG_PAGED, capacity,
                                           KSP_STRING_TAG);
    if (String->Buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    String->Buffer[ContentBytes / sizeof(WCHAR)] = L'\0';
    String->Length = (USHORT)ContentBytes;
    String->MaximumLength = capacity;
    return STATUS_SUCCESS;
}

VOID
KspFreeString(_Inout_ PUNICODE_STRING String)
{
    if (String == NULL) {
        return;
    }
    if (String->Buffer != NULL) {
        ExFreePoolWithTag(String->Buffer, KSP_STRING_TAG);
    }
    RtlZeroMemory(String, sizeof(*String));
}

/* Duplicate a trusted, well-formed counted string. */
NTSTATUS
KspDuplicateString(_Out_ PUNICODE_STRING Destination,
                   _In_ PCUNICODE_STRING Source)
{
    NTSTATUS status;

    if (Destination == NULL || Source == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlZeroMemory(Destination, sizeof(*Destination));
    if (Source->Length == 0) {
        return STATUS_SUCCESS;              /* empty: NULL buffer is legal */
    }
    if (Source->Buffer == NULL || (Source->Length & 1) != 0) {
        return STATUS_INVALID_PARAMETER;
    }
    if (Source->Length > KSP_MAX_CONTENT) {
        return STATUS_INVALID_PARAMETER;    /* policy limit, not a memory limit */
    }

    status = KspAllocateBuffer(Destination, Source->Length);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    RtlCopyMemory(Destination->Buffer, Source->Buffer, Source->Length);
    return STATUS_SUCCESS;
}

/* Copy a UNICODE_STRING that came from a user-mode caller. */
NTSTATUS
KspCaptureUserString(_In_ PCUNICODE_STRING UserString,
                     _Out_ PUNICODE_STRING Destination)
{
    UNICODE_STRING snapshot;
    NTSTATUS status = STATUS_SUCCESS;

    if (UserString == NULL || Destination == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlZeroMemory(Destination, sizeof(*Destination));
    RtlZeroMemory(&snapshot, sizeof(snapshot));

    /* Pass 1: snapshot the descriptor. It may live on a different thread's stack. */
    __try {
        ProbeForRead(UserString, sizeof(snapshot),
                     TYPE_ALIGNMENT(UNICODE_STRING));
        snapshot = *UserString;
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        return GetExceptionCode();
    }

    if (snapshot.Length == 0) {
        return STATUS_SUCCESS;
    }
    if (snapshot.Buffer == NULL || (snapshot.Length & 1) != 0) {
        return STATUS_INVALID_PARAMETER;
    }
    /* Length is the only field trusted from user mode; MaximumLength is not. */
    if (snapshot.Length > KSP_MAX_CONTENT) {
        return STATUS_INVALID_PARAMETER;
    }

    status = KspAllocateBuffer(Destination, snapshot.Length);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    /* Pass 2: copy exactly the snapshot's bytes, never re-reading Length. */
    __try {
        ProbeForRead(snapshot.Buffer, snapshot.Length, sizeof(WCHAR));
        RtlCopyMemory(Destination->Buffer, snapshot.Buffer, snapshot.Length);
    } __except (EXCEPTION_EXECUTE_HANDLER) {
        status = GetExceptionCode();
    }
    if (!NT_SUCCESS(status)) {
        KspFreeString(Destination);
        return status;
    }
    return STATUS_SUCCESS;
}

/* Format into a freshly allocated UNICODE_STRING. */
NTSTATUS
KspFormatString(_Out_ PUNICODE_STRING Destination,
                _In_ ULONG MaximumBytes,
                _In_ PCWSTR Format,
                ...)
{
    NTSTATUS status;
    va_list arguments;
    size_t length = 0;
    ULONG capacity;

    if (Destination == NULL || Format == NULL ||
        MaximumBytes < 2 * sizeof(WCHAR)) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlZeroMemory(Destination, sizeof(*Destination));

    if (MaximumBytes > KSP_MAX_CONTENT + sizeof(WCHAR)) {
        MaximumBytes = KSP_MAX_CONTENT + sizeof(WCHAR);
    }
    capacity = MaximumBytes & ~1UL;         /* keep MaximumLength even */

    Destination->Buffer = (PWCH)ExAllocatePool2(POOL_FLAG_PAGED, capacity,
                                                KSP_STRING_TAG);
    if (Destination->Buffer == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    Destination->MaximumLength = (USHORT)capacity;
    Destination->Buffer[0] = L'\0';

    va_start(arguments, Format);
    status = RtlStringCbVPrintfW(Destination->Buffer, capacity, Format,
                                 arguments);
    va_end(arguments);

    if (status == STATUS_BUFFER_OVERFLOW) {
        KspFreeString(Destination);         /* truncation is a failure here */
        return STATUS_BUFFER_OVERFLOW;
    }
    if (!NT_SUCCESS(status)) {
        KspFreeString(Destination);
        return status;
    }

    status = RtlStringCbLengthW(Destination->Buffer, capacity, &length);
    if (!NT_SUCCESS(status)) {
        KspFreeString(Destination);
        return status;
    }
    Destination->Length = (USHORT)length;   /* bytes, terminator excluded */
    return STATUS_SUCCESS;
}

VOID
KspPrintString(_In_ PCSTR Label, _In_ PCUNICODE_STRING String)
{
    if (Label == NULL || String == NULL) {
        return;
    }
    if (String->Buffer == NULL) {
        DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
                   "KSP: %s: <empty>\n", Label);
        return;
    }
    DbgPrintEx(DPFLTR_IHVDRIVER_ID, DPFLTR_INFO_LEVEL,
               "KSP: %s: \"%wZ\" (Length=%u, MaximumLength=%u)\n",
               Label, String, String->Length, String->MaximumLength);
}

/* IOCTL: validate a counted UTF-16 payload and echo a formatted summary. */
NTSTATUS
KspSetNameIoctl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PIO_STACK_LOCATION stack = IoGetCurrentIrpStackLocation(Irp);
    ULONG inputLength = stack->Parameters.DeviceIoControl.InputBufferLength;
    ULONG outputLength = stack->Parameters.DeviceIoControl.OutputBufferLength;
    PKSP_NAME_INPUT input = (PKSP_NAME_INPUT)Irp->AssociatedIrp.SystemBuffer;
    PKSP_NAME_OUTPUT output = (PKSP_NAME_OUTPUT)Irp->AssociatedIrp.SystemBuffer;
    UNICODE_STRING supplied;
    UNICODE_STRING message;
    NTSTATUS status;

    UNREFERENCED_PARAMETER(DeviceObject);
    RtlZeroMemory(&supplied, sizeof(supplied));
    RtlZeroMemory(&message, sizeof(message));

    if (stack->Parameters.DeviceIoControl.IoControlCode != KSP_IOCTL_SET_NAME) {
        status = STATUS_INVALID_DEVICE_REQUEST;
        goto Complete;
    }
    if (input == NULL || output == NULL) {
        status = STATUS_INVALID_PARAMETER;
        goto Complete;
    }
    if (inputLength < FIELD_OFFSET(KSP_NAME_INPUT, Text)) {
        status = STATUS_INVALID_PARAMETER;
        goto Complete;
    }
    if ((input->ByteCount & 1) != 0 ||
        input->ByteCount > KSP_MAX_CONTENT ||
        input->ByteCount > inputLength - FIELD_OFFSET(KSP_NAME_INPUT, Text)) {
        status = STATUS_INVALID_PARAMETER;
        goto Complete;
    }

    /* Build a view over the system buffer, then take a private copy. */
    supplied.Buffer = input->Text;
    supplied.Length = (USHORT)input->ByteCount;
    supplied.MaximumLength = supplied.Length;

    status = KspDuplicateString(&message, &supplied);
    if (!NT_SUCCESS(status)) {
        goto Complete;
    }
    KspPrintString("input", &message);
    KspFreeString(&message);

    /* Numbers only: %wZ is a debugger specifier, not an ntstrsafe one. */
    status = KspFormatString(&message, 128, L"name bytes=%u length=%u",
                             input->ByteCount, (ULONG)supplied.Length);
    if (!NT_SUCCESS(status)) {
        goto Complete;
    }

    if (outputLength < FIELD_OFFSET(KSP_NAME_OUTPUT, Text) ||
        outputLength - FIELD_OFFSET(KSP_NAME_OUTPUT, Text) < message.Length) {
        status = STATUS_BUFFER_TOO_SMALL;
        goto Complete;
    }

    /* The system buffer is reused for output: input is no longer needed. */
    RtlCopyMemory(output->Text, message.Buffer, message.Length);
    output->ByteCount = message.Length;
    output->Reserved = 0;
    Irp->IoStatus.Information =
        FIELD_OFFSET(KSP_NAME_OUTPUT, Text) + message.Length;
    status = STATUS_SUCCESS;

Complete:
    KspFreeString(&message);
    RtlZeroMemory(&supplied, sizeof(supplied));  /* a view owns no pool buffer */
    Irp->IoStatus.Status = status;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return status;
}
```

Two details in this example are worth copying deliberately: the capture routine never
re-reads `Length` after the snapshot, which closes the double-fetch window, and the format
helper treats `STATUS_BUFFER_OVERFLOW` as failure instead of forwarding truncated text.
`KspCaptureUserString` covers the other common protocol shape, where the input buffer
embeds a `UNICODE_STRING` whose `Buffer` points back into user memory; the IOCTL above
avoids that shape on purpose by putting a byte count in front of the payload, which cannot be
reinterpreted by a 32-bit client.

## Variations and advanced use

- **View versus copy.** `RtlInitUnicodeString` over a wide string literal is ideal for a
  parameter that is used within the same call and never written. The moment the string must
  survive the call, or be written to, switch to a private allocation; mixing the two is
  the source of the read-only-memory bugcheck.
- **Fixed-size stack buffer with `RtlStringCbPrintfW`.** A 128-byte local buffer plus a
  `STATUS_BUFFER_OVERFLOW` branch handles most log messages without an allocation. Keep
  stack buffers under about 256 bytes; the kernel stack is 12 KB on x86 and 24 KB on x64.
- **Grow-on-demand pattern for unknown output size.** Call the formatting routine with the
  caller's buffer; on `STATUS_BUFFER_OVERFLOW`, report `STATUS_BUFFER_TOO_SMALL` and the
  required size through `Information`, and let user mode retry with a larger buffer.
- **One internal representation.** Choose UTF-16 everywhere inside the driver and convert
  at the IOCTL or registry boundary. Mixed ANSI and Unicode paths double the test matrix
  and produce the `?`-character reports.

## Do and do not

| Do | Do not |
|---|---|
| Size buffers in bytes and include terminator room | Treat `Length` as a character count |
| Set `MaximumLength` to the buffer's real capacity | Leave `MaximumLength` at 0 and then append |
| Use `RtlEqualUnicodeString` / `RtlCompareUnicodeString` | Call `wcscmp` on a counted, unterminated buffer |
| Use `RtlStringCbCopyW` / `RtlStringCchCopyW` for copies | Use `wcscpy`, `wcscat`, `strcpy`, or `sprintf` |
| Test `status == STATUS_SUCCESS` when truncation matters | Accept `NT_SUCCESS` on a possibly truncated result |
| Take a private copy of caller data you keep | Cache a pointer to another component's transient buffer |
| Free `Rtl*`-allocated strings with `RtlFree*` | Pass an `RtlAnsiStringToUnicodeString` buffer to `ExFreePoolWithTag` |
| Write into a string you allocated | Write through a string built by `RtlInitUnicodeString` over a literal |
| Terminate before calling `%ws`, `wcscmp`, or any `CHAR`-based API | Pass a counted string to a function that expects a terminator |
| Cap user lengths against a policy maximum first | Trust `MaximumLength` supplied by user mode |
| Keep string work at `PASSIVE_LEVEL` with paged buffers | Pass a paged buffer to `DbgPrint` at `DISPATCH_LEVEL` |
| Convert to ANSI once, at the boundary, and check the status | Store user-facing text as ANSI internally |

## Failure modes and diagnosis

| Symptom | Likely cause | First command or fix |
|---|---|---|
| Bugcheck 0xBE `ATTEMPTED_WRITE_TO_READONLY_MEMORY` | Wrote through a buffer created by `RtlInitUnicodeString` over a literal | Convert to `RtlInitEmptyUnicodeString` plus an allocated buffer |
| Bugcheck 0x50 `PAGE_FAULT_IN_NONPAGED_AREA` | Paged string buffer touched at `DISPATCH_LEVEL`, or a freed buffer printed | `!pool <addr>`, `!analyze -v`; move the logging below `DISPATCH_LEVEL` |
| Bugcheck 0xA / 0xD1 at a string call | Paged `UNICODE_STRING` data accessed at raised IRQL | `!irql`, `kv`; copy to non-paged memory before raising |
| Bugcheck 0x1E or 0x8E inside `wcslen`/`wcscmp` | Unterminated buffer treated as a C string | Use counted routines; verify the terminator is written |
| Bugcheck 0x7F from a dispatch routine | Large `WCHAR` local array exhausted the kernel stack | Move the buffer to pool |
| Text replaced with `?` or mojibake | ANSI conversion or `%s`/`%ws` mismatch | Print with `%wZ`; confirm the encoding and the specifier |
| Reply truncated but the IOCTL returns success | `STATUS_BUFFER_OVERFLOW` accepted as success | Compare against `STATUS_SUCCESS`; set `Information` correctly |
| `STATUS_BUFFER_OVERFLOW` from `RtlStringCbPrintfW` | Destination too small for the formatted result | Compute the needed size or return `STATUS_BUFFER_TOO_SMALL` |
| `STATUS_BUFFER_TOO_SMALL` from `RtlAppendUnicodeStringToString` | `MaximumLength` has no free capacity | Allocate capacity up front; views cannot grow |
| `STATUS_INVALID_PARAMETER` from `RtlStringCbCopyW` | NULL destination, zero size, or size above the internal maximum | Check arguments before the call |
| `STATUS_INVALID_PARAMETER` from `RtlUnicodeStringToInteger` | Empty string, bad digit, or unsupported base | Validate the text; treat `Value` as undefined on failure |
| Pool grows per request | Forgot `KspFreeString`/`RtlFreeUnicodeString` on an error path | `!poolfind KSrs`; free on every exit |
| Bugcheck 0xC2 `BAD_POOL_CALLER` | Freed an `Rtl*`-allocated string with `ExFreePoolWithTag`, or double free | Match the allocator to the string |
| Garbled IOCTL text | Client sent ANSI but the driver parsed UTF-16, or a 32-bit client sent pointers | Fix the protocol: explicit byte counts, no pointers |

## Safety, IRQL, and cleanup checklist

- [ ] Every `UNICODE_STRING` that escapes the current call owns a pool buffer, and its
      owner is named in a comment.
- [ ] `Length` and `MaximumLength` are byte counts; `Length` is even and excludes the
      terminator; `MaximumLength` includes terminator room.
- [ ] Content is capped at 65532 bytes before any `USHORT` cast, so capacity arithmetic
      cannot wrap.
- [ ] The terminator is written at `Buffer[Length / sizeof(WCHAR)]`, not at a byte offset.
- [ ] User-supplied `Length` is validated (even, non-zero only with a buffer, within the
      policy cap) and `MaximumLength` from user mode is ignored.
- [ ] The capture path reads the descriptor once, copies exactly that many bytes, and does
      not re-read the length afterwards.
- [ ] No `wcscpy`, `wcscat`, `wcslen`, `strcpy`, `sprintf`, or `RtlCopyUnicodeString` into
      a destination whose capacity is unknown.
- [ ] Truncation is detected and reported, not hidden behind `NT_SUCCESS`.
- [ ] `RtlFreeUnicodeString` is used only for `Rtl*`-allocated strings; pool strings use
      `ExFreePoolWithTag` with the same tag.
- [ ] `%wZ` is passed a pointer, and any `%ws` buffer is known to be terminated.
- [ ] String work that may block on paging happens below `DISPATCH_LEVEL`; logging at `DISPATCH_LEVEL`
      uses non-paged buffers only.
- [ ] Large buffers are on the heap, not on the stack.

## Verification

```text
:: Build both configurations; ntstrsafe.lib must be in the link line
msbuild KspStrings.vcxproj /p:Configuration=Debug /p:Platform=x64 /warnaserror
msbuild KspStrings.vcxproj /p:Configuration=Release /p:Platform=x64 /warnaserror
```

Debugger checks on a loaded driver:

```text
kd> dt nt!_UNICODE_STRING ffffa20b1c2d3e40   :: Length, MaximumLength, Buffer
kd> !pool ffffa20b1c2d3e40                   :: tag and size of the Buffer block
kd> !poolfind KSrs                       :: any leaked string buffers
```

Expected debugger output after one load/unload cycle with no requests: `!poolfind KSrs`
returns nothing. After a request, the tag may appear until the handler completes; a
persistent hit means a handler path skipped `KspFreeString`.

Functional checks with a user-mode client that sends `KSP_IOCTL_SET_NAME`:

1. `ByteCount = 10`, valid text: IOCTL returns `STATUS_SUCCESS`, DebugView prints
   `KSP: input: "Hello" (Length=10, MaximumLength=12)`, and the reply contains
   `name bytes=10 length=10`.
2. `ByteCount = 11` (odd): returns `STATUS_INVALID_PARAMETER` (Win32 87) and allocates
   nothing.
3. `ByteCount = 0xFFFE`: returns `STATUS_INVALID_PARAMETER` because the policy cap is
   checked before any allocation.
4. Output buffer of 8 bytes: returns `STATUS_BUFFER_TOO_SMALL` with `Information == 0`.
5. A second request with a `UNICODE_STRING` whose `Buffer` points at an unmapped address:
   returns `STATUS_ACCESS_VIOLATION` (0xC0000005), and the driver stays loaded.

Finally, run the client in a loop for 10,000 iterations under Driver Verifier with special
pool, then check `!poolfind KSrs` and the tag count in `!poolused`: both must return to the
pre-test baseline.
