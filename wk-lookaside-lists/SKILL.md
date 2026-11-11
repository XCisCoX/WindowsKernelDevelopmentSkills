---
name: wk-lookaside-lists
description: Use when a driver allocates and frees the same fixed-size object at high rate and pool contention or fragmentation shows up. Covers GENERAL_LOOKASIDE and LOOKASIDE_LIST_EX, the classic versus Ex initialization and allocation APIs, per-processor caching, depth behaviour, paged versus non-paged IRQL limits, custom allocate/free callbacks, constructor and destructor discipline, draining at unload, and 0xC2, 0x19, 0x50 diagnosis.
---

# Lookaside lists: a per-processor free list for one fixed-size object

Every `ExAllocatePool2` call is a small negotiation with the memory manager: choose a size class, take the pool lock for that class, walk the free list, possibly split a block, stamp a header, update accounting. At `PASSIVE_LEVEL` that cost disappears next to the I/O the allocation supports. In a path that runs thousands of times per second, or at `DISPATCH_LEVEL` where every processor competes for the same pool lock, it becomes the profile. A lookaside list removes the negotiation for the one case where removal is safe: an object of a single fixed size, allocated and freed over and over. The mechanism is a free list the driver owns, split per processor, where a free pushes a block with one atomic operation and an allocation pops it the same way. The rule that prevents the classic disasters: **a block that leaves the list must return to that same list exactly once, before the list is deleted, and its contents mean nothing until a constructor initializes them**.

## When to use

- One object type of one fixed size is allocated and freed at high rate: request objects, work descriptors, per-I/O contexts, fixed-size packet buffers, table nodes.
- The allocation happens at `DISPATCH_LEVEL` or inside a spinlock-protected path where the pool lock would serialize the machine.
- Pool contention appears in a profile or in `!poolused` churn, and the size is already constant.
- A per-I/O object must be recycled without touching the global pool on every request.
- A driver is already tracking a free list by hand and would rather use the supported one.
- `0xC2` or `0x19` bugchecks appear after an object is freed and the free path must be audited.
- Do not use for: variable-size buffers (`wk-kernel-memory-pools`); deferred work objects (`wk-work-items`, which recycles its own); blocks allocated once at load (`wk-kernel-memory-pools`); objects large enough that a cache of them wastes memory.

## Prerequisites and related skills

- Kernel C with `<ntddk.h>` or `<wdm.h>`. The classic API exists from Windows 2000; `ExInitializeLookasideListEx` and `LOOKASIDE_LIST_EX` require Windows Vista or later; `ExAllocatePool2` inside a custom callback requires Windows 10 2004 (build 19041) or later with a matching WDK.
- The list structure itself must live in non-paged memory even when it serves paged blocks. A device extension or a driver global works; a paged-pool allocation does not.
- Initialization and deletion run at `PASSIVE_LEVEL`. Allocation and free run at `PASSIVE_LEVEL` for a paged list and up to `DISPATCH_LEVEL` for a non-paged list.
- Sibling skills: `wk-kernel-memory-pools` (pool flags, tags, quota, `ExAllocatePool2`), `wk-irql` (what may be called at each level), `wk-kernel-linked-lists` (the intrusive list types these objects sit in), `wk-spinlocks` (protecting the live list), `wk-driver-anatomy` (the unload sequence the drain fits into), `wk-driver-verifier` (special pool interaction).

## Mental model

The pool's job is to serve any size, so its data structures are general: size classes, free lists per class, a lock per pool, headers, accounting, and a validation step on every free. A lookaside list is the special case where all of that generality is unnecessary. The size is known at initialization, the block count is bounded, and the caller is the only owner. What remains is a stack of free blocks and a pointer to it.

```text
   ExAllocatePool2(size)                    ExAllocateFromLookasideListEx(list)
        |                                          |
        v                                          v
   size class lookup  -> lock pool            pop current CPU's list (atomic)
        |                                          |          |
        v                                          | hit      | miss
   free list walk, header stamp, accounting        |          v
        |                                          |     allocate callback
        v                                          |     (or plain pool)
   block  ---------------------------------------->+-----------> block
```

The cache is **per processor**, and that is the entire performance story. Each processor keeps its own singly-linked list of free blocks, pushed and popped with an interlocked operation on the list head, so two processors allocating at the same moment never touch the same cache line. Three consequences follow, and they are what surprises people:

1. **A block follows its free, not its allocation.** Allocating on CPU 0 and freeing on CPU 1 leaves the block cached on CPU 1. Nothing breaks, but a workload that always allocates in one context and frees in another gets no reuse on the allocation side; the cache migrates instead.
2. **Depth is not yours to pin.** `Depth` is reserved and must be passed as 0. The manager chooses a starting depth and then raises or lowers the limit from the observed miss rate, so the memory a lookaside holds is bounded but not fixed — roughly `MaximumDepth × Size × active processors`. That is why a multi-kilobyte object in a lookaside is a poor idea.
3. **The fast path trusts the caller completely.** No zeroing, no size check on free, no quota charge, no pointer validation. A wrong pointer, a double free, or a block freed with `ExFreePool` corrupts a list head or a pool header, and the bugcheck usually arrives long after the mistake.

