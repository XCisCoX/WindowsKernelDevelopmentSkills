---
name: wk-kernel-memory-pools
description: Use when a kernel driver must allocate or free memory: choosing non-paged or paged pool, calling ExAllocatePool2 with POOL_FLAG values or legacy ExAllocatePoolWithTag, overflow-safe size arithmetic, pool tags and leak hunting with !poolused and !poolfind, secure pools through ExCreatePool and ExAllocatePool3, and overloaded new/delete with an RAII wrapper. Includes C cleanup-label and C++ RAII versions.
---

# Kernel memory pools: allocate once, free once, tag everything

Every driver allocates. The pool APIs are simple, and that is the trap: the failure modes
are silent corruption, leaks that only disappear on reboot, and frees at the wrong IRQL.
This skill treats an allocation as a contract with four parts — who allocates, from which
pool, at which IRQL, and who frees with which tag — and shows how to keep all four true on
every path, including the error paths. The rule that prevents most pool disasters:
**one allocation, one owner, one free site, one tag, and a size that was checked for
overflow before the call**.

## When to use

- You are adding any `ExAllocatePool2`, `ExAllocatePool3`, `ExAllocatePoolWithTag`, or
  `ExAllocatePoolWithQuotaTag` call to a driver.
- You need to decide between paged and non-paged memory, or you are calling an allocator
  from a DPC, completion routine, or spinlock-protected region.
- A pool leak is suspected: memory grows per request, per handle, or per unload cycle.
- You are reviewing size arithmetic such as `count * sizeof(T)` that a user can influence.
- You must overload `new`/`delete` for kernel C++, or wrap pool ownership in RAII.
- You need a buffer that other kernel components cannot read (secure pool), or you must
  decide that secure pool is the wrong tool.

Do not use this skill for fixed-size object caches; see `wk-lookaside-lists` for lookaside
lists, and `wk-kernel-linked-lists` for intrusive list nodes that need no allocation at all.

## Prerequisites and related skills

- **Headers:** `<ntddk.h>` declares the pool APIs; `<ntintsafe.h>` declares the checked size helpers.
- **Availability:** `ExAllocatePool2` and `ExAllocatePool3` require Windows 10, version
  1909 or later as the load target. A driver that imports them on an older build fails to
  load with `STATUS_ENTRYPOINT_NOT_FOUND` (0xC0000139). Older targets use
  `ExAllocatePoolWithTag` and explicit zeroing.
- **IRQL:** paged pool below `DISPATCH_LEVEL`; non-paged pool up to `DISPATCH_LEVEL`;
  secure-pool management at `PASSIVE_LEVEL`.
- **Related skills:** `wk-kernel-programming-rules` (status handling and IRQL basics),
  `wk-irql` (the full legality matrix), `wk-lookaside-lists` (cached fixed-size blocks),
  `wk-kernel-libraries` (CRT and linker settings for C++), `wk-driver-verifier` (pool
  tracking and special pool).

## Mental model

The pool is a set of system-wide heaps, split by pageability. Each allocation is a block
with a header that records its size and tag; the header is what makes `ExFreePoolWithTag`
possible and what pool-tracking verification checks.

```text
   paged pool (pageable)                    non-paged pool (always resident)
   +--------------------------+             +--------------------------+
   | block hdr: size, tag     |             | block hdr: size, tag     |
   | payload bytes            |             | payload bytes            |
   +--------------------------+             +--------------------------+
        |                                        |
        | may be written to disk under pressure   | never leaves RAM
        | illegal to touch at DISPATCH_LEVEL      | legal at any IRQL <= DISPATCH
        v                                        v
   allocate/free only below DISPATCH        allocate/free up to DISPATCH
```

Four properties decide which pool and which call:

| Property | Paged pool | Non-paged pool |
|---|---|---|
| Pageable | Yes | No |
| Allocation IRQL | `< DISPATCH_LEVEL` | `<= DISPATCH_LEVEL` |
| Free IRQL | `< DISPATCH_LEVEL` | `<= DISPATCH_LEVEL` |
| Reachable from a DPC or ISR | No | Yes |
| Cost | Cheaper under memory pressure | Consumes committed RAM permanently |
| Default for | Long-lived data, buffers touched at `PASSIVE_LEVEL` | Spinlock-protected state, DPC data, DMA-adjacent structures, anything at raised IRQL |

