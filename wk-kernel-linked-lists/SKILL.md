---
name: wk-kernel-linked-lists
description: Use when storing variable numbers of objects in a Windows kernel driver without per-node allocation: intrusive LIST_ENTRY lists, interlocked SINGLE_LIST_ENTRY/SLIST free lists, and sequenced lists, including safe iteration, removal during traversal, lock pairing, teardown, and corruption debugging.
---

# Intrusive lists and lock-free free lists

Kernel code rarely allocates a container node per element. Instead, the element embeds a
link structure and the list head points at those embedded links. This is intrusive: the
list does not own the element, does not allocate, and cannot fail. It also means the
element's lifetime is entirely your responsibility — a list can hold a pointer to freed
memory just as easily as a valid one, and the resulting bugcheck usually appears far from
the removal site.

Two families cover almost everything:

- `LIST_ENTRY` — doubly linked, ordered/unordered, used with an external lock. Iteration
  and removal are O(1); enumeration is O(n).
- `SINGLE_LIST_ENTRY` / `SLIST_HEADER` — singly linked, push/pop at the head, designed for
  interlocked (lock-free) operation on multiprocessor systems.

Sequenced singly-linked lists (`Rtl*SList*`) extend the SLIST idea with a depth counter
and atomic list splice, and are the primitive behind the kernel's lookaside lists.

## When to use

- A device extension must track pending IRPs, open handles, per-client state, timers, or
  work items.
- You need O(1) insert/remove from an arbitrary position and already hold a lock.
- You need an ordered enumeration of live objects (use `LIST_ENTRY` with tail insertion,
  or a generic table when key lookup is required).
- You need a per-processor free list of fixed-size objects with no lock (SLIST).
- You need to splice a whole list atomically (sequenced list).

Do not use for keyed lookup by value (`wk-generic-tables`), for large or long-lived
allocations (pool), or as a substitute for reference counting. A list does not keep
objects alive.

## Prerequisites and related skills

- `LIST_ENTRY` operations are non-paged and legal at any IRQL ≤ `DISPATCH_LEVEL` as long
  as the elements themselves are non-paged; they do not block and do not raise IRQL.
- The caller must provide synchronization. At `PASSIVE_LEVEL` use a mutex, fast mutex, or
  `ERESOURCE`; at `DISPATCH_LEVEL` or above use a spin lock (`wk-spinlocks`).
- `ExInterlockedPushEntryList` / `ExInterlockedPopEntryList` require non-paged memory.
- Related: `wk-synchronization`, `wk-spinlocks`, `wk-lookaside-lists`,
  `wk-generic-tables`, `wk-irp-queues-cancel`.

## Mental model

`LIST_ENTRY` is two pointers: `Flink` (next) and `Blink` (previous). The list head is
itself a `LIST_ENTRY`; an empty list points both fields at the head. There is no tail
pointer and no count; a tail insertion is O(1) because the head's `Blink` is the last
element.

```
head.Flink -> [A] -> [B] -> [C] -> head
head.Blink <-     <-     <-     <-
```

The link is embedded at some offset inside your structure. `CONTAINING_RECORD(link,
TYPE, Member)` recovers the object pointer by subtracting that offset. The offset must be
computed from the type, not guessed.

```c
typedef struct _REQUEST {
    LIST_ENTRY  Link;       // must be initialized before insertion
    PIRP        Irp;
    ULONG       Tag;
} REQUEST, *PREQUEST;

PREQUEST r = CONTAINING_RECORD(head->Flink, REQUEST, Link);
```

Removal does not clear the removed links' contents beyond making them point to each
other in a debug build. `RemoveEntryList` is O(1) but does not know whether the entry is
currently in a list; removing twice corrupts the neighbors.

## Key APIs and structures

| API / type | Purpose | IRQL | Notes |
|---|---|---|---|
| `LIST_ENTRY` | Doubly linked node | any | `Flink`, `Blink` |
| `InitializeListHead` | Make an empty head | any | head points to itself |
| `InsertHeadList` | Push front | any | O(1) |
| `InsertTailList` | Push back | any | O(1) |
| `RemoveHeadList` | Pop front | any | returns the removed link |
| `RemoveTailList` | Pop back | any | O(1) |
| `RemoveEntryList` | Unlink a known node | any | entry must be linked |
| `IsListEmpty` | Empty test | any | checks `Flink == head` |
| `ExIsListEmpty` | Same, exported | any | useful in binaries |
| `CONTAINING_RECORD` | Link → object | any | compile-time offset |
| `LIST_FOR_EACH_ENTRY` | Typed iteration | any | not safe against removal |
| `LIST_FOR_EACH_ENTRY_SAFE` | Iteration with next cached | any | safe removal of current |
| `SINGLE_LIST_ENTRY` | Singly linked node | any | `Next` only |
| `InterlockedPushEntrySList` | Lock-free push | any | non-paged, 16-byte aligned |
| `InterlockedPopEntrySList` | Lock-free pop | any | returns NULL when empty |
| `ExInterlockedPushEntryList` | Locked push with spin lock | any | legacy, still supported |
| `ExInterlockedPopEntryList` | Locked pop | any | legacy |
| `SLIST_HEADER` | Sequenced list header | any | 16-byte aligned, opaque |
| `RtlInitializeSListHead` | Init sequenced list | any | zeroes header |
| `RtlInterlockedPushListSList` | Splice a whole list | any | atomic multi-push |
| `RtlFirstEntrySList` | Read head without pop | any | racy read, for diagnostics |
| `RtlQueryDepthSList` | Approximate depth | any | for stats only |