`GENERAL_LOOKASIDE` is the shared state behind both API generations: the list head, `Depth`, `MaximumDepth`, `TotalAllocates`, `AllocateMisses`, `TotalFrees`, `FreeMisses`, `Type`, `Tag`, `Size`, and the two callbacks. The classic API wraps it in `PAGED_LOOKASIDE_LIST` and `NPAGED_LOOKASIDE_LIST`, one type per pool kind, and the pool kind is implied by which function you call. The newer API wraps it in `LOOKASIDE_LIST_EX` and takes the pool kind as an argument, so one type serves both. The counters are readable diagnostics and unsynchronized snapshots; the list head is not to be touched.

## Key APIs and structures

Classic API, one function family per pool kind:

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `ExInitializePagedLookasideList(&List, Allocate, Free, Flags, Size, Tag, Depth)` | Initialize a paged list | `PASSIVE_LEVEL` | `Depth` must be 0 |
| `ExInitializeNPagedLookasideList(&List, Allocate, Free, Flags, Size, Tag, Depth)` | Initialize a non-paged list | `PASSIVE_LEVEL` | `Depth` must be 0 |
| `ExAllocateFromPagedLookasideList(&List)` | Allocate `Size` bytes | `PASSIVE_LEVEL` | `NULL` if a miss and the allocation failed |
| `ExAllocateFromNPagedLookasideList(&List)` | Allocate `Size` bytes | `<= DISPATCH_LEVEL` | Legal inside a spinlock |
| `ExFreeToPagedLookasideList(&List, Entry)` | Return a block | `PASSIVE_LEVEL` | `Entry` must have come from this list |
| `ExFreeToNPagedLookasideList(&List, Entry)` | Return a block | `<= DISPATCH_LEVEL` | Same |
| `ExDeletePagedLookasideList(&List)` | Release cached blocks | `PASSIVE_LEVEL` | Returns `VOID`; cannot fail |
| `ExDeleteNPagedLookasideList(&List)` | Release cached blocks | `PASSIVE_LEVEL` | Same |

Extended API, one function set for both pool kinds:

| API | Purpose | IRQL | Notes |
|---|---|---|---|
| `ExInitializeLookasideListEx(&List, AllocateEx, FreeEx, PoolType, Flags, Size, Tag, Depth)` | Initialize | `PASSIVE_LEVEL` | Returns `NTSTATUS`; check it |
| `ExAllocateFromLookasideListEx(&List)` | Allocate `Size` bytes | `PASSIVE` for paged, `<= DISPATCH` for non-paged | `NULL` on failure unless a raise flag is set |
| `ExFreeToLookasideListEx(&List, Entry)` | Return a block | Same as above | `Entry` must have come from this list |
| `ExDeleteLookasideListEx(&List)` | Release cached blocks | `PASSIVE_LEVEL` | Returns `VOID` |

Exact declarations of the extended set:

```c
NTSTATUS ExInitializeLookasideListEx(
    _Out_ PLOOKASIDE_LIST_EX Lookaside, _In_opt_ PALLOCATE_FUNCTION_EX Allocate,
    _In_opt_ PFREE_FUNCTION_EX Free, _In_ POOL_TYPE PoolType, _In_ ULONG Flags,
    _In_ SIZE_T Size, _In_ ULONG Tag, _In_ USHORT Depth);
PVOID ExAllocateFromLookasideListEx(_Inout_ PLOOKASIDE_LIST_EX Lookaside);
VOID ExFreeToLookasideListEx(_Inout_ PLOOKASIDE_LIST_EX Lookaside, _In_ PVOID Entry);
VOID ExDeleteLookasideListEx(_Inout_ PLOOKASIDE_LIST_EX Lookaside);
```

The callback types differ between the generations, and only the newer one receives the list:

```c
/* classic: no way to reach driver state */
typedef PVOID (NTAPI *PALLOCATE_FUNCTION)(
    _In_ POOL_TYPE PoolType, _In_ SIZE_T NumberOfBytes, _In_ ULONG Tag);
typedef VOID (NTAPI *PFREE_FUNCTION)(_In_ PVOID Buffer);

/* extended: the list is passed back, so driver context is reachable */
typedef PVOID (NTAPI *PALLOCATE_FUNCTION_EX)(
    _In_ POOL_TYPE PoolType, _In_ SIZE_T NumberOfBytes, _In_ ULONG Tag,
    _Inout_ PLOOKASIDE_LIST_EX Lookaside);
typedef VOID (NTAPI *PFREE_FUNCTION_EX)(
    _In_ PVOID Buffer, _Inout_ PLOOKASIDE_LIST_EX Lookaside);
```

Flags for the two initializers:

| Flag | API | Meaning |
|---|---|---|
| `0` | both | Allocation failure returns `NULL` |
| `POOL_RAISE_IF_ALLOCATION_FAILURE` | classic | Raise `STATUS_INSUFFICIENT_RESOURCES` instead of returning `NULL` (Windows 8+) |
| `POOL_NX_ALLOCATION` | classic | Non-paged blocks come from non-executable pool |
| `POOL_QUOTA_FAIL_INSTEAD_OF_RAISE` | classic | With a custom callback and a quota pool type; quota failure returns `NULL` |
| `EX_LOOKASIDE_LIST_EX_FLAGS_RAISE_ON_FAIL` | extended | Raise instead of returning `NULL` |
| `EX_LOOKASIDE_LIST_EX_FLAGS_FAIL_NO_RAISE` | extended | Requires a custom callback; ORs `POOL_QUOTA_FAIL_INSTEAD_OF_RAISE` into the pool type it receives |

The two extended flags are mutually exclusive; `ExInitializeLookasideListEx` returns `STATUS_INVALID_PARAMETER` when both are set. For the pool type prefer `NonPagedPoolNx` (or `POOL_NX_ALLOCATION` with the classic API) so a cached object can never be executed; `PagedPool` is correct only when the object is never reachable at or above `DISPATCH_LEVEL`.

IRQL matrix:

| Operation | Non-paged list | Paged list |
|---|---|---|
| Initialization, either API | `PASSIVE_LEVEL` | `PASSIVE_LEVEL` |
| Allocate | `<= DISPATCH_LEVEL` | `PASSIVE_LEVEL` only |
| Free | `<= DISPATCH_LEVEL` | `PASSIVE_LEVEL` only |
| Delete | `PASSIVE_LEVEL` | `PASSIVE_LEVEL` |
| Custom allocate / free callback | Runs at the caller's IRQL and must be safe there | `PASSIVE_LEVEL` |
| Reading `Depth` / miss counters | Any IRQL; the values are a snapshot | Same |

The paged restriction is not a formality: a miss allocates paged pool, which needs a pageable context, and the block itself must not be touched once IRQL reaches `DISPATCH_LEVEL`. Freeing a paged block at `DISPATCH_LEVEL` produces `0xC2 BAD_POOL_CALLER`.

## Step-by-step

1. **Prove the size is fixed.** `sizeof(OBJECT)` must be identical for every user. If the object ends in a variable-length array, move that part to a separate allocation or use a size-class table of lists. Check: one `sizeof` expression appears in exactly one place, the initializer.
2. **Choose the pool kind from the highest caller.** If any path that allocates or frees can run at `DISPATCH_LEVEL`, the list is non-paged, and every field of the object touched at that level must be non-paged too. Check: search for the free call inside DPCs, completion routines, and spinlock-protected regions.
3. **Place the list structure in non-paged memory.** Device extension, driver global, or a block from `ExAllocatePool2` with `POOL_FLAG_NON_PAGED`. Never a paged allocation, even for a paged list, and never a stack local that can expire while blocks are cached. Check: the structure outlives the unload path.
4. **Initialize once with `Depth` 0.** Check the returned status and unwind the device on failure. Passing a non-zero depth is not a tuning knob; the value is reserved. Check: initialization happens before the device is published to clients.
5. **Write a constructor and a destructor, and call them from the wrappers only.** The constructor initializes every field, because a cache hit holds the previous owner's data. The destructor clears every pointer and length so a stale kernel pointer cannot reach the next request. Check: no call site allocates without constructing or frees without destroying.
6. **Route every allocation and free through the two wrappers.** A block freed with `ExFreePoolWithTag` instead of `ExFreeToLookasideListEx` corrupts the pool; a block allocated with `ExAllocatePool2` and freed with `ExFreeToLookasideListEx` corrupts the list. Check: the pool APIs appear only inside the two callbacks.
7. **Protect the live list, not the cache.** The lookaside is internally safe for concurrent allocate and free; the list of objects the driver has handed out is not, and needs its own lock. Check: every insert and remove of a live object holds the same lock.
8. **Drain in the right order at unload.** Stop new producers, wait for in-flight users, free every outstanding object back to the list, delete the list, then delete the device. Check: the outstanding count is zero before `ExDeleteLookasideListEx`.
9. **Measure before believing.** Warm the cache with a few hundred allocations, then confirm `AllocateMisses` stops rising while the request rate stays constant. Check: `!poolused` for the tag is flat across a long run.

## Complete example

A per-request object cache with a constructor, a destructor, a live list and a stress command. The header is shared with the client.