Ownership rules that hold for every allocation:

1. **The allocator states the pool flag and the tag; the free site must match both.**
2. **Failure is reported by `NULL`**, or raised as `STATUS_INSUFFICIENT_RESOURCES` when
   `POOL_FLAG_RAISE_ON_FAILURE` or a raising quota variant is in use.
3. **An IRP's system buffer and MDL pages are not yours.** Free only what you allocated.
4. **Freed memory is not zeroed and is immediately reusable.** A stale pointer after free
   is a use-after-free, and the next allocation is likely to be the same address.
5. **A zero-length request is a caller bug.** Never allocate zero bytes; reject it.

## Key APIs and structures

### Allocation and free APIs

| API | Pool selection | Zeroed by default | Quota | Max IRQL | Notes |
|---|---|---|---|---|---|
| `ExAllocatePool2(POOL_FLAGS Flags, SIZE_T Bytes, ULONG Tag)` | `POOL_FLAG_PAGED`, `POOL_FLAG_NON_PAGED`, `POOL_FLAG_NON_PAGED_EXECUTABLE` | Yes | `POOL_FLAG_USE_QUOTA` | `DISPATCH_LEVEL` for non-paged, `APC_LEVEL` for paged | Preferred for new code; returns `NULL` on failure |
| `ExAllocatePool3(POOL_FLAGS, SIZE_T, ULONG, PCPOOL_EXTENDED_PARAMETER, ULONG)` | Same flags | Yes | Same flag | Same | Extensible: priority, NUMA node, secure pool; one array of `POOL_EXTENDED_PARAMETER` |
| `ExAllocatePoolWithTag(POOL_TYPE, SIZE_T, ULONG)` | `PagedPool`, `NonPagedPoolNx` | No | No | Same as above | Legacy; not for new code; requires explicit `RtlZeroMemory` |
| `ExAllocatePoolZero(POOL_TYPE, SIZE_T, ULONG)` | Same | Yes | No | Same | Inline wrapper that ORs the internal `POOL_ZERO_ALLOCATION` bit (0x400) into the pool type |
| `ExAllocatePoolWithQuotaTag(POOL_TYPE, SIZE_T, ULONG)` | Same | No | Yes | Same | **Raises** `STATUS_INSUFFICIENT_RESOURCES` on failure unless `POOL_QUOTA_FAIL_INSTEAD_OF_RAISE` is ORed in |
| `ExAllocatePoolWithTagPriority(POOL_TYPE, SIZE_T, ULONG, EX_POOL_PRIORITY)` | Same | No | No | Same | `EX_POOL_PRIORITY` values: `LowPoolPriority` 0, `NormalPoolPriority` 16, `HighPoolPriority` 32, plus special-pool overrun/underrun variants (8, 9, 24, 25, 40, 41) |
| `ExFreePool(PVOID)` | Any | n/a | n/a | Same as matching allocate | No tag check; use only when the tag is genuinely unknown |
| `ExFreePoolWithTag(PVOID, ULONG)` | Must match | n/a | n/a | Same as matching allocate | Preferred; a wrong tag is a bugcheck, not a leak |
| `ExFreePool2(PVOID, ULONG, PCPOOL_EXTENDED_PARAMETER, ULONG)` | Secure pool | n/a | n/a | `PASSIVE_LEVEL` for secure pool | With no extended parameters it forwards to `ExFreePool` and fails for secure allocations |

`POOL_FLAGS` is a 64-bit flag set; combine values with `|`. Exactly one of the pageability
flags below must be present.