## Step-by-step: a pending-request list

1. **Embed the link first** in the object structure. Keep the object non-paged if it can
   be touched at `DISPATCH_LEVEL`.
2. **Initialize the head once**, in `DriverEntry` or device creation:
   `InitializeListHead(&ctx->PendingList);`
3. **Initialize each node's link** before the first insertion. Insertion overwrites
   `Flink`/`Blink`, so initialization is only needed for nodes that may be removed or
   inspected before insertion, but doing it consistently catches bugs.
4. **Choose the lock** by the highest IRQL that touches the list. A list touched by an
   ISR/DPC needs a spin lock; a list touched only by dispatch routines at
   `PASSIVE_LEVEL` can use a mutex or `ERESOURCE`.
5. **Insert under the lock**, increment the count in the same critical section.
6. **Remove under the same lock**, decrement the count, then release the lock before
   completing the IRP (completion can re-enter your driver).
7. **Teardown under the lock:** pop until empty, collect the objects in a local list,
   release the lock, then free/complete them outside the lock. Freeing under a spin lock
   is legal only for pool, but completion and arbitrary callbacks are not.
8. **Verify** with a debug counter and a stress test; run under Driver Verifier with
   special pool to catch use-after-free.

## Complete example

The driver below queues write IRPs, completes them from a timer DPC, and drains the list
on unload. It is deliberately small but exercises the full lifecycle.

```c
// queue.c
#include <ntddk.h>

#define POOL_TAG 'ueuQ'

typedef struct _QUEUE_CTX {
    LIST_ENTRY   Head;
    KSPIN_LOCK   Lock;
    ULONG        Count;
    KTIMER       Timer;
    KDPC         TimerDpc;
    PDEVICE_OBJECT Device;
} QUEUE_CTX, *PQUEUE_CTX;

typedef struct _PENDING_WRITE {
    LIST_ENTRY  Link;
    PIRP        Irp;
} PENDING_WRITE, *PPENDING_WRITE;

static VOID CompleteOne(PQUEUE_CTX ctx)
{
    LIST_ENTRY *entry;
    KIRQL oldIrql;

    KeAcquireSpinLock(&ctx->Lock, &oldIrql);
    if (IsListEmpty(&ctx->Head)) {
        KeReleaseSpinLock(&ctx->Lock, oldIrql);
        return;
    }
    entry = RemoveHeadList(&ctx->Head);
    ctx->Count--;
    KeReleaseSpinLock(&ctx->Lock, oldIrql);

    PPENDING_WRITE item = CONTAINING_RECORD(entry, PENDING_WRITE, Link);
    PIRP irp = item->Irp;
    ExFreePoolWithTag(item, POOL_TAG);

    irp->IoStatus.Status = STATUS_SUCCESS;
    irp->IoStatus.Information = 0;
    IoCompleteRequest(irp, IO_NO_INCREMENT);
}

static VOID TimerDpc(PKDPC Dpc, PVOID Context, PVOID Arg1, PVOID Arg2)
{
    UNREFERENCED_PARAMETER(Dpc);
    UNREFERENCED_PARAMETER(Arg1);
    UNREFERENCED_PARAMETER(Arg2);

    PQUEUE_CTX ctx = (PQUEUE_CTX)Context;
    CompleteOne(ctx);
    KeSetTimer(&ctx->Timer, (LARGE_INTEGER){ .QuadPart = -10000000LL }, &ctx->TimerDpc);
}

NTSTATUS QueueWrite(PQUEUE_CTX ctx, PIRP Irp)
{
    PPENDING_WRITE item = (PPENDING_WRITE)ExAllocatePool2(
        POOL_FLAG_NON_PAGED, sizeof(*item), POOL_TAG);
    if (item == NULL) {
        return STATUS_INSUFFICIENT_RESOURCES;
    }

    item->Irp = Irp;
    InitializeListHead(&item->Link);

    KIRQL oldIrql;
    KeAcquireSpinLock(&ctx->Lock, &oldIrql);
    InsertTailList(&ctx->Head, &item->Link);
    ctx->Count++;
    KeReleaseSpinLock(&ctx->Lock, oldIrql);

    IoMarkIrpPending(Irp);
    return STATUS_PENDING;
}

VOID QueueTeardown(PQUEUE_CTX ctx)
{
    LIST_ENTRY drained;
    InitializeListHead(&drained);

    KeCancelTimer(&ctx->Timer);
    KeFlushQueuedDpcs();

    KIRQL oldIrql;
    KeAcquireSpinLock(&ctx->Lock, &oldIrql);
    while (!IsListEmpty(&ctx->Head)) {
        LIST_ENTRY *entry = RemoveHeadList(&ctx->Head);
        InsertTailList(&drained, entry);
        ctx->Count--;
    }
    KeReleaseSpinLock(&ctx->Lock, oldIrql);

    while (!IsListEmpty(&drained)) {
        LIST_ENTRY *entry = RemoveHeadList(&drained);
        PPENDING_WRITE item = CONTAINING_RECORD(entry, PENDING_WRITE, Link);
        PIRP irp = item->Irp;
        ExFreePoolWithTag(item, POOL_TAG);

        irp->IoStatus.Status = STATUS_CANCELLED;
        irp->IoStatus.Information = 0;
        IoCompleteRequest(irp, IO_NO_INCREMENT);
    }
}
```