```c
/* KLookPublic.h -- shared between the driver and its client */
#pragma once

#define KLK_DEVICE_TYPE 0x8344

typedef struct _KLK_STATS {
    ULONG LiveCount;         /* objects currently handed out         */
    ULONG TotalAllocated;    /* lifetime allocations                 */
    ULONG Constructed;       /* constructor runs                     */
    ULONG Destroyed;         /* destructor runs                      */
    ULONG CacheMisses;       /* allocations that went to the pool    */
    ULONG CacheReleases;     /* blocks actually returned to the pool */
    ULONG StressIterations;  /* size of the last stress run          */
} KLK_STATS, *PKLK_STATS;

#define IOCTL_KLK_ALLOC \
    CTL_CODE(KLK_DEVICE_TYPE, 0x800, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_KLK_FREE_ALL \
    CTL_CODE(KLK_DEVICE_TYPE, 0x801, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_KLK_STRESS \
    CTL_CODE(KLK_DEVICE_TYPE, 0x802, METHOD_BUFFERED, FILE_ANY_ACCESS)
#define IOCTL_KLK_STATS \
    CTL_CODE(KLK_DEVICE_TYPE, 0x803, METHOD_BUFFERED, FILE_ANY_ACCESS)
```

```c
/* klook.c -- request-object cache with constructor, destructor and stress path */
#include <ntddk.h>
#include "KLookPublic.h"

#define KLK_DEVNAME    L"\\Device\\KLook"
#define KLK_SYMLINK    L"\\??\\KLook"
#define KLK_TAG        'kLKK'          /* bytes appear reversed in a dump */
#define KLK_SIGNATURE  0x7165524B      /* 'KReq' */
#define KLK_MAX_LIVE   4096
#define KLK_MAX_STRESS 1000000

typedef enum _KLK_STATE { KLK_STATE_FREE = 0, KLK_STATE_LIVE = 1 } KLK_STATE;

typedef struct _KLK_REQUEST {
    LIST_ENTRY    Link;            /* valid only while on the live list */
    ULONG         Signature;
    ULONG         Id;
    ULONG         State;
    ULONG         ResultLength;
    PVOID         ClientBuffer;    /* must never survive into the next user */
    ULONG         ClientLength;
    KEVENT        Done;
    LARGE_INTEGER CreatedAt;
} KLK_REQUEST, *PKLK_REQUEST;

typedef struct _KLK_DEVICE_EXT {
    LOOKASIDE_LIST_EX RequestLookaside;   /* non-paged, always */
    LIST_ENTRY        LiveList;
    KSPIN_LOCK        LiveLock;
    volatile LONG     LiveCount;
    volatile LONG     TotalAllocated;
    volatile LONG     Constructed;
    volatile LONG     Destroyed;
    volatile LONG     CacheMisses;
    volatile LONG     CacheReleases;
    volatile LONG     StressIterations;
    BOOLEAN           CacheReady;
} KLK_DEVICE_EXT, *PKLK_DEVICE_EXT;

/*
 * Constructor: runs after every allocation and must initialize every field.
 * A cache hit is not clean memory; it holds whatever the last user left.
 */
static VOID KlkRequestCtor(_Inout_ PKLK_REQUEST Request)
{
    RtlZeroMemory(Request, sizeof(*Request));
    Request->Signature = KLK_SIGNATURE;
    Request->State = KLK_STATE_FREE;
    Request->CreatedAt.QuadPart = (LONGLONG)KeQueryInterruptTime();
    KeInitializeEvent(&Request->Done, NotificationEvent, TRUE);
}

/*
 * Destructor: runs before the block goes back to the cache. Clear anything
 * that could carry a pointer or a length into the next request.
 */
static VOID KlkRequestDtor(_Inout_ PKLK_REQUEST Request)
{
    NT_ASSERT(Request->Signature == KLK_SIGNATURE);

    Request->ClientBuffer = NULL;
    Request->ClientLength = 0;
    Request->ResultLength = 0;
    Request->Id = 0;
    Request->State = KLK_STATE_FREE;
}

/* The list pointer reaches driver state, which the classic callback cannot do. */
static PVOID KlkAllocCallback(_In_ POOL_TYPE PoolType, _In_ SIZE_T NumberOfBytes,
                              _In_ ULONG Tag, _Inout_ PLOOKASIDE_LIST_EX Lookaside)
{
    PKLK_DEVICE_EXT ext =
        CONTAINING_RECORD(Lookaside, KLK_DEVICE_EXT, RequestLookaside);

    UNREFERENCED_PARAMETER(PoolType);   /* NonPagedPoolNx was chosen at init */
    NT_ASSERT(NumberOfBytes == sizeof(KLK_REQUEST));

    InterlockedIncrement(&ext->CacheMisses);
    return ExAllocatePool2(POOL_FLAG_NON_PAGED, NumberOfBytes, Tag);
}

static VOID KlkFreeCallback(_In_ PVOID Buffer, _Inout_ PLOOKASIDE_LIST_EX Lookaside)
{
    PKLK_DEVICE_EXT ext =
        CONTAINING_RECORD(Lookaside, KLK_DEVICE_EXT, RequestLookaside);

    InterlockedIncrement(&ext->CacheReleases);
    ExFreePoolWithTag(Buffer, KLK_TAG);
}

static PKLK_REQUEST KlkAllocateRequest(_In_ PKLK_DEVICE_EXT Ext)
{
    PKLK_REQUEST request =
        (PKLK_REQUEST)ExAllocateFromLookasideListEx(&Ext->RequestLookaside);

    if (request == NULL) {
        return NULL;
    }
    KlkRequestCtor(request);
    InterlockedIncrement(&Ext->TotalAllocated);
    InterlockedIncrement(&Ext->Constructed);
    return request;
}

static VOID KlkFreeRequest(_In_ PKLK_DEVICE_EXT Ext, _Inout_ PKLK_REQUEST Request)
{
    KlkRequestDtor(Request);
    InterlockedIncrement(&Ext->Destroyed);
    ExFreeToLookasideListEx(&Ext->RequestLookaside, Request);
}

static VOID KlkFreeAllLive(_Inout_ PKLK_DEVICE_EXT Ext)
{
    for (;;) {
        LIST_ENTRY *entry;
        KIRQL       irql;

        KeAcquireSpinLock(&Ext->LiveLock, &irql);
        if (IsListEmpty(&Ext->LiveList)) {
            KeReleaseSpinLock(&Ext->LiveLock, irql);
            break;
        }
        entry = RemoveHeadList(&Ext->LiveList);
        InterlockedDecrement(&Ext->LiveCount);
        KeReleaseSpinLock(&Ext->LiveLock, irql);

        KlkFreeRequest(Ext, CONTAINING_RECORD(entry, KLK_REQUEST, Link));
    }
}

static VOID KlkFillStats(_In_ PKLK_DEVICE_EXT Ext, _Out_ PKLK_STATS Stats)
{
    Stats->LiveCount = (ULONG)InterlockedCompareExchange(&Ext->LiveCount, 0, 0);
    Stats->TotalAllocated =
        (ULONG)InterlockedCompareExchange(&Ext->TotalAllocated, 0, 0);
    Stats->Constructed = (ULONG)InterlockedCompareExchange(&Ext->Constructed, 0, 0);
    Stats->Destroyed = (ULONG)InterlockedCompareExchange(&Ext->Destroyed, 0, 0);
    Stats->CacheMisses = (ULONG)InterlockedCompareExchange(&Ext->CacheMisses, 0, 0);
    Stats->CacheReleases =
        (ULONG)InterlockedCompareExchange(&Ext->CacheReleases, 0, 0);
    Stats->StressIterations =
        (ULONG)InterlockedCompareExchange(&Ext->StressIterations, 0, 0);
}

/* Allocate Count requests and put them on the live list under the spinlock. */
static NTSTATUS KlkAllocLive(_Inout_ PKLK_DEVICE_EXT Ext, _In_ ULONG Count)
{
    ULONG i;

    if (Count == 0 || Count > KLK_MAX_LIVE ||
        (ULONG)InterlockedCompareExchange(&Ext->LiveCount, 0, 0) + Count > KLK_MAX_LIVE) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    for (i = 0; i < Count; i++) {
        PKLK_REQUEST request = KlkAllocateRequest(Ext);
        KIRQL        irql;

        if (request == NULL) {
            return STATUS_INSUFFICIENT_RESOURCES;   /* caller frees what exists */
        }
        request->Id = (ULONG)InterlockedIncrement(&Ext->LiveCount);
        request->State = KLK_STATE_LIVE;

        KeAcquireSpinLock(&Ext->LiveLock, &irql);
        InsertTailList(&Ext->LiveList, &request->Link);
        KeReleaseSpinLock(&Ext->LiveLock, irql);
    }
    return STATUS_SUCCESS;
}

/*
 * Allocate and free in a tight loop. After the warm-up every allocation must
 * be a cache hit, and no request may inherit state from the previous one.
 */
static NTSTATUS KlkStress(_Inout_ PKLK_DEVICE_EXT Ext, _In_ ULONG Iterations)
{
    PKLK_REQUEST check;
    ULONG        i;

    for (i = 0; i < Iterations; i++) {
        PKLK_REQUEST request = KlkAllocateRequest(Ext);

        if (request == NULL) {
            return STATUS_INSUFFICIENT_RESOURCES;
        }
        request->Id = i;
        request->State = KLK_STATE_LIVE;
        request->ResultLength = i * sizeof(ULONG);
        request->ClientBuffer = (PVOID)(ULONG_PTR)(i | 1);   /* poison */
        request->ClientLength = i;

        KlkFreeRequest(Ext, request);
    }

    /* The destructor must have cleared the poison before the block was cached. */
    check = KlkAllocateRequest(Ext);
    if (check == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }
    if (check->ClientBuffer != NULL || check->ClientLength != 0) {
        KlkFreeRequest(Ext, check);
        return STATUS_INTERNAL_ERROR;
    }
    KlkFreeRequest(Ext, check);

    InterlockedExchange(&Ext->StressIterations, (LONG)Iterations);
    return STATUS_SUCCESS;
}

static NTSTATUS KlkComplete(_In_ PIRP Irp, _In_ NTSTATUS Status,
                            _In_ ULONG_PTR Information)
{
    Irp->IoStatus.Status = Status;
    Irp->IoStatus.Information = Information;
    IoCompleteRequest(Irp, IO_NO_INCREMENT);
    return Status;
}

static NTSTATUS KlkCreateClose(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    UNREFERENCED_PARAMETER(DeviceObject);
    return KlkComplete(Irp, STATUS_SUCCESS, 0);
}

static NTSTATUS KlkControl(_In_ PDEVICE_OBJECT DeviceObject, _In_ PIRP Irp)
{
    PKLK_DEVICE_EXT    ext = (PKLK_DEVICE_EXT)DeviceObject->DeviceExtension;
    PIO_STACK_LOCATION sp = IoGetCurrentIrpStackLocation(Irp);
    PVOID              buffer = Irp->AssociatedIrp.SystemBuffer;
    ULONG              inLen = sp->Parameters.DeviceIoControl.InputBufferLength;
    ULONG              outLen = sp->Parameters.DeviceIoControl.OutputBufferLength;
    NTSTATUS           status = STATUS_INVALID_DEVICE_REQUEST;
    ULONG_PTR          info = 0;
    ULONG              count;

    /*
     * METHOD_BUFFERED uses one buffer for both directions, so the input is
     * captured before anything writes the output structure over it.
     */
    count = (buffer != NULL && inLen >= sizeof(ULONG)) ? *(PULONG)buffer : 0;

    switch (sp->Parameters.DeviceIoControl.IoControlCode) {
    case IOCTL_KLK_ALLOC:
        if (buffer == NULL || inLen < sizeof(ULONG)) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        status = KlkAllocLive(ext, count);
        break;

    case IOCTL_KLK_FREE_ALL:
        KlkFreeAllLive(ext);
        status = STATUS_SUCCESS;
        break;

    case IOCTL_KLK_STRESS:
        if (buffer == NULL || inLen < sizeof(ULONG) ||
            count == 0 || count > KLK_MAX_STRESS) {
            status = STATUS_INVALID_PARAMETER;
            break;
        }
        status = KlkStress(ext, count);
        break;

    case IOCTL_KLK_STATS:
        if (buffer == NULL || outLen < sizeof(KLK_STATS)) {
            status = STATUS_BUFFER_TOO_SMALL;
            break;
        }
        status = STATUS_SUCCESS;
        break;

    default:
        break;
    }

    if (NT_SUCCESS(status) && buffer != NULL && outLen >= sizeof(KLK_STATS)) {
        KlkFillStats(ext, (PKLK_STATS)buffer);
        info = sizeof(KLK_STATS);
    }

    return KlkComplete(Irp, status, info);
}

/*
 * Drain order: no new clients, no outstanding objects, then the cache, then
 * the device that contains both the cache and the live list.
 */
static VOID KlkUnload(_In_ PDRIVER_OBJECT DriverObject)
{
    PDEVICE_OBJECT  device = DriverObject->DeviceObject;
    PKLK_DEVICE_EXT ext = (PKLK_DEVICE_EXT)device->DeviceExtension;
    UNICODE_STRING  link;

    RtlInitUnicodeString(&link, KLK_SYMLINK);
    IoDeleteSymbolicLink(&link);

    KlkFreeAllLive(ext);

    if (ext->CacheReady) {
        ExDeleteLookasideListEx(&ext->RequestLookaside);
        ext->CacheReady = FALSE;
    }

    NT_ASSERT(InterlockedCompareExchange(&ext->LiveCount, 0, 0) == 0);
    IoDeleteDevice(device);
}

NTSTATUS DriverEntry(_In_ PDRIVER_OBJECT DriverObject,
                     _In_ PUNICODE_STRING RegistryPath)
{
    PDEVICE_OBJECT  device = NULL;
    PKLK_DEVICE_EXT ext;
    UNICODE_STRING  devName;
    UNICODE_STRING  link;
    NTSTATUS        status;

    UNREFERENCED_PARAMETER(RegistryPath);

    RtlInitUnicodeString(&devName, KLK_DEVNAME);
    status = IoCreateDevice(DriverObject, sizeof(KLK_DEVICE_EXT), &devName,
                            FILE_DEVICE_UNKNOWN, FILE_DEVICE_SECURE_OPEN,
                            FALSE, &device);
    if (!NT_SUCCESS(status)) {
        return status;
    }

    ext = (PKLK_DEVICE_EXT)device->DeviceExtension;
    RtlZeroMemory(ext, sizeof(*ext));
    InitializeListHead(&ext->LiveList);
    KeInitializeSpinLock(&ext->LiveLock);

    status = ExInitializeLookasideListEx(&ext->RequestLookaside,
                                         KlkAllocCallback, KlkFreeCallback,
                                         NonPagedPoolNx, 0, sizeof(KLK_REQUEST),
                                         KLK_TAG, 0);
    if (!NT_SUCCESS(status)) {
        IoDeleteDevice(device);
        return status;
    }
    ext->CacheReady = TRUE;

    DriverObject->MajorFunction[IRP_MJ_CREATE] = KlkCreateClose;
    DriverObject->MajorFunction[IRP_MJ_CLOSE] = KlkCreateClose;
    DriverObject->MajorFunction[IRP_MJ_DEVICE_CONTROL] = KlkControl;
    DriverObject->DriverUnload = KlkUnload;

    RtlInitUnicodeString(&link, KLK_SYMLINK);
    status = IoCreateSymbolicLink(&link, &devName);
    if (!NT_SUCCESS(status)) {
        ExDeleteLookasideListEx(&ext->RequestLookaside);
        IoDeleteDevice(device);
        return status;
    }

    device->Flags &= ~DO_DEVICE_INITIALIZING;
    return STATUS_SUCCESS;
}
```