| Flag | Effect | Must be recognized |
|---|---|---|
| `POOL_FLAG_PAGED` | Allocate pageable memory | Yes |
| `POOL_FLAG_NON_PAGED` | Allocate resident memory (no execute permission on x64/ARM64) | Yes |
| `POOL_FLAG_NON_PAGED_EXECUTABLE` | Resident with execute permission; only for legacy code that runs from pool | Yes |
| `POOL_FLAG_USE_QUOTA` | Charge the allocation to the calling process | Yes |
| `POOL_FLAG_UNINITIALIZED` | Skip zeroing; the block contains old data | Yes |
| `POOL_FLAG_CACHE_ALIGNED` | Best-effort cache-line alignment | Yes |
| `POOL_FLAG_RAISE_ON_FAILURE` | Raise `STATUS_INSUFFICIENT_RESOURCES` instead of returning `NULL` | Yes |
| `POOL_FLAG_SPECIAL_POOL` | Allocate from special pool to catch overruns | No (optional hint) |

Do not hard-code flag values; use the names from `wdm.h`. Zeroing is on by default, and
`POOL_FLAG_UNINITIALIZED` is the only opt-out.

### Pool tags

- **A tag is four bytes, usually characters in the 0x20-0x7E range**, for example
  `'KPol'`. Tags outside that range make Driver Verifier complain.
- **The characters appear reversed in memory and in dumps.** MSVC evaluates the
  multi-character literal `'abcd'` as `0x61626364`, which is stored little-endian as
  `d c b a`. Writing `'loPK'` displays as `KPol`; writing `'KPol'` displays as `loPK`.
  Choose the literal whose reversal reads well, and comment the intended display form.
- **Use one tag per driver, defined once in a header**, never another component's tag, and
  never `'NONE'` or `'????'`. A tag is the fastest way to attribute a leak.
- **The tag passed to free must match the allocation.** Tracking verification checks it;
  without verification a mismatch corrupts accounting silently.

### IRQL matrix

| Operation | Paged pool | Non-paged pool | Notes |
|---|---|---|---|
| Allocate | `<= APC_LEVEL` | `<= DISPATCH_LEVEL` | A paged allocation at `DISPATCH_LEVEL` bugchecks 0xA/0xD1 |
| Free | `<= APC_LEVEL` | `<= DISPATCH_LEVEL` | Freeing paged memory at `DISPATCH_LEVEL` touches a possibly non-resident header |
| `ExCreatePool`, `ExDestroyPool` | n/a | `PASSIVE_LEVEL` | Secure pool only |
| `ExSecurePoolUpdate` | n/a | `PASSIVE_LEVEL` | Secure pool only |
| Lookaside allocate/free | `<= APC_LEVEL` for paged lists | `<= DISPATCH_LEVEL` | See `wk-lookaside-lists` |

### Size arithmetic that cannot overflow

`ExAllocatePool2` takes a `SIZE_T`, but the multiplication that produces it usually comes
from a `ULONG` count and a `sizeof`. Do the arithmetic with checked helpers and reject
before allocating:

```c
#include <ntintsafe.h>

NTSTATUS
KspAllocateArray(_In_ SIZE_T ElementSize,
                 _In_ SIZE_T ElementCount,
                 _In_ POOL_FLAGS Flags,
                 _In_ ULONG Tag,
                 _Outptr_ PVOID* Buffer)
{
    NTSTATUS status;
    SIZE_T bytes = 0;

    *Buffer = NULL;
    if (ElementSize == 0 || ElementCount == 0) {
        return STATUS_INVALID_PARAMETER;
    }
    status = RtlSizeTMult(ElementSize, ElementCount, &bytes);
    if (!NT_SUCCESS(status)) {
        return status;                      /* STATUS_INTEGER_OVERFLOW */
    }
    if (bytes > KSP_MAX_ALLOCATION) {
        return STATUS_INVALID_PARAMETER;    /* policy cap, before pool pressure */
    }
    *Buffer = ExAllocatePool2(Flags, bytes, Tag);
    return (*Buffer != NULL) ? STATUS_SUCCESS : STATUS_INSUFFICIENT_RESOURCES;
}
```

The same rule applies to `offset + length` and to `(count + 1) * sizeof(WCHAR)`: use
`RtlSizeTAdd` and `RtlSizeTMult`, and treat `STATUS_INTEGER_OVERFLOW` (0xC0000095) as
invalid input, not as a transient failure.