Key points in this example:

- The list lock is dropped before `IoCompleteRequest`; completion can call back into the
  driver and would deadlock if the lock were still held.
- `KeFlushQueuedDpcs` guarantees the timer DPC is not running on another CPU before the
  list is drained. `KeCancelTimer` alone only prevents future queuing.
- The drained list is a second `LIST_ENTRY` used as a temporary stack, so objects are
  freed outside the lock.
- `PENDING_WRITE` is non-paged because it is allocated with `POOL_FLAG_NON_PAGED`; the
  list operations are safe at `DISPATCH_LEVEL`.

## Safe iteration and removal

`LIST_FOR_EACH_ENTRY_SAFE` caches the next pointer, so removing the current entry is
safe. Removing the *next* entry, or any other entry, is not safe unless you restart the
iteration.

```c
LIST_ENTRY *entry;
LIST_ENTRY *next;
for (entry = head->Flink, next = entry->Flink;
     entry != head;
     entry = next, next = entry->Flink) {
    PREQUEST r = CONTAINING_RECORD(entry, REQUEST, Link);
    if (ShouldRemove(r)) {
        RemoveEntryList(entry);
        // r is still valid until freed; next was cached
    }
}
```

Never traverse a list without holding the lock that protects membership, unless the list
is immutable for the duration (for example, a list built at init and only read after).
If you must drop the lock while processing an element, take a reference on the element
first, or revalidate that it is still linked.

## Singly linked and sequenced lists

`SINGLE_LIST_ENTRY` has only `Next`. Push and pop at the head are O(1) and can be done
with `InterlockedPushEntrySList` / `InterlockedPopEntrySList` without a lock. The classic
pattern is a per-device free list, or a queue consumed by a single DPC.

```c
typedef struct _FREE_NODE {
    SINGLE_LIST_ENTRY Link;
    ULONG             Value;
} FREE_NODE, *PFREE_NODE;

SLIST_HEADER FreeList;   // must be 16-byte aligned, non-paged

// init once
RtlInitializeSListHead(&FreeList);

// push
PFREE_NODE node = (PFREE_NODE)ExAllocatePool2(
    POOL_FLAG_NON_PAGED, sizeof(*node), POOL_TAG);
if (node != NULL) {
    node->Value = 42;
    InterlockedPushEntrySList(&FreeList, &node->Link);
}

// pop
PSINGLE_LIST_ENTRY e = InterlockedPopEntrySList(&FreeList);
if (e != NULL) {
    PFREE_NODE popped = CONTAINING_RECORD(e, FREE_NODE, Link);
    // use popped; it is exclusively owned now
}
```

The `SLIST_HEADER` must be 16-byte aligned. The compiler aligns a standalone
`SLIST_HEADER` correctly, but a header inside a packed structure or at an odd offset in
a device extension may not be. Use `DECLSPEC_ALIGN(16)` or place it first in the
extension and verify with `static_assert`.

`RtlInterlockedPushListSList` splices an entire chain in one atomic operation, which is
how you move a batch of completed items to a consumer without per-item interlocked
traffic:

```c
// Detach everything currently in the list, then process the chain locally.
PSINGLE_LIST_ENTRY chain = InterlockedFlushSList(&FreeList);
while (chain != NULL) {
    PSINGLE_LIST_ENTRY next = chain->Next;
    PFREE_NODE n = CONTAINING_RECORD(chain, FREE_NODE, Link);
    Process(n);
    chain = next;
}
```

`InterlockedFlushSList` empties the list and returns the chain in one step; it is the
correct way to drain a lock-free list without racing producers.

## Variations and trade-offs

| Structure | Order | Insert/remove | Enumeration | Lock | Best for |
|---|---|---|---|---|---|
| `LIST_ENTRY` | any | O(1) | O(n) | external | general device state |
| `LIST_ENTRY` sorted | sorted | O(n) | O(n) | external | small ordered sets |
| `SINGLE_LIST_ENTRY` | LIFO | O(1) | O(n) | external or interlocked | free lists |
| `SLIST_HEADER` | LIFO | O(1) atomic | O(n) | none | per-CPU caches, DPC queues |
| Sequenced list | LIFO | O(1) atomic | O(n) | none | batch transfer, lookaside |
| Generic table | key order | O(log n) | O(n) | external | lookup by key (`wk-generic-tables`) |

A doubly linked list is a poor choice for membership tests (`Is X in the list?`) because
each test is O(n). If lookups dominate, use a hash table or a generic table.

## Failure modes and diagnosis

| Symptom | Likely cause | First action |
|---|---|---|
| 0x1A `MEMORY_MANAGEMENT` | list corruption from stale `Flink`/`Blink` | `!analyze -v`; inspect the list head and neighbors with `dt` |
| 0xC2 `BAD_POOL_CALLER` | freeing a node still linked, or double free | special pool; `!pool` on the node |
| 0x50 `PAGE_FAULT_IN_NONPAGED_AREA` | paged node touched at `DISPATCH_LEVEL` | check pool type and IRQL |
| 0x139 `KERNEL_SECURITY_CHECK_FAILURE` | `RemoveEntryList` on an unlinked entry | enable verifier; inspect `Blink` of the removed entry |
| 0xD1 `DRIVER_IRQL_NOT_LESS_OR_EQUAL` | list touched at raised IRQL without a spin lock, or paged data | `!irql`, verify IRQL at access |
| Hang | lock held across `IoCompleteRequest` or a blocking call | `!stacks 2`, `!locks`, inspect lock order |
| UAF after removal | another thread still traversing | reference-count objects; never traverse unlocked |
| 0x9F on unload | list still populated, IRPs pending | drain before unload; assert `IsListEmpty` |

Debugging notes:

- `dt nt!_LIST_ENTRY <addr>` shows `Flink`/`Blink`; a valid entry's neighbors' back
  pointers must point back to it.
- `!pool <addr>` identifies the pool tag and allocation size of a node.
- With Driver Verifier's special pool, a use-after-free usually becomes an immediate
  bugcheck at the faulting instruction instead of silent corruption.
- `!analyze -v` on 0xC2 prints the pool tag and the caller's stack; match the tag to your
  driver's tags first.

## Safety, IRQL, and cleanup checklist

- [ ] Every list head is initialized exactly once before any insert.
- [ ] Every node's `LIST_ENTRY` is initialized before insertion (especially stack nodes).
- [ ] Membership changes happen under one lock, consistently.
- [ ] The lock is never held across `IoCompleteRequest`, `KeWaitFor*`, or a callback that
      can re-enter the driver.
- [ ] Teardown cancels timers/DPCs and flushes them before draining the list.
- [ ] Objects are removed from the list before being freed.
- [ ] `CONTAINING_RECORD` uses the correct type and member.
- [ ] `SLIST_HEADER` is 16-byte aligned and non-paged.
- [ ] A debug counter is checked at unload: it must be zero after draining.
- [ ] `LIST_FOR_EACH_ENTRY_SAFE` is used whenever the current entry may be removed.

## Verification

1. Build with `/W4` and run a stress client that opens many handles and issues
   overlapping writes.
2. Enable Driver Verifier with special pool and forced IRQL checking:
   `verifier /standard /driver your.sys`, reboot, rerun the stress client.
3. In WinDbg, break after the stress run and inspect the list:
   `dt nt!_LIST_ENTRY <head>`, then follow `Flink` until it returns to the head; verify
   `Blink` symmetry.
4. Unload while I/O is in flight. The driver must cancel and complete every queued IRP,
   `!irpfind` must show none belonging to the device, and the unload must not hang.
5. Check `!poolused 2 <tag>` before and after 100 load/unload cycles; the tag count must
   return to the baseline.