```c
/* klookclient.c -- warm the cache, stress it, and compare the counters */
#include <windows.h>
#include <stdio.h>
#include "KLookPublic.h"

static void PrintStats(const char *Label, const KLK_STATS *s)
{
    printf("%-12s live=%lu total=%lu ctor=%lu dtor=%lu "
           "misses=%lu releases=%lu stress=%lu\n",
           Label, s->LiveCount, s->TotalAllocated, s->Constructed,
           s->Destroyed, s->CacheMisses, s->CacheReleases, s->StressIterations);
}

int main(void)
{
    HANDLE    device;
    KLK_STATS stats;
    DWORD     bytes = 0;
    ULONG     count;

    device = CreateFileW(L"\\\\.\\KLook", GENERIC_READ | GENERIC_WRITE,
                         0, NULL, OPEN_EXISTING, 0, NULL);
    if (device == INVALID_HANDLE_VALUE) {
        printf("open failed: %lu\n", GetLastError());
        return 1;
    }

    count = 64;
    if (!DeviceIoControl(device, IOCTL_KLK_ALLOC, &count, sizeof(count),
                         &stats, sizeof(stats), &bytes, NULL)) {
        printf("alloc failed: %lu\n", GetLastError());
        return 1;
    }
    PrintStats("after-alloc", &stats);

    count = 200000;
    if (DeviceIoControl(device, IOCTL_KLK_STRESS, &count, sizeof(count),
                        &stats, sizeof(stats), &bytes, NULL)) {
        PrintStats("after-stress", &stats);
    } else {
        printf("stress failed: %lu\n", GetLastError());
    }

    if (DeviceIoControl(device, IOCTL_KLK_FREE_ALL, NULL, 0,
                        &stats, sizeof(stats), &bytes, NULL)) {
        PrintStats("after-free", &stats);
    }

    CloseHandle(device);
    return 0;
}
```