## Step-by-step

1. **Name the owner.** One function allocates and one function frees, or a single context
   structure owns the pointer and is destroyed in one place. Check: every `ExAllocatePool2`
   has a matching `ExFreePoolWithTag` on every path, including failures after it.
2. **Choose the pool from the IRQL of the *consumer*, not the allocator.** If a DPC,
   completion routine, or spinlock-protected path will touch the data, it is non-paged.
   Check: every access site is at or below the allocation's IRQL limit.
3. **Pick the flags.** `POOL_FLAG_PAGED` or `POOL_FLAG_NON_PAGED`, plus `POOL_FLAG_USE_QUOTA`
   when a user handle owns the lifetime. Check: no allocation relies on zeroing while
   passing `POOL_FLAG_UNINITIALIZED`.
4. **Compute the size with checked arithmetic** and cap it against a policy maximum before
   the call. Check: no `count * sizeof(T)` remains in the argument expression.
5. **Check for `NULL`** (or wrap the call in `__try` when using `POOL_FLAG_RAISE_ON_FAILURE`
   or a raising quota variant) and propagate `STATUS_INSUFFICIENT_RESOURCES`. Check: the
   error path frees everything allocated earlier.
6. **Zero explicitly if the flag says uninitialized**, or when the block is handed to user
   mode. Check: no uninitialized byte can reach a caller.
7. **Store the tag next to the pointer** in the owning structure if more than one tag is in
   use, so the free site cannot guess wrong. Check: allocation tag == free tag, textually.
8. **Free exactly once, in reverse order of acquisition**, and null the pointer afterwards.
   Check: no early `return` between allocate and free that skips cleanup.
9. **Test the failure paths** with low-resource simulation (Driver Verifier flag 0x8), then
   after unload confirm zero blocks remain with `!poolfind <Tag>`. Check: no leak on the
   failure path, and the count returns to the pre-load baseline after every cycle.

## Complete example

### C version: cleanup labels and checked sizes