The counters tell the whole story. After `alloc 64` the first allocations are misses. After the stress run `CacheMisses` is essentially unchanged while `TotalAllocated` grew by 200000, which proves the loop was satisfied from the cache and not from the pool. `live=0` after `free-all`, together with `ctor == dtor`, proves that every object came back and that the destructor ran once per constructor.

## Variations and advanced use

- **Custom callbacks that do more than forward.** The callbacks are the place to charge quota, prefer a NUMA node through `ExAllocatePool3`, or instrument allocation latency. Keep them allocation-shaped: they must allocate exactly `NumberOfBytes`, must be safe at whatever IRQL the caller used, and must return `NULL` on failure unless a raise flag was requested.

- **Rundown before delete.** If a block can be freed by a path that may still run while unload proceeds — a completion routine, a timer, a worker — the delete must be preceded by a rundown: stop the producers, wait for the in-flight count to reach zero with an event or `KeFlushQueuedDpcs`, then delete. Deleting a list another processor is still popping from is a use-after-free of kernel memory, with `0xC2` or `0x50` as the symptom.

- **Verifier changes the measurement.** With Driver Verifier and special pool enabled, cache hits are reduced because fresh allocations are used to catch overruns; `AllocateMisses` climbs and the timing looks like a plain pool. Tune for performance unverified, for correctness verified.