```c
/* KspPool.c - a batch that owns three allocations and unwinds in reverse. */
#include <ntddk.h>
#include <ntintsafe.h>

#define KSP_POOL_TAG        'loPK'      /* reads as "KPol" in a pool dump */
#define KSP_MAX_ENTRIES     256
#define KSP_NAME_CHARS      64          /* including the terminator */
#define KSP_MAX_ALLOCATION  (1024 * 1024)

typedef struct _KSP_ENTRY {
    ULONG Id;
    ULONG Flags;
} KSP_ENTRY, *PKSP_ENTRY;

typedef struct _KSP_BATCH {
    PKSP_ENTRY Entries;
    PULONG     Offsets;
    PWSTR      Names;
    ULONG      EntryCount;
} KSP_BATCH, *PKSP_BATCH;

/* Frees in reverse acquisition order and tolerates partially built batches. */
VOID
KspFreeBatch(_Inout_ PKSP_BATCH Batch)
{
    if (Batch == NULL) {
        return;
    }
    if (Batch->Names != NULL) {
        ExFreePoolWithTag(Batch->Names, KSP_POOL_TAG);
        Batch->Names = NULL;
    }
    if (Batch->Offsets != NULL) {
        ExFreePoolWithTag(Batch->Offsets, KSP_POOL_TAG);
        Batch->Offsets = NULL;
    }
    if (Batch->Entries != NULL) {
        ExFreePoolWithTag(Batch->Entries, KSP_POOL_TAG);
        Batch->Entries = NULL;
    }
    Batch->EntryCount = 0;
}

NTSTATUS
KspBuildBatch(_In_ ULONG EntryCount,
              _In_ PCUNICODE_STRING SourceName,
              _Out_ PKSP_BATCH Batch)
{
    NTSTATUS status;
    SIZE_T entryBytes = 0;
    SIZE_T offsetBytes = 0;
    SIZE_T nameChars = 0;
    SIZE_T nameBytes = 0;
    ULONG index;
    ULONG copyBytes;

    if (Batch == NULL) {
        return STATUS_INVALID_PARAMETER;
    }
    RtlZeroMemory(Batch, sizeof(*Batch));
    if (EntryCount == 0 || EntryCount > KSP_MAX_ENTRIES || SourceName == NULL) {
        return STATUS_INVALID_PARAMETER;
    }

    status = RtlSizeTMult(EntryCount, sizeof(KSP_ENTRY), &entryBytes);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    status = RtlSizeTMult(EntryCount, sizeof(ULONG), &offsetBytes);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    status = RtlSizeTMult(EntryCount, KSP_NAME_CHARS, &nameChars);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    status = RtlSizeTMult(nameChars, sizeof(WCHAR), &nameBytes);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    if (entryBytes > KSP_MAX_ALLOCATION || offsetBytes > KSP_MAX_ALLOCATION ||
        nameBytes > KSP_MAX_ALLOCATION) {
        return STATUS_INVALID_PARAMETER;
    }

    Batch->Entries = (PKSP_ENTRY)ExAllocatePool2(POOL_FLAG_PAGED, entryBytes,
                                                 KSP_POOL_TAG);
    if (Batch->Entries == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }
    Batch->Offsets = (PULONG)ExAllocatePool2(POOL_FLAG_PAGED, offsetBytes,
                                             KSP_POOL_TAG);
    if (Batch->Offsets == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }
    Batch->Names = (PWSTR)ExAllocatePool2(POOL_FLAG_PAGED, nameBytes,
                                          KSP_POOL_TAG);
    if (Batch->Names == NULL) {
        status = STATUS_INSUFFICIENT_RESOURCES;
        goto Cleanup;
    }

    for (index = 0; index < EntryCount; index++) {
        Batch->Entries[index].Id = index;
        Batch->Entries[index].Flags = 0;
        Batch->Offsets[index] = 0;
    }

    /* Copy at most KSP_NAME_CHARS - 1 characters; the block arrived zeroed. */
    copyBytes = SourceName->Length;
    if (copyBytes > (KSP_NAME_CHARS - 1) * sizeof(WCHAR)) {
        copyBytes = (KSP_NAME_CHARS - 1) * sizeof(WCHAR);
    }
    if (copyBytes != 0 && SourceName->Buffer != NULL) {
        RtlCopyMemory(Batch->Names, SourceName->Buffer, copyBytes);
    }

    Batch->EntryCount = EntryCount;
    return STATUS_SUCCESS;

Cleanup:
    KspFreeBatch(Batch);
    return status;
}
```

### C++ version: overloaded new/delete and an RAII owner