- **The cache is not a security boundary.** A block returned to the cache still contains whatever the request carried, including kernel addresses copied from a client. The destructor must scrub those fields; otherwise the next request can read the previous request's pointers by simply asking for its own state. This is the reason the constructor zeroes the whole object rather than only its own fields.

### When not to use a lookaside list

| Situation | Why a lookaside is wrong | Use instead |
|---|---|---|
| Sizes vary per request | One list serves one size; a wrong size corrupts adjacent memory | `ExAllocatePool2`, or a size-class table of lists |
| Object is several kilobytes | `MaximumDepth × Size × processors` of permanently held memory | plain pool, or `MmAllocateContiguousMemory` for hardware |
| Object lives for minutes or the life of the driver | The cache never sees it back, so the list only adds a wrapper | plain pool, allocated once |
| Allocation rate is a few per second | No measurable win; the depth tuning never activates | plain pool |
| Every allocation must be zeroed | The cache hands back dirty memory, so the win is partly spent in the constructor | `ExAllocatePool2` with its default zeroing |
| Allocation must be charged to a process quota | The cache bypasses quota accounting | `ExAllocatePoolWithQuotaTag`, or `POOL_FLAG_USE_QUOTA` |

## Failure modes and diagnosis

| Symptom | Likely cause | Fix / first command |
|---|---|---|
| `0xC2 BAD_POOL_CALLER` on a free | A cached block was freed with `ExFreePoolWithTag`, freed twice, or freed after the list was deleted | `!analyze -v`, then `!pool` on the address; route every free through `ExFreeToLookasideListEx` |
| `0xC2` at `DISPATCH_LEVEL` with a paged list | `ExFreeToPagedLookasideList` or its allocate twin was called above `PASSIVE_LEVEL` | `!irql` in the caller; switch the list to `NonPagedPoolNx` and move the object's data with it |
| `0x19 BAD_POOL_HEADER` right after a request completes | The object overran its own allocation and stamped the next header; the damage surfaces at the next free | Enable special pool (`verifier /flags 0x1 /driver klook.sys`), re-run the stress, compare the size passed to the initializer with `sizeof` |
| `0x19` only after a long soak | The list head itself was overwritten: a block was written after being returned to the cache | Poison the object in the destructor and confirm no code keeps a pointer past the free |
| `0x50 PAGE_FAULT_IN_NONPAGED_AREA` inside the cache | A paged block was touched at `DISPATCH_LEVEL`, or a block was used after the list was deleted | `!analyze -v`, then `!pool` and `!irql`; re-check the unload order |
| `0x9F DRIVER_POWER_STATE_FAILURE` | A cached request object still owns a pending IRP or a device reference, so a power transition never completes | The destructor must assert that every field is empty; `!irpfind` and `!analyze -v` |
| `AllocateMisses` keeps climbing under constant load | Frees happen on a different processor than allocations, or objects are held far longer than expected | Expected for cross-processor traffic; check the live count for a leak, then accept the migration |
| `live` never returns to zero after `free-all` | An error path dropped a pointer, or an object left the live list without being returned | Walk `LiveList` with `dl` in the debugger and compare with the counter |
| Pool for the tag grows without bound | A free path is missing, so cached and leaked blocks both accumulate | `!poolused 2 <tag>` across repeated runs; the count must plateau, not climb |
| `0xC4` verifier stop on a lookaside address | Special pool and the cache disagree about the block's boundaries because an overrun exists | Read the verifier parameters and fix the write, not the verifier setting |
| `STATUS_INVALID_PARAMETER` from the initializer | Both raise flags set, a non-zero depth, or a null list pointer | Pass `Depth` 0 and exactly one flag |