```cpp
/* KspPool.h - include from .cpp files only; no exceptions, no STL. */
#pragma once
#include <ntddk.h>

#define KSP_POOL_TAG 'loPK'

/* Placement new: construct into memory someone else allocated. */
inline void* __cdecl operator new(size_t, void* Where) noexcept { return Where; }
inline void __cdecl operator delete(void*, void*) noexcept {}

/* Throwing is impossible in the kernel, so this tag selects the NULL-returning form. */
struct ksp_nothrow_t { explicit ksp_nothrow_t() = default; };
constexpr ksp_nothrow_t ksp_nothrow{};

inline void* __cdecl operator new(size_t Size, POOL_FLAGS Flags, ULONG Tag) noexcept
{
    if (Size == 0) {
        return nullptr;
    }
    return ExAllocatePool2(Flags & ~POOL_FLAG_RAISE_ON_FAILURE, Size, Tag);
}

inline void* __cdecl operator new(size_t Size, POOL_FLAGS Flags, ULONG Tag,
                                  const ksp_nothrow_t&) noexcept
{
    if (Size == 0) {
        return nullptr;
    }
    return ExAllocatePool2(Flags & ~POOL_FLAG_RAISE_ON_FAILURE, Size, Tag);
}

/* Every operator delete must free with the tag used at allocation. */
inline void __cdecl operator delete(void* Pointer, size_t) noexcept
{
    if (Pointer != nullptr) {
        ExFreePoolWithTag(Pointer, KSP_POOL_TAG);
    }
}

/* Move-only owner: no copy, deterministic release, explicit detach. */
template <typename T>
class KPoolPtr {
public:
    KPoolPtr() noexcept : m_Data(nullptr), m_Tag(0) {}
    ~KPoolPtr() { Reset(); }

    KPoolPtr(const KPoolPtr&) = delete;
    KPoolPtr& operator=(const KPoolPtr&) = delete;

    KPoolPtr(KPoolPtr&& Other) noexcept : m_Data(Other.m_Data), m_Tag(Other.m_Tag)
    {
        Other.m_Data = nullptr;
        Other.m_Tag = 0;
    }

    KPoolPtr& operator=(KPoolPtr&& Other) noexcept
    {
        if (this != &Other) {
            Reset();
            m_Data = Other.m_Data;
            m_Tag = Other.m_Tag;
            Other.m_Data = nullptr;
            Other.m_Tag = 0;
        }
        return *this;
    }

    NTSTATUS Allocate(_In_ POOL_FLAGS Flags, _In_ ULONG Tag,
                      _In_ SIZE_T Count = 1) noexcept
    {
        NTSTATUS status;
        SIZE_T bytes = 0;

        if (m_Data != nullptr) {
            return STATUS_INVALID_DEVICE_STATE;
        }
        if (Count == 0) {
            return STATUS_INVALID_PARAMETER;
        }
        status = RtlSizeTMult(sizeof(T), Count, &bytes);
        if (!NT_SUCCESS(status)) {
            return status;
        }
        m_Data = static_cast<T*>(ExAllocatePool2(Flags, bytes, Tag));
        if (m_Data == nullptr) {
            return STATUS_INSUFFICIENT_RESOURCES;
        }
        m_Tag = Tag;
        return STATUS_SUCCESS;
    }

    void Reset() noexcept
    {
        if (m_Data != nullptr) {
            ExFreePoolWithTag(m_Data, m_Tag);
            m_Data = nullptr;
            m_Tag = 0;
        }
    }

    T* Detach() noexcept
    {
        T* pointer = m_Data;
        m_Data = nullptr;
        m_Tag = 0;
        return pointer;
    }

    T* Get() const noexcept { return m_Data; }
    T* operator->() const noexcept { return m_Data; }
    T& operator*() const noexcept { return *m_Data; }
    explicit operator bool() const noexcept { return m_Data != nullptr; }

private:
    T* m_Data;
    ULONG m_Tag;
};
```

Usage:

```cpp
struct KSP_REQUEST {
    KSP_REQUEST() : Id(0), Bytes(0) {}
    ULONG Id;
    ULONG Bytes;
};

NTSTATUS
KspCreateRequest(_In_ ULONG Id, _Out_ KPoolPtr<KSP_REQUEST>* Out)
{
    NTSTATUS status = Out->Allocate(POOL_FLAG_NON_PAGED, KSP_POOL_TAG);
    if (!NT_SUCCESS(status)) {
        return status;
    }
    (*Out)->Id = Id;        /* the block arrived zeroed from ExAllocatePool2 */
    return STATUS_SUCCESS;
}
```

For a type with a constructor, use placement new over `Get()` and call the destructor
before the owner releases the block. `KPoolPtr` deliberately does not adopt arbitrary
pointers; add a `static Attach(T*, ULONG Tag)` factory when existing code must hand
ownership over.

## Secure pools

A secure pool is memory that other kernel components cannot read, enforced by the
hypervisor at virtual trust level 1. It exists to hold secrets (keys, tokens) that must
survive a compromised kernel. Constraints:

- **Requires Virtualization Based Security (VBS) and the hypervisor.** Without them the
  calls fail; plan a feature check and a fallback path.
- **Windows 10, version 1909 and later**, and only partially documented. Treat the
  interfaces as settled but sparsely specified, and test on the exact target build.
- **Management happens at `PASSIVE_LEVEL`.** Create, update, and destroy are not for DPCs.
- **Free needs its own call.** `ExFreePool` and `ExFreePoolWithTag` fail on a secure
  allocation; use `ExFreePool2` with the pool handle in an extended parameter.

```c
#define KSP_SECURE_TAG 'crSK'

NTSTATUS
KspSecureStore(_In_ PCUCHAR Secret, _In_ ULONG SecretBytes)
{
    HANDLE poolHandle = NULL;
    POOL_EXTENDED_PARAMS_SECURE_POOL secure = {0};
    POOL_EXTENDED_PARAMETER parameter = {0};
    PVOID allocation = NULL;
    NTSTATUS status;

    status = ExCreatePool(POOL_CREATE_FLG_SECURE_POOL | POOL_CREATE_FLG_USE_GLOBAL_POOL,
                          KSP_SECURE_TAG, NULL, &poolHandle);
    if (!NT_SUCCESS(status)) {
        return status;                  /* STATUS_NOT_SUPPORTED without VBS */
    }

    secure.SecurePoolHandle = poolHandle;
    secure.Buffer = NULL;               /* no initial content */
    secure.Cookie = (ULONG_PTR)poolHandle;
    secure.SecurePoolFlags = SECURE_POOL_FLAGS_FREEABLE;

    parameter.Type = PoolExtendedParameterSecurePool;
    parameter.Optional = FALSE;
    parameter.SecurePoolParams = &secure;

    allocation = ExAllocatePool3(POOL_FLAG_NON_PAGED, SecretBytes, KSP_SECURE_TAG,
                                 &parameter, 1);
    if (allocation == NULL) {
        ExDestroyPool(poolHandle);
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    status = ExSecurePoolUpdate(poolHandle, KSP_SECURE_TAG, allocation,
                                secure.Cookie, 0, SecretBytes, (PVOID)Secret);

    secure.Buffer = NULL;
    ExFreePool2(allocation, KSP_SECURE_TAG, &parameter, 1);
    ExDestroyPool(poolHandle);
    return status;
}
```

For ordinary secrets that only need to be scrubbed before release, skip secure pool
entirely: allocate from non-paged pool, and clear with `RtlSecureZeroMemory`, which the
compiler is not allowed to elide (a plain `RtlZeroMemory` before a free can be removed as a
dead store). Freed pool is never zeroed by the system.

## Variations and advanced use

- **Lookaside lists for fixed-size hot objects.** A `LOOKASIDE_LIST_EX` caches freed blocks
  per processor; free through the list, delete it on unload (`wk-lookaside-lists`).
- **Contiguous memory and large buffers.** `MmAllocateContiguousMemorySpecifyCache` and
  `MmAllocatePagesForMdlEx` are for DMA and multi-megabyte transfers, not general storage;
  they are non-paged, expensive, and released with the matching `MmFree*` call.
- **One allocation instead of four.** When several buffers share a lifetime, allocate one
  block and hand out interior pointers: one free site, one tag, no partial-failure leaks.

## Finding leaks with pool tags

```text
kd> !poolused
kd> !poolused 2
kd> !poolused 4
kd> !poolfind KPol
kd> !pool ffff9c0a12345678
kd> !poolval ffff9c0a12345678
```

- `!poolused` reports per-tag usage; the numeric flags select the pool class (paged,
  non-paged, or both) and the sort order, and the printed header repeats the selection.
  `!poolused 4` sorts by usage, which surfaces a growing tag immediately.
- `!poolfind KPol` walks the pool and lists every block carrying the tag, with address,
  size, and pool type. **After unload it must return nothing**; before unload it must match
  the number of live objects you believe you have.
- `!pool <address>` decodes one block's header: tag, size, and pool type; `!poolval <address>`
  validates it. A validation failure is corruption, not a logic bug.
- **Pool tracking in Driver Verifier** (`verifier /standard` includes it) records every
  allocation with its tag and reports leaks at unload with the tag and the allocating
  stack. This is the fastest route from "leak" to "line of code".

Reading a leak: growth per IOCTL means a handler path missed a free; growth per handle means
a per-file context is not released at `IRP_MJ_CLEANUP`; growth after unload points at the
unload path itself.

## Failure modes and diagnosis