## Safety, IRQL, and cleanup checklist

- [ ] The list structure is in non-paged memory and outlives every allocate, free, and delete call.
- [ ] The initializer's `Size` is a single `sizeof` expression, no caller requests another size, and `Depth` is 0.
- [ ] Initialization status is checked and the failure path unwinds the device and the symlink.
- [ ] Every allocation runs the constructor and every free runs the destructor, from the wrappers only.
- [ ] The constructor initializes every field and the destructor clears every pointer, because a cache hit holds the previous owner's data.
- [ ] The pool kind matches the highest IRQL of any caller; a paged list is never touched at `DISPATCH_LEVEL`.
- [ ] Custom callbacks work at the caller's IRQL, allocate exactly the requested size, and free with the tag they were given.
- [ ] The live list has its own lock; the cache's internal concurrency is not an excuse to skip it.
- [ ] Unload order: stop producers, drain in-flight users, free outstanding objects, delete the list, delete the device; nothing is freed afterwards.
- [ ] The pool tag is unique to the driver and shows up in `!poolused` and `!poolfind` output.

## Verification

1. Build, sign, `sc create klook type= kernel binPath= C:\Windows\System32\drivers\klook.sys`, `sc start klook`. DebugView stays silent on a clean load; a failure here is almost always the initializer status.
2. Run the client once with `alloc 64`: `misses` between 1 and 64, `live=64`, `ctor=64`, `dtor=0`. Then `free-all`: `live=0`, `dtor=64`, `releases=0` because the blocks were cached, not returned to the pool.
3. Run `stress 200000`: `total` grows by 200000, `ctor` and `dtor` by the same amount, while `misses` grows by at most a few dozen after warm-up. A `misses` value that tracks the iteration count means the frees are not reaching the cache.
4. In the debugger, `!poolused 2 kLKK` before and after the stress must show the same block count, and `!poolfind kLKK` must show a bounded set of addresses.
5. Break inside the stress loop and run `!irql`; it must print `PASSIVE_LEVEL`. Then call the same allocation from a DPC and confirm it still works, because the list is `NonPagedPoolNx`.
6. `sc stop klook` while the client runs must not bugcheck; the client either completes or receives `ERROR_OPERATION_ABORTED`, and `!poolused` afterwards shows zero blocks for the tag.
7. With `verifier /standard /driver klook.sys` active and a reboot behind it, repeat steps 2 through 6: no `0xC2`, `0x19`, or `0x50`, and `!verifier 3 klook.sys` reports no outstanding allocations at unload.