| Symptom | Likely cause | First command or fix |
|---|---|---|
| Bugcheck 0xC2 `BAD_POOL_CALLER` | Double free, free with the wrong tag, or free of a non-pool pointer | `!analyze -v`; parameter 1 is the subcode; `!pool <addr>` |
| Bugcheck 0x19 `BAD_POOL_HEADER` | Header overwritten by an underrun or an overrun | `!poolval`, enable special pool |
| Bugcheck 0x50 `PAGE_FAULT_IN_NONPAGED_AREA` | Used a freed block, or touched paged pool at raised IRQL | `!pool <addr>`, special pool to catch the free |
| Bugcheck 0xA / 0xD1 | Paged allocation or free at `DISPATCH_LEVEL` or in a DPC | `!irql`, `kv`; move the allocation below `DISPATCH_LEVEL` |
| Bugcheck 0xC4 with a pool subcode | Verifier caught a tag mismatch, double free, or overrun | `!verifier 3`; read parameter 1 |
| Bugcheck 0x139 `KERNEL_SECURITY_CHECK_FAILURE` | Pool metadata corrupted; fast-fail on a bad list entry | `!analyze -v`, `!poolval`; reproduce under special pool |
| `STATUS_INSUFFICIENT_RESOURCES` under load only | Allocation failure not handled, or a runaway retry loop | Low-resource simulation (`verifier /flags 0x8`); bound the retries |
| Memory grows per request | Missing free on an error path | `!poolused 4` during the run, `!poolfind` after |
| Memory grows per handle | Per-file context not released at `IRP_MJ_CLEANUP` | Check the close path, not the pool code |
| Secret visible in a dump | Secret stored in ordinary pool and never scrubbed | `RtlSecureZeroMemory` on release, or secure pool |

## Safety, IRQL, and cleanup checklist

- [ ] Every allocation has one owner, one free site, and a free tag textually identical to
      the allocation tag.
- [ ] Every `NT_SUCCESS`/`NULL` check is followed by a path that frees everything acquired
      earlier, in reverse order.
- [ ] Paged pool is allocated and freed only below `DISPATCH_LEVEL`; code reachable from a
      DPC or spinlock uses non-paged pool.
- [ ] `ExAllocatePoolWithTag` results are explicitly zeroed when they can reach a caller.
- [ ] Sizes are computed with `RtlSizeTAdd`/`RtlSizeTMult` and capped by policy before the
      call; zero-length requests are rejected.
- [ ] Quota-charged memory is freed in the same process context that allocated it.
- [ ] IRP system buffers, MDL pages, and other components' buffers are never freed.
- [ ] Blocks returned to a lookaside list never reach `ExFreePoolWithTag`, and vice versa.
- [ ] Secrets are cleared with `RtlSecureZeroMemory`, or kept in a secure pool.
- [ ] Unload frees every remaining block and then proves it with `!poolfind`.
- [ ] Failure paths are exercised with low-resource simulation, not just review.

## Verification

```text
:: Build both configurations at /W4
msbuild KspPool.vcxproj /p:Configuration=Debug /p:Platform=x64 /warnaserror
msbuild KspPool.vcxproj /p:Configuration=Release /p:Platform=x64 /warnaserror

:: Pool tracking and special pool for this image, then reboot
verifier /standard /driver KspPool.sys
verifier /querysettings
```

Leak test loop: start the driver, note the baseline, run 10,000 create/destroy cycles from
a client, unload, and compare.

```text
kd> !poolfind KPol
kd> !poolused 4
```

Expected: `!poolfind KPol` lists the live batch blocks while the driver is loaded, then
returns nothing after the driver unloads. `!poolused 4` must show the `KPol` count back at
its pre-test value; a count that rises with every cycle names the leaking path directly.

Also test the failure path: enable low-resource simulation (`verifier /flags 0x8`) and run
the same loop. The driver must return `STATUS_INSUFFICIENT_RESOURCES` for some requests,
never leak a partially built batch, and never crash. A count of `KPol` blocks that stays at
zero after each failed request proves `KspFreeBatch` is reached from every
`goto Cleanup`.
