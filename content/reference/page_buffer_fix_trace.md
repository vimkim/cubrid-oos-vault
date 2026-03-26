# pgbuf_fix() End-to-End Trace — CUBRID Buffer Pool Deep Dive

> **Pinned to commit**: `8a3ff8901b608a284e5838d88e57eefe6dc13f20` (develop branch)
> **Source file**: `src/storage/page_buffer.c` (16,931 lines)
> **Header file**: `src/storage/page_buffer.h` (499 lines)
> **Generated**: 2026-03-27 via deep-interview → ralplan → autopilot pipeline

---

## Table of Contents

1. [Overview](#1-overview)
2. [Caller Entry Points](#2-caller-entry-points)
3. [The Main pgbuf_fix Body](#3-the-main-pgbuf_fix-body)
4. [Lockfree Fast Path](#4-lockfree-fast-path)
5. [Hash Lookup](#5-hash-lookup)
6. [Cache Miss: BCB Allocation](#6-cache-miss-bcb-allocation)
7. [Disk Read](#7-disk-read)
8. [Latch Acquisition](#8-latch-acquisition)
9. [Latch Promotion](#9-latch-promotion)
10. [Return Path](#10-return-path)
11. [Marking Pages Dirty](#11-marking-pages-dirty)
12. [pgbuf_unfix](#12-pgbuf_unfix)
13. [Ordered Fix](#13-ordered-fix)
14. [Concurrency Model and Flow Diagrams](#14-concurrency-model-and-flow-diagrams)

---

## 1. Overview

The buffer pool is CUBRID's page cache — it mediates all access between the database engine and disk storage. Every B-tree traversal, heap scan, and log operation must go through `pgbuf_fix()` to acquire a page, and `pgbuf_unfix()` to release it. Understanding this lifecycle is understanding the foundation of the entire storage engine.

### 1.1 What pgbuf_fix Does

`pgbuf_fix()` is the single entry point for acquiring a database page. Given a VPID (volume ID + page ID), it:
1. Looks up the page in an in-memory hash table
2. If found (cache hit): acquires a latch and returns a pointer to the page
3. If not found (cache miss): allocates a BCB, reads the page from disk, acquires a latch, and returns

The caller receives a `PAGE_PTR` — a pointer directly into the buffer pool's memory. The page stays pinned (cannot be evicted) until `pgbuf_unfix()` is called.

### 1.2 The Macro Dispatch

`pgbuf_fix` is not a function — it is a macro that dispatches to different implementations depending on the build:

```c
// Debug build (page_buffer.h:275-276):
#define pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) \
        pgbuf_fix_debug(thread_p, vpid, fetch_mode, requestmode, condition, ARG_FILE_LINE_FUNC)

// Release build (page_buffer.h:319-320):
#define pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) \
        pgbuf_fix_release(thread_p, vpid, fetch_mode, requestmode, condition)
```

The debug version passes `__FILE__`, `__LINE__`, `__func__` so every fix can be traced back to its caller. Both variants share the same function body — they are a single `#if !defined(NDEBUG)` / `#else` block at `page_buffer.c:2032-2040`. The same pattern applies to `pgbuf_unfix`, `pgbuf_ordered_fix`, `pgbuf_promote_read_latch`, and `pgbuf_set_dirty`.

### 1.3 Key Data Structures

#### BCB (Buffer Control Block) — `page_buffer.c:503-535`

Every buffered page has a BCB that tracks its metadata:

```c
struct pgbuf_bcb {
    pthread_mutex_t  mutex;           // Per-BCB mutex (SERVER_MODE only)
    int              owner_mutex;     // Debug: which thread holds the mutex
    VPID             vpid;            // Which page this BCB holds (volid + pageid)
    PGBUF_ATOMIC_LATCH atomic_latch;  // Lockfree latch state (see §1.4)
    volatile int     flags;           // BCB_DIRTY | FLUSHING | VICTIM_DIRECT | etc.
    THREAD_ENTRY    *next_wait_thrd;  // Head of waiter queue (threads blocked on latch)
    PGBUF_BCB       *hash_next;       // Hash chain linkage (for VPID lookup)
    PGBUF_BCB       *prev_BCB;        // LRU doubly-linked list (prev)
    PGBUF_BCB       *next_BCB;        // LRU doubly-linked list (next) / invalid list
    int              tick_lru_list;    // Age at insertion into LRU (for zone promotion decisions)
    int              tick_lru3;        // Position in zone 3 (for victim hint)
    volatile int     count_fix_and_avoid_dealloc;  // Upper 16 bits: fix count, lower 16: avoid-dealloc
    int              hit_age;          // Last fix time (for quota/activity tracking)
    LOG_LSA          oldest_unflush_lsa; // Oldest dirty LSA (for checkpoint ordering)
    PGBUF_IOPAGE_BUFFER *iopage_buffer;  // Pointer to the actual page data
};
```

The BCB and its page data (`PGBUF_IOPAGE_BUFFER`) are separately allocated but cross-referenced. The page data structure is:

```c
struct pgbuf_iopage_buffer {   // page_buffer.c:538-547
    PGBUF_BCB  *bcb;           // Back-pointer to BCB
    FILEIO_PAGE iopage;        // The actual page: header + DB_PAGESIZE bytes of content
};
```

**Why separate allocation?** BCBs are allocated as a contiguous array for fast indexing, while iopage buffers need specific alignment for direct I/O. The cross-references enable O(1) navigation in both directions.

#### Pointer Cast Macros — `page_buffer.c:144-166`

These macros convert between the three pointer types:

```c
// PAGE_PTR → PGBUF_BCB*  (page_buffer.c:145-150)
// Walks backward from the page content pointer to find the containing iopage_buffer,
// then follows the bcb back-pointer.
CAST_PGPTR_TO_BFPTR(bufptr, pgptr):
    bufptr = ((PGBUF_IOPAGE_BUFFER*)((char*)pgptr - offsetof(iopage.page)))->bcb

// PGBUF_BCB* → PAGE_PTR  (page_buffer.c:162-166)
// Follows the iopage_buffer pointer and offsets to the page content.
CAST_BFPTR_TO_PGPTR(pgptr, bufptr):
    pgptr = (char*)bufptr->iopage_buffer + offsetof(iopage.page)
```

When `pgbuf_fix` returns `pgptr`, it points directly into `iopage_buffer->iopage.page` — the raw page content in buffer memory. The caller never sees the BCB directly.

### 1.4 The Atomic Latch

The latch is the concurrency primitive that controls who can read/write a page. CUBRID uses a 64-bit atomic for lockfree latch operations:

```c
typedef std::atomic<uint64_t> PGBUF_ATOMIC_LATCH;  // page_buffer.c:362

union pgbuf_atomic_latch_impl {     // page_buffer.c:491-500
    uint64_t raw;                   // CAS target
    struct {
        PGBUF_LATCH_MODE latch_mode;  // uint16_t: NO_LATCH=0, READ=1, WRITE=2, FLUSH=3, INVALID=4
        uint16_t         waiter_exists; // 1 if any thread is blocked waiting for this BCB
        int32_t          fcnt;          // Fix count (number of concurrent latch holders)
    } impl;
};
```

**Why pack into 64 bits?** A single CAS (compare-and-swap) can atomically check the latch mode, waiter flag, AND fix count — then update all three. This enables the lockfree fast path (§4) that avoids all mutexes for the common case of concurrent readers on a cached page.

### 1.5 Fetch Modes

Every `pgbuf_fix` call specifies a fetch mode (`page_buffer.h:172-187`) that tells the buffer pool what to expect:

| Mode | Value | Meaning | Disk Read? |
|------|-------|---------|-----------|
| `OLD_PAGE` | 0 | Normal fetch — page must exist on disk or in buffer | Yes, if not cached |
| `NEW_PAGE` | 1 | Newly allocated page — no disk read needed | No |
| `OLD_PAGE_IF_IN_BUFFER` | 2 | Return page only if already cached; NULL otherwise | No |
| `OLD_PAGE_PREVENT_DEALLOC` | 3 | Fix and atomically prevent deallocation | Yes, if not cached |
| `OLD_PAGE_DEALLOCATED` | 4 | Explicitly fetch a deallocated page | Yes, if not cached |
| `OLD_PAGE_MAYBE_DEALLOCATED` | 5 | Page might be deallocated; return NULL if so | Yes, if not cached |
| `RECOVERY_PAGE` | 6 | Recovery context — any state is possible | Yes, if not cached |

### 1.6 Latch Modes

| Mode | Value | Meaning |
|------|-------|---------|
| `PGBUF_NO_LATCH` | 0 | No latch held |
| `PGBUF_LATCH_READ` | 1 | Shared read latch — multiple concurrent readers allowed |
| `PGBUF_LATCH_WRITE` | 2 | Exclusive write latch — only one holder |
| `PGBUF_LATCH_FLUSH` | 3 | Flush latch — used internally by flush daemon, never by callers |
| `PGBUF_LATCH_INVALID` | 4 | BCB is being victimized — no latch possible |

### 1.7 BCB Flags — `page_buffer.c:220-248`

| Flag | Value | Meaning |
|------|-------|---------|
| `PGBUF_BCB_DIRTY_FLAG` | `0x80000000` | Page modified, not yet flushed |
| `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` | `0x40000000` | Flush in progress |
| `PGBUF_BCB_VICTIM_DIRECT_FLAG` | `0x20000000` | Assigned as direct victim to a waiting thread |
| `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` | `0x10000000` | Direct victim cancelled (someone fixed it) |
| `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG` | `0x08000000` | Move to bottom of LRU on unfix (set on dealloc) |
| `PGBUF_BCB_TO_VACUUM_FLAG` | `0x04000000` | Page needs vacuuming |
| `PGBUF_BCB_ASYNC_FLUSH_REQ` | `0x02000000` | Async flush requested |

A BCB cannot be victimized if any of these flags are set: DIRTY, FLUSHING, VICTIM_DIRECT, or INVALIDATE_DIRECT_VICTIM (the `PGBUF_BCB_INVALID_VICTIM_CANDIDATE_MASK` at line 255-259).

---

## 2. Caller Entry Points

Before diving into `pgbuf_fix` internals, it helps to see how real callers invoke it. Every subsystem that touches data pages uses this API.

### 2.1 B-tree Root Page Fix — `btree.c:1869`

```c
root_page = pgbuf_fix (thread_p, root_vpid_p, OLD_PAGE, latch_mode, PGBUF_UNCONDITIONAL_LATCH);
```

This is the starting point of every B-tree operation — fixing the root page. `latch_mode` is `PGBUF_LATCH_READ` for searches and `PGBUF_LATCH_WRITE` for structure modifications (splits/merges). The `UNCONDITIONAL` condition means "wait as long as needed."

### 2.2 B-tree Latch Coupling (Crab Latch)

During B-tree traversal, CUBRID uses the classic latch coupling pattern: fix the child page before releasing the parent.

```c
// Fix root unconditionally (will wait if needed)
root_pgptr = pgbuf_fix (thread_p, &current_vpid, OLD_PAGE, request_mode, PGBUF_UNCONDITIONAL_LATCH);

// Fix child/sibling conditionally (fail immediately if can't latch)
current_pgptr = pgbuf_fix (thread_p, &current_vpid, OLD_PAGE, request_mode, PGBUF_CONDITIONAL_LATCH);
if (current_pgptr == NULL)
  {
    // Another thread holds this page — retry from root
    retry_count++;
    goto retry_repair;
  }
```

**Why conditional for children?** Unconditional latching of two pages simultaneously risks deadlock (thread A holds page X and waits for Y; thread B holds Y and waits for X). Conditional latching avoids this: if you can't get the child immediately, release everything and retry.

### 2.3 Heap Page Scan with Ordered Fix — `heap_file.c:3885`

```c
PGBUF_WATCHER pg_watcher;
PGBUF_INIT_WATCHER (&pg_watcher, PGBUF_ORDERED_HEAP_NORMAL, hfid);
ret = pgbuf_ordered_fix (thread_p, &vpid, OLD_PAGE_PREVENT_DEALLOC, PGBUF_LATCH_READ, &pg_watcher);
```

Heap operations use `pgbuf_ordered_fix` instead of `pgbuf_fix` to prevent deadlocks when holding multiple heap/overflow pages. See §13 for the full ordered fix protocol.

### 2.4 The Retry Wrapper — `page_buffer.c:1990-2022`

```c
PAGE_PTR pgbuf_fix_with_retry (THREAD_ENTRY *thread_p, const VPID *vpid,
                               PAGE_FETCH_MODE fetch_mode, PGBUF_LATCH_MODE request_mode, int retry)
```

A thin wrapper that calls `pgbuf_fix(..., PGBUF_UNCONDITIONAL_LATCH)` in a loop up to `retry` times, retrying only on `ER_LK_PAGE_TIMEOUT` or `ER_PAGE_LATCH_TIMEDOUT`. Used by some heap operations that expect rare, transient latch failures.

---

## 3. The Main pgbuf_fix Body

The complete function is at `page_buffer.c:2034-2457`. Here is the annotated control flow in execution order.

### 3.1 Parameter Validation — Lines 2064-2094

```c
// Only READ and WRITE latches are valid for callers
if (request_mode != PGBUF_LATCH_READ && request_mode != PGBUF_LATCH_WRITE)
    return NULL;
// Only UNCONDITIONAL and CONDITIONAL are valid
if (condition != PGBUF_UNCONDITIONAL_LATCH && condition != PGBUF_CONDITIONAL_LATCH)
    return NULL;
```

Then a global fix request counter is bumped atomically (line 2076):
```c
pgbuf_Pool.monitor.fix_req_cnt.fetch_add (1, std::memory_order_relaxed);
```

Followed by optional page validity check (lines 2078-2086) — verifying the page is allocated on disk. Skipped for `RECOVERY_PAGE` since anything is possible during recovery.

### 3.2 Wait Mode Adjustment — Lines 2096-2106

```c
if (condition == PGBUF_UNCONDITIONAL_LATCH) {
    wait_msecs = pgbuf_find_current_wait_msecs (thread_p);
    if (wait_msecs == LK_ZERO_WAIT || wait_msecs == LK_FORCE_ZERO_WAIT)
        condition = PGBUF_CONDITIONAL_LATCH;  // Downgrade to conditional
}
```

**Why?** If the transaction's lock wait timeout is zero (no-wait mode), an unconditional page latch would violate that contract. The function silently downgrades to conditional, which fails immediately if the latch can't be acquired.

### 3.3 The try_again Label — Lines 2116-2127

```c
try_again:
    // Interrupt check — was the transaction interrupted?
    if (logtb_get_check_interrupt (thread_p) == true)
        if (logtb_is_interrupted (thread_p, true, &pgbuf_Pool.check_for_interrupts) == true)
            return NULL;  // ER_INTERRUPTED already set
```

This label is the target for retry loops. If `pgbuf_claim_bcb_for_fix` fails because another thread was loading the same page, control jumps back here (line 2194: `goto try_again`).

### 3.4 Branch: Lockfree Fast Path → Hash Lookup → Cache Miss

After the preamble, the code branches into three paths:

1. **Lockfree fast path** (lines 2132-2151): For READ + OLD_PAGE + UNCONDITIONAL on a cached page → zero-mutex path
2. **Hash lookup** (lines 2153-2211): Standard path with hash chain search and BCB mutex
3. **Cache miss** (lines 2186-2211): Page not in buffer → allocate BCB, read from disk

The rest of this document traces each path in detail.

---

## 4. Lockfree Fast Path

This is the HOT PATH for CUBRID's buffer pool. Most read operations on cached pages (the common case in a warm database) hit this path and never touch a single mutex or lock.

### 4.1 Entry Condition — Lines 2132-2134

```c
if (request_mode == PGBUF_LATCH_READ
    && (fetch_mode == OLD_PAGE || fetch_mode == OLD_PAGE_PREVENT_DEALLOC
        || fetch_mode == OLD_PAGE_MAYBE_DEALLOCATED)
    && condition == PGBUF_UNCONDITIONAL_LATCH)
{
    pgptr = pgbuf_lockfree_fix_ro (thread_p, vpid, fetch_mode);
    if (pgptr != NULL) goto fast_path;  // Success — skip all locking
}
```

Three conditions must all be true:
- **READ latch** (not WRITE)
- **OLD_PAGE variant** (not NEW_PAGE or IF_IN_BUFFER)
- **UNCONDITIONAL** (willing to wait, but on this path, waiting never happens)

If any condition fails, or if `pgbuf_lockfree_fix_ro` returns NULL, we fall through to the standard hash lookup path.

### 4.2 pgbuf_lockfree_fix_ro — `page_buffer.c:7449-7511`

This function performs a completely lockfree page fix:

**Step 1**: Find the BCB via lockfree hash scan.

```c
bufptr = pgbuf_search_hash_chain_no_bcb_lock (hash_anchor, vpid);
if (bufptr == NULL) return NULL;  // Not in buffer — fall through to standard path
```

`pgbuf_search_hash_chain_no_bcb_lock` (lines 7514-7528) walks the hash chain without acquiring ANY lock — not even the hash bucket mutex. It simply follows `hash_next` pointers looking for a VPID match. This is safe because:
- BCBs are never freed (only recycled), so pointers remain valid
- VPID comparison is on immutable fields during a fix
- If the BCB was just victimized, the CAS in the next step will fail

**Step 2**: CAS loop on atomic_latch.

```c
do {
    old_latch.raw = bufptr->atomic_latch.load (std::memory_order_acquire);
    if (old_latch.impl.latch_mode != PGBUF_LATCH_READ  // Must already be in READ mode
        || old_latch.impl.waiter_exists                  // No waiters (write waiter = unfair to skip)
        || old_latch.impl.fcnt <= 0                      // Must have at least one holder
        || !VPID_EQ (&bufptr->vpid, vpid))               // VPID must still match (not victimized)
        return NULL;  // Can't use fast path — fall through

    new_latch.raw = old_latch.raw;
    new_latch.impl.fcnt++;  // Increment fix count
} while (!bufptr->atomic_latch.compare_exchange_weak (old_latch.raw, new_latch.raw));
```

**Why require fcnt > 0?** If fcnt is 0, the page is being released by another thread. The state is in flux — the page might be about to be victimized. We need the standard path with proper mutex protection for that case.

**Why fail on waiter_exists?** If a writer is waiting, letting more readers in would starve the writer. Fairness requires falling through to the standard path where the waiter queue is properly managed.

**Step 3**: Find or allocate a holder, return the page pointer.

After the CAS succeeds, the thread's fix count has been atomically incremented. The function finds (or allocates) a `PGBUF_HOLDER` for this thread-BCB pair, increments `holder->fix_count`, and returns the page pointer.

**Performance impact**: This path executes in ~50-100ns (CAS + pointer arithmetic). The standard path with mutex acquisition takes 200-500ns or more under contention. For a hot buffer pool where most pages are cached and read-mostly, the lockfree path handles the vast majority of fixes.

---

## 5. Hash Lookup

When the lockfree fast path fails (write latch, page not cached, conditional latch, or waiters present), we fall through to the standard hash lookup path.

### 5.1 Hash Table Structure — `page_buffer.c:567-574`

```c
struct pgbuf_buffer_hash {
    pthread_mutex_t hash_mutex;    // Per-bucket mutex
    PGBUF_BCB      *hash_next;     // Head of BCB chain
    PGBUF_BUFFER_LOCK *lock_next;  // Head of buffer-lock chain (for serializing disk reads)
};
```

The global hash table is `pgbuf_Pool.buf_hash_table` — a flat array of `PGBUF_HASH_SIZE` = `1 << 20` = **1,048,576** buckets (line 293). With 1M buckets, most pages hash to their own bucket, making collisions rare.

### 5.2 Hash Function — `page_buffer.c:1438-1468`

```c
STATIC_INLINE unsigned int pgbuf_hash_func_mirror (const VPID *vpid)
{
    // Bit-reverse the 8 LSBs of volid into the upper bits of a 20-bit value
    volid_lsb = vpid->volid;
    for (i = 8; i > 0; i--) {
        reversed_volid_lsb |= (volid_lsb & 1) << (19 - i);
        volid_lsb >>= 1;
    }
    hash_val = vpid->pageid ^ reversed_volid_lsb;
    hash_val = hash_val & ((1 << 20) - 1);  // Mask to 20 bits
    return hash_val;
}
```

**Why bit-reversal?** Sequential page IDs (0, 1, 2, 3...) would all cluster in neighboring buckets if we just used pageid directly. The bit-reversal of the volume ID into the HIGH bits creates spacing between volumes, while the XOR with pageid distributes sequential pages across the hash space. This prevents hot-bucket contention during sequential scans.

### 5.3 pgbuf_search_hash_chain — `page_buffer.c:7324-7446`

This is a two-phase optimistic search:

**Phase 1 — Lockfree scan** (label `one_phase`):

Walk `hash_anchor->hash_next` without holding `hash_mutex`. On VPID match, try `PGBUF_BCB_TRYLOCK(bufptr)`:
- **Trylock succeeds**: Verify VPID still matches (BCB may have been replaced between finding it and locking it). If match, return `bufptr` (locked).
- **Trylock fails (EBUSY)**: Another thread holds the BCB mutex. Fall through to Phase 2.

**Phase 2 — Mutex-protected scan** (label `two_phase`):

1. Acquire `hash_anchor->hash_mutex`
2. Walk the chain again. On VPID match: trylock BCB.
3. If BCB busy: **release hash_mutex**, then unconditionally lock BCB (blocking). This order prevents holding two locks simultaneously.
4. After acquiring BCB mutex: re-verify VPID match (it may have changed). Loop on mismatch.

**Return contract**:
- If `bufptr != NULL`: caller holds `bufptr->mutex`, does NOT hold `hash_mutex`
- If `bufptr == NULL`: caller holds `hash_mutex` (needed to insert a new BCB into the chain)

### 5.4 Cache Hit: Direct Victim Interlock — Lines 2157-2161

```c
if (bufptr != NULL && pgbuf_bcb_is_direct_victim (bufptr)) {
    pgbuf_bcb_update_flags (thread_p, bufptr,
                            PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG,
                            PGBUF_BCB_VICTIM_DIRECT_FLAG);
}
```

**Why?** There is a race between victim assignment and page fix. Thread A may be victimizing this BCB (has set `VICTIM_DIRECT_FLAG`) while Thread B finds it in the hash chain and wants to fix it. Thread B wins — it sets the `INVALIDATE` flag to cancel the victimization. Thread A will check this flag and pick a different victim.

### 5.5 Cache Hit: OLD_PAGE_IF_IN_BUFFER — Lines 2180-2185

```c
else if (fetch_mode == OLD_PAGE_IF_IN_BUFFER) {
    pthread_mutex_unlock (&hash_anchor->hash_mutex);
    return NULL;  // Page not in buffer, and caller doesn't want a disk read
}
```

This is a "peek" mode — the caller only wants the page if it's already cached. No error is set.

---

## 6. Cache Miss: BCB Allocation

When `pgbuf_search_hash_chain` returns NULL, the page is not in the buffer pool. The caller holds `hash_mutex` and needs to allocate a BCB, load the page from disk, and insert it into the hash chain.

### 6.1 pgbuf_claim_bcb_for_fix — `page_buffer.c:8130-8361`

This is the orchestrator for the cache miss path.

#### Step 1: Serialize Concurrent Fetchers

```c
if (pgbuf_lock_page (thread_p, hash_anchor, vpid) != PGBUF_LOCK_HOLDER) {
    // Another thread is already loading this exact page.
    // We sleep until they finish, then retry from try_again.
    *try_again = true;
    return NULL;
}
```

**pgbuf_lock_page** (`page_buffer.c:7715-7812`):
- Uses the `hash_anchor->lock_next` chain (a linked list of `PGBUF_BUFFER_LOCK` entries, one per thread)
- If VPID already in the lock chain: another thread is fetching this page. Append current thread to `cur_buffer_lock->next_wait_thrd` and sleep via `pgbuf_sleep()`.
- If VPID not found: add our entry to the chain, return `PGBUF_LOCK_HOLDER`.

**Why serialize?** Without this, two threads requesting the same uncached page would both allocate BCBs and both read from disk. One would overwrite the other, wasting I/O and creating an orphan BCB. The page lock ensures only ONE thread does the disk read; others wait and then retry (finding the page in the hash chain on the second try).

#### Step 2: Allocate a BCB

```c
bufptr = pgbuf_allocate_bcb (thread_p, vpid);
```

**pgbuf_allocate_bcb** (`page_buffer.c:7913-8116`) tries three sources in priority order:

1. **Invalid (free) list**: `pgbuf_get_bcb_from_invalid_list()` — pops from `pgbuf_Pool.buf_invalid_list.invalid_top` under `invalid_mutex`. This is the cheapest source — the BCB is already free.

2. **LRU victim**: `pgbuf_get_victim()` — searches LRU zone 3 for a non-dirty, non-fixed, non-victimized BCB. See §6.2.

3. **Direct victim wait** (SERVER_MODE only): If no victim is available anywhere:
   - Thread enqueues itself into `pgbuf_Pool.direct_victims.waiter_threads_high_priority` or `waiter_threads_low_priority` (lockfree circular queues)
   - Wakes the page flush daemon to create victims by flushing dirty pages
   - Blocks via `thread_suspend_timeout_wakeup_and_unlock_entry()` with a 300-second timeout
   - When woken, calls `pgbuf_get_direct_victim()` to retrieve the BCB assigned by the flush daemon

After obtaining a BCB, `pgbuf_victimize_bcb()` is called to prepare it:
- Remove the BCB from its old hash chain via `pgbuf_delete_from_hash_chain()`
- Set `atomic_latch` to `PGBUF_LATCH_INVALID` atomically (prevents any concurrent fix)
- Set the new VPID

### 6.2 Victim Selection — `page_buffer.c:8802-8979`

**pgbuf_get_victim** searches for a candidate BCB to reuse. The search order optimizes for locality:

1. **Own private LRU list** — only if over quota (avoids starving own transaction's cache)
2. **Other private lists** — via lockfree circular queues (`big_private_lrus_with_victims`, then `private_lrus_with_victims`)
3. **Shared LRU lists** — via `pgbuf_lfcq_get_victim_from_shared_lru()`
4. **Fallback**: retry own private LRU even if under quota

**pgbuf_get_victim_from_lru_list** (`page_buffer.c:9050-9230`):

```
Acquire lru_list->mutex
Start from lru_list->victim_hint (or bottom if NULL)
Scan backward through zone-3 BCBs:
    Skip if: dirty | flushing | victim_direct flags set
    Skip if: fix count > 0 or waiters exist
    On candidate: PGBUF_BCB_TRYLOCK
        Success and still victimizable:
            pgbuf_remove_from_lru_list()
            Release LRU mutex
            Add old VPID to Aout list (for 2Q promotion tracking)
            Return BCB (still holding BCB mutex)
        Fail or changed: continue scan
    Max search depth: 1000 BCBs
Release lru_list->mutex, return NULL
```

**The 3-zone LRU** (`page_buffer.c:181-209`):

| Zone | Victimizable? | Boost on unfix? | Purpose |
|------|:---:|:---:|---------|
| Zone 1 (hot) | No | No | Hottest pages; minimal unfix overhead |
| Zone 2 (buffer) | No | Yes, if old enough | Gives pages falling from zone 1 a second chance |
| Zone 3 (victim) | Yes | Yes, always | Cold pages; aggressive victimization |

New pages enter at the top (zone 1 boundary). As newer pages push them down, they pass through zone 2 into zone 3. If a zone 3 page is fixed again, it gets boosted back to the top (second chance). This is similar to MySQL InnoDB's young/old sublist but with an extra buffer zone.

---

## 7. Disk Read

After `pgbuf_allocate_bcb` returns a fresh BCB, `pgbuf_claim_bcb_for_fix` loads the page from disk (lines 8221-8323).

### 7.1 The Read Path

```c
if (fetch_mode != NEW_PAGE) {
    perfmon_inc_stat (thread_p, PSTAT_PB_NUM_IOREADS);

    // Try Double Write Buffer first
    if (dwb_read_page (thread_p, vpid, &bufptr->iopage_buffer->iopage, &success) != NO_ERROR)
        goto error;
    if (success == false) {
        // DWB miss — read directly from disk
        fileio_read (thread_p,
                     fileio_get_volume_descriptor (vpid->volid),
                     &bufptr->iopage_buffer->iopage,
                     vpid->pageid,
                     IO_PAGESIZE);
    }

    // TDE decryption if encrypted
    tde_algo = pgbuf_get_tde_algorithm (pgptr);
    if (tde_algo != TDE_ALGORITHM_NONE)
        tde_decrypt_data_page (&bufptr->iopage_buffer->iopage, tde_algo, ...);
}
```

**Double Write Buffer (DWB)**: CUBRID uses a DWB for crash recovery — pages are written to the DWB first, then to their final location. On read, we check the DWB in case the final write was a partial page (torn write). If the DWB has a complete copy, we use that instead.

**fileio_read**: The actual disk I/O. `fileio_get_volume_descriptor()` maps a volume ID to an open file descriptor. `fileio_read` reads `IO_PAGESIZE` bytes at the page offset.

### 7.2 NEW_PAGE Case

For `NEW_PAGE`, no disk read is needed — the page was just allocated:
- LSA is initialized to null (permanent volume) or temp LSA (temporary volume)
- In debug builds, page content is scrambled to catch use-before-initialize bugs

---

## 8. Latch Acquisition

After the page is in the buffer (either from cache hit or disk read), `pgbuf_fix` must acquire a latch before returning the page to the caller. This is the concurrency control point.

### 8.1 pgbuf_latch_bcb_upon_fix — `page_buffer.c:6070-6391`

At entry, the caller holds `bufptr->mutex`. The function uses CAS loops on `atomic_latch` to determine the latch decision.

### 8.2 The Latch Decision Matrix

| Current State | Request | Condition | Action |
|---|---|---|---|
| Page idle (`buf_lock_acquired` or `latch_mode == NO_LATCH`) | READ or WRITE | Any | CAS: set `latch_mode = request_mode`, `fcnt = 1`. Allocate holder. Return. |
| READ, no waiters | READ | Any | CAS: `fcnt++`. Increment holder's fix_count. Return. |
| READ, waiters exist | READ (already a holder) | Any | CAS: `fcnt++`. Return (existing holder can piggyback). |
| READ, waiters exist | READ (new holder) | Any | **Block** — fairness: don't starve the writer waiting in queue |
| WRITE (same holder) | WRITE | Any | CAS: `fcnt++`. Increment holder's fix_count. Return. |
| READ (sole reader, same holder) | WRITE (promote) | Any | CAS: set `latch_mode = WRITE`. Return. |
| READ (other readers) | WRITE (promote) | CONDITIONAL | Return `ER_FAILED` immediately |
| READ (other readers) | WRITE (promote) | UNCONDITIONAL | `promote_needed = true`, subtract own fcnt, block |
| Any conflict | Any | CONDITIONAL | Return `ER_FAILED` (`ER_LK_PAGE_TIMEOUT`) |
| Any conflict | Any | UNCONDITIONAL | Block via `pgbuf_block_bcb()` |

### 8.3 Blocking Path — `page_buffer.c:6800-6912`

When a thread must wait for a latch:

```c
pgbuf_block_bcb (thread_p, bufptr, request_mode, request_fcnt, is_promoter);
```

1. **Append to waiter queue**: `cur_thrd_entry` is added to `bufptr->next_wait_thrd` (singly-linked list). Normal waiters are FIFO (append to tail). Promoters are prepended (LIFO) — they get priority because they already hold a partial latch.

2. **Set waiter_exists flag**: CAS on `atomic_latch` to set `waiter_exists = 1`. This prevents new readers from using the lockfree fast path, ensuring fairness.

3. **Sleep**: Calls `pgbuf_timed_sleep()` (`page_buffer.c:7011-7100`):
   - Releases BCB mutex (cannot hold mutex while sleeping)
   - Computes timeout: `pgbuf_latch_timeout` (default: 300 seconds)
   - Calls `thread_suspend_timeout_wakeup_and_unlock_entry()` — blocks on a per-thread condition variable
   - On timeout: `pgbuf_timed_sleep_error_handling()` removes self from waiter queue, attempts to grant latch to compatible waiters, returns error

### 8.4 Waiter Wakeup — `page_buffer.c:7183-7322`

When `pgbuf_unlatch_bcb_upon_unfix` detects `fcnt == 0` and `waiter_exists`, it calls:

```c
pgbuf_wakeup_reader_writer (thread_p, bufptr)
```

This walks `bufptr->next_wait_thrd` and grants latches:
- **NO_LATCH waiter** (timed out): remove from list, skip
- **FLUSH waiter**: leave in list (flush daemon, handled separately)
- **READ waiters**: CAS `latch_mode = READ`, accumulate `fcnt` for all consecutive readers, wake them all
- **WRITE waiter**: CAS `latch_mode = WRITE`, `fcnt = request_fcnt`, wake first writer only, stop

**Fairness**: Once a WRITE waiter is found, no more READs behind it are granted. This prevents writer starvation.

---

## 9. Latch Promotion

Sometimes a caller fixes a page with READ (for cheap inspection), then discovers it needs to modify the page. Rather than unfixing and refixing with WRITE (which might fail if the page is evicted), the caller can promote the latch.

### 9.1 pgbuf_promote_read_latch — `page_buffer.c:2621-2813`

```c
// Macro dispatch (page_buffer.h:288-289):
#define pgbuf_promote_read_latch(thread_p, pgptr_p, condition) \
    pgbuf_promote_read_latch_debug(thread_p, pgptr_p, condition, ARG_FILE_LINE_FUNC)
```

Two promotion conditions (`page_buffer.h:206-209`):
- `PGBUF_PROMOTE_ONLY_READER`: Promote only if I am the SOLE reader (fcnt == my fix_count)
- `PGBUF_PROMOTE_SHARED_READER`: Promote even with other readers (they will be unlatched)

**Promotion algorithm**:

1. **Check sole reader**: If `atomic_latch.fcnt == holder->fix_count`, I'm the only reader:
   - CAS: set `latch_mode = WRITE`. Done.

2. **Multiple readers, PROMOTE_ONLY_READER**: Fail — return error. Caller must unfix and refix.

3. **Multiple readers, PROMOTE_SHARED_READER**:
   - Subtract my fix count from `fcnt` via CAS
   - Set `waiter_exists = 1`
   - Block via `pgbuf_block_bcb()` as a promoter (prepended to waiter queue for priority)
   - When all other readers unfix, I'm woken and granted the WRITE latch

**Usage example in B-tree**: A search fixes the leaf page with READ. If a key must be deleted, it promotes to WRITE. If promotion fails (other readers), it unfixes and retries from the root with WRITE all the way down.

---

## 10. Return Path

After latch acquisition succeeds, `pgbuf_fix` finalizes and returns the page to the caller.

### 10.1 Hash Chain Insertion (New Pages Only) — Lines 2306-2316

If `buf_lock_acquired` is true (this was a cache miss), the BCB must be connected into the hash chain:

```c
if (buf_lock_acquired) {
    pgbuf_insert_into_hash_chain (thread_p, hash_anchor, bufptr);
    pgbuf_unlock_page (thread_p, hash_anchor, vpid, false);
    // pgbuf_unlock_page wakes any threads that were sleeping in pgbuf_lock_page
    // waiting for this same VPID to finish loading
}
```

### 10.2 Pointer Conversion — Line 2318

```c
CAST_BFPTR_TO_PGPTR (pgptr, bufptr);
```

The BCB pointer is converted to a `PAGE_PTR` that points directly into `iopage_buffer->iopage.page`. This is what the caller receives and operates on.

### 10.3 Deallocated Page Check — Lines 2344-2388

After returning from latch (or from the lockfree fast path at label `fast_path`), the code inspects the page type:

```c
if (bufptr->iopage_buffer->iopage.prv.ptype == PAGE_UNKNOWN) {
    // Page is deallocated
    switch (fetch_mode) {
    case NEW_PAGE:
    case OLD_PAGE_DEALLOCATED:
    case OLD_PAGE_IF_IN_BUFFER:
    case RECOVERY_PAGE:
        break;  // Expected — return the page

    case OLD_PAGE:
    case OLD_PAGE_PREVENT_DEALLOC:
        assert (false);  // Bug — caller didn't expect a deallocated page
        pgbuf_unfix (thread_p, pgptr);
        return NULL;

    case OLD_PAGE_MAYBE_DEALLOCATED:
        er_set (ER_WARNING_SEVERITY, ...);  // Not an error, just a warning
        pgbuf_unfix (thread_p, pgptr);
        return NULL;
    }
}
```

**Why check after latch?** The page could have been deallocated by another transaction between the hash lookup and the latch acquisition. By checking with the latch held, we prevent a TOCTOU race.

### 10.4 What the Caller Receives

After `pgbuf_fix` returns successfully:
- `pgptr` points to `DB_PAGESIZE` bytes of page content in buffer memory
- The caller holds a READ or WRITE latch (tracked in `atomic_latch.fcnt`)
- The page is pinned — it cannot be evicted until `pgbuf_unfix` is called
- **No mutex is held** by the caller — the BCB mutex was released inside `pgbuf_latch_bcb_upon_fix`
- The latch is tracked in the thread's `PGBUF_HOLDER` linked list

---

## 11. Marking Pages Dirty

After fixing a page with WRITE latch and modifying its contents, the caller must mark it dirty before unfixing. This is the bridge between "page is latched" and "modifications will persist."

### 11.1 pgbuf_set_dirty — `page_buffer.h:369-374`

```c
// Macro dispatch:
#define pgbuf_set_dirty(...)  pgbuf_set_dirty_debug(__VA_ARGS__, ARG_FILE_LINE_FUNC)
void pgbuf_set_dirty (THREAD_ENTRY *thread_p, PAGE_PTR pgptr, bool free_page);
```

The `free_page` parameter controls whether to also unfix the page (convenience for the common pattern of modify → dirty → unfix).

**What it does**:
1. Sets `PGBUF_BCB_DIRTY_FLAG` on `bufptr->flags`
2. Updates `bufptr->oldest_unflush_lsa` — the oldest LSA of unflushed modifications to this page. This is critical for checkpoint ordering: the checkpoint process uses this LSA to determine which pages must be flushed.
3. If `free_page == true`, calls `pgbuf_unfix`

**Connection to WAL**: Dirty pages must not be written to disk until their corresponding log records have been flushed (the WAL protocol: write-ahead logging). The `oldest_unflush_lsa` enables this check — before flushing a dirty page, the flush daemon verifies that the log has been flushed up to at least this LSA.

**Convenience macro** (`page_buffer.h:375`):
```c
#define pgbuf_set_dirty_and_free(thread_p, pgptr) \
    pgbuf_set_dirty (thread_p, pgptr, FREE); pgptr = NULL
```

---

## 12. pgbuf_unfix

The counterpart to `pgbuf_fix` — releases a page latch and potentially allows the page to be evicted.

### 12.1 Entry — `page_buffer.c:2847-3046`

```c
void pgbuf_unfix (THREAD_ENTRY *thread_p, PAGE_PTR pgptr) {
    CAST_PGPTR_TO_BFPTR (bufptr, pgptr);  // Reverse pointer arithmetic to find BCB

    // Step 1: Release thread's holder
    holder_status = pgbuf_unlatch_thrd_holder (thread_p, bufptr, &holder_perf_stat);

    // Step 2: Try lockfree fast path
    if (pgbuf_lockfree_unfix_ro (thread_p, bufptr))
        return;  // Done — no mutex needed

    // Step 3: Full path with BCB mutex
    PGBUF_BCB_LOCK (bufptr);
    pgbuf_unlatch_bcb_upon_unfix (thread_p, bufptr, holder_status);
    // BCB mutex released inside above function
}
```

### 12.2 Holder Release — `page_buffer.c:5907-5956`

`pgbuf_unlatch_thrd_holder` finds the `PGBUF_HOLDER` for this thread-BCB pair:

```c
holder->fix_count--;
if (holder->fix_count == 0)
    pgbuf_remove_thrd_holder (thread_p, holder);
    // Moves holder from thrd_hold_list to thrd_free_list (recycled, not freed)
```

A holder represents one thread's claim on a page. If the thread called `pgbuf_fix` twice on the same page (nested fix), the holder's `fix_count` is 2. Only when it reaches 0 is the holder removed.

### 12.3 Lockfree Unfix Fast Path — `page_buffer.c:7531-7553`

Mirror of the lockfree fix path:

```c
do {
    old_latch = bufptr->atomic_latch.load();
    if (old_latch.impl.latch_mode != PGBUF_LATCH_READ
        || old_latch.impl.waiter_exists
        || old_latch.impl.fcnt <= 1)  // Must have >1 holders (I'm not the last)
        return false;  // Can't use fast path

    new_latch = old_latch;
    new_latch.impl.fcnt--;
} while (!CAS(old_latch, new_latch));
return true;
```

**Why fail when fcnt <= 1?** If I'm the last holder, releasing the latch requires LRU zone adjustments and potentially waking waiters — operations that need the BCB mutex.

### 12.4 Full Unlatch — `page_buffer.c:6414-6636`

`pgbuf_unlatch_bcb_upon_unfix` handles the general case with BCB mutex held:

```c
// CAS loop: decrement fcnt
// If fcnt becomes 0: set latch_mode = NO_LATCH
```

**When fcnt reaches 0 and no waiters**:

The BCB's LRU zone is adjusted based on access patterns:

| Current Zone | Action | Why |
|---|---|---|
| `PGBUF_VOID_ZONE` | `pgbuf_unlatch_void_zone_bcb()` — check Aout list for 2Q promotion, place into appropriate LRU position | New page entering LRU for the first time |
| `PGBUF_LRU_1_ZONE` | No movement. Register hit. May move private→shared if over quota. | Already hot — minimize unfix overhead |
| `PGBUF_LRU_2_ZONE` | If BCB is "old enough" (`PGBUF_IS_BCB_OLD_ENOUGH`): boost to top via `pgbuf_lru_boost_bcb()` | Gives falling pages a second chance |
| `PGBUF_LRU_3_ZONE` | Always boost to top via `pgbuf_lru_boost_bcb()` | Cold page that's still being used — rescue it |
| `MOVE_TO_LRU_BOTTOM_FLAG` | `pgbuf_move_bcb_to_bottom_lru()` | Page was deallocated — make it the first victim candidate |

**Dirty page tracking during unfix**:
- If the holder dirtied the page, `oldest_unflush_lsa` is updated to track the earliest unflushed modification
- If `PGBUF_BCB_ASYNC_FLUSH_REQ` flag is set, an async flush is triggered

**When waiters exist**: `pgbuf_wakeup_reader_writer()` is called (see §8.4) to grant the latch to the next thread in the waiter queue.

---

## 13. Ordered Fix

`pgbuf_ordered_fix` is a deadlock-prevention wrapper around `pgbuf_fix`. It is MANDATORY for heap and overflow pages (`PAGE_HEAP`, `PAGE_OVERFLOW`) — the `PGBUF_IS_ORDERED_PAGETYPE` macro at `page_buffer.h:166-167` enforces this.

### 13.1 The Deadlock Problem

Consider two threads:
- Thread A holds page X (heap data), wants page Y (overflow)
- Thread B holds page Y (overflow), wants page X (heap data)

Both will wait forever. `pgbuf_ordered_fix` prevents this by enforcing a global ordering on page latches.

### 13.2 PGBUF_WATCHER and PGBUF_ORDERED_RANK

```c
struct pgbuf_watcher {          // page_buffer.h:234-249
    PAGE_PTR  pgptr;            // The fixed page
    PGBUF_WATCHER *next, *prev; // Doubly-linked watcher list on holder
    PGBUF_ORDERED_GROUP group_id; // VPID of heap header (group anchor)
    unsigned latch_mode:7;
    unsigned page_was_unfixed:1;  // Set if refix occurred (caller must re-read)
    unsigned initial_rank:4;      // Rank at init time
    unsigned curr_rank:4;         // Rank after fix
};

typedef enum {                  // page_buffer.h:222-229
    PGBUF_ORDERED_HEAP_HDR = 0,     // Heap header — highest priority (fixed first)
    PGBUF_ORDERED_HEAP_NORMAL = 1,  // Normal heap data page
    PGBUF_ORDERED_HEAP_OVERFLOW = 2, // Overflow page — lowest priority
    PGBUF_ORDERED_RANK_UNDEFINED = 3
} PGBUF_ORDERED_RANK;
```

**Ordering rule**: Within a group (same heap file), pages are ordered: HEAP_HDR < HEAP_NORMAL < HEAP_OVERFLOW. Across groups, ordering is by VPID. Lower rank/VPID must be fixed FIRST.

### 13.3 The Algorithm — `page_buffer.c:11977-12835`

**Phase 1: Try conditional fix** (lines 12051-12067):

```c
// If thread holds no other pages, use UNCONDITIONAL (no deadlock risk)
// Otherwise, use CONDITIONAL (fail immediately if can't latch)
latch_condition = (holder == NULL || only_req_page_held) ?
                  PGBUF_UNCONDITIONAL_LATCH : PGBUF_CONDITIONAL_LATCH;
ret_pgptr = pgbuf_fix (thread_p, req_vpid, fetch_mode, request_mode, latch_condition);
```

If conditional fix **succeeds**: attach watcher, return. Most calls succeed here — the conditional retry is rare.

**Phase 2: Reorder** (if conditional fix failed):

1. Walk the thread's holder list to find all pages held with watchers
2. For each held page, compare `(group_id, rank, vpid)` against the requested page
3. Pages that rank HIGHER than the requested page (i.e., should be fixed AFTER) must be temporarily released
4. Call `pgbuf_bcb_register_avoid_deallocation()` on each page to prevent it from being deallocated while temporarily unfixed
5. Unfix those pages

**Phase 3: Fix requested page unconditionally**:

```c
pgptr = pgbuf_fix (thread_p, req_vpid, fetch_mode, request_mode, PGBUF_UNCONDITIONAL_LATCH);
```

Now safe because all higher-ranked pages have been released.

**Phase 4: Refix released pages**:

Re-acquire the previously unfixed pages using `pgbuf_fix()`. Set `watcher->page_was_unfixed = true` on each refixed watcher so callers know the page content may have changed (another thread may have modified it while we didn't hold the latch).

---

## 14. Concurrency Model and Flow Diagrams

### 14.1 Lock Hierarchy

Locks must be acquired in this order to prevent deadlock:

```
hash_mutex → BCB mutex → LRU mutex
```

- **hash_mutex** (per bucket): Acquired during `pgbuf_search_hash_chain` Phase 2, `pgbuf_insert_into_hash_chain`, `pgbuf_delete_from_hash_chain`
- **BCB mutex** (per BCB): Acquired during latch operations, unfix, victim selection
- **LRU mutex** (per LRU list): Acquired during `pgbuf_get_victim_from_lru_list`, `pgbuf_lru_boost_bcb`, zone adjustments in unfix

**Key invariant**: A thread never holds a lower-precedence lock while trying to acquire a higher-precedence one. Specifically:
- `pgbuf_search_hash_chain` releases `hash_mutex` BEFORE blocking on `BCB mutex`
- Victim selection acquires `LRU mutex`, then tries `BCB mutex` via trylock (never blocking)

### 14.2 Other Synchronization Primitives

| Lock | Scope | When Held |
|------|-------|-----------|
| `buf_invalid_list.invalid_mutex` | Global | Pop/push free BCB list |
| `atomic_latch` CAS | Per BCB | Lockfree latch state transitions |
| `buf_lock_table` entry | Per VPID miss | Serialise concurrent disk reads for same page |
| `direct_victims` queues | Global | Thread enqueue/dequeue for victim assignment |

### 14.3 pgbuf_fix Complete Flow Diagram

```
pgbuf_fix(thread_p, vpid, fetch_mode, request_mode, condition)
  │
  ├── [macro] → pgbuf_fix_debug() / pgbuf_fix_release()
  │
  ├── Validate request_mode (READ|WRITE) and condition (UNCONDITIONAL|CONDITIONAL)
  ├── pgbuf_Pool.monitor.fix_req_cnt++ (atomic, relaxed)
  ├── Optional page validity check (skip for RECOVERY_PAGE)
  ├── Adjust condition to CONDITIONAL if transaction is in zero-wait mode
  │
  │try_again:
  ├── Interrupt check → return NULL if transaction interrupted
  │
  │   ┌─── LOCKFREE FAST PATH (§4) ─────────────────────────────────────────┐
  ├──►│ if READ + OLD_PAGE* + UNCONDITIONAL:                                │
  │   │   pgbuf_lockfree_fix_ro()                                           │
  │   │     pgbuf_search_hash_chain_no_bcb_lock() [ZERO locks]              │
  │   │     CAS atomic_latch: fcnt++ if READ && !waiters && fcnt>0 && VPID= │
  │   │     find/allocate PGBUF_HOLDER                                      │
  │   │     → return pgptr ────────────────────────────────────────► DONE   │
  │   │   if NULL: fall through to hash lookup                              │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── HASH LOOKUP (§5) ────────────────────────────────────────────────┐
  ├──►│ hash_anchor = buf_hash_table[pgbuf_hash_func_mirror(vpid)]          │
  │   │ bufptr = pgbuf_search_hash_chain()                                  │
  │   │   Phase 1: lockfree scan + BCB trylock                              │
  │   │   Phase 2: hash_mutex + BCB lock (fallback)                         │
  │   │                                                                      │
  │   │ if bufptr != NULL (CACHE HIT):                                      │
  │   │   Cancel direct_victim flag if set                                  │
  │   │   → continue to LATCH ──────────────────────────────────────► [A]   │
  │   │                                                                      │
  │   │ if fetch_mode == OLD_PAGE_IF_IN_BUFFER:                             │
  │   │   unlock hash_mutex → return NULL                                   │
  │   │                                                                      │
  │   │ else (CACHE MISS):                                                  │
  │   │   → continue to BCB ALLOCATION ─────────────────────────────► [B]   │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── BCB ALLOCATION (§6) ────────────────────────────── [B] ──────────┐
  │   │ pgbuf_claim_bcb_for_fix()                                           │
  │   │   pgbuf_lock_page() — serialize concurrent fetchers for same VPID   │
  │   │     if another thread is loading: sleep, retry=true → goto try_again│
  │   │   pgbuf_allocate_bcb()                                              │
  │   │     1. pgbuf_get_bcb_from_invalid_list() [free BCB pool]            │
  │   │     2. pgbuf_get_victim() [LRU zone-3 scan]                        │
  │   │     3. direct_victim wait [sleep, wake flush daemon]                │
  │   │   pgbuf_victimize_bcb()                                             │
  │   │     pgbuf_delete_from_hash_chain() → set latch INVALID              │
  │   │                                                                      │
  │   │   DISK READ (§7):                                                   │
  │   │     if fetch_mode != NEW_PAGE:                                      │
  │   │       dwb_read_page() → fileio_read() → tde_decrypt()              │
  │   │     else:                                                           │
  │   │       initialize LSA, scramble in debug                             │
  │   │                                                                      │
  │   │   buf_lock_acquired = true                                          │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── POST-LOOKUP (both hit and miss paths) ── [A] ───────────────────┐
  │   │ pgbuf_bcb_register_fix() [hot-page counter, up to 64]              │
  │   │ pgbuf_set_bcb_page_vpid() [ensure VPID set, recovery case]        │
  │   │ pgbuf_check_bcb_page_vpid() [validate page]                       │
  │   │ if OLD_PAGE_PREVENT_DEALLOC: register avoid_deallocation            │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── LATCH ACQUISITION (§8) ─────────────────────────────────────────┐
  │   │ pgbuf_latch_bcb_upon_fix()                                          │
  │   │   CAS loop on atomic_latch {latch_mode, waiter_exists, fcnt}        │
  │   │                                                                      │
  │   │   [idle page] → set mode=request, fcnt=1 → allocate holder         │
  │   │   [READ+READ, no waiters] → fcnt++                                 │
  │   │   [same holder, WRITE] → fcnt++                                    │
  │   │   [promote READ→WRITE, sole reader] → CAS latch_mode=WRITE        │
  │   │   [conflict, CONDITIONAL] → return ER_FAILED                       │
  │   │   [conflict, UNCONDITIONAL] → pgbuf_block_bcb()                    │
  │   │       append to next_wait_thrd queue                                │
  │   │       pgbuf_timed_sleep() [300s timeout]                           │
  │   │                                                                      │
  │   │   Allocate PGBUF_HOLDER, set fix_count                             │
  │   │   BCB mutex released                                                │
  │   └─────────────────────────────────────────────────────────────────────┘
  │
  │   ┌─── RETURN PATH (§10) ──────────────────────────────────────────────┐
  │   │ if buf_lock_acquired:                                               │
  │   │   pgbuf_insert_into_hash_chain()                                    │
  │   │   pgbuf_unlock_page() → wake threads waiting for same VPID         │
  │   │                                                                      │
  │   │ CAST_BFPTR_TO_PGPTR(pgptr, bufptr)                                 │
  │   │                                                                      │
  │   │ fast_path: (lockfree path joins here)                               │
  │   │                                                                      │
  │   │ Deallocated page check:                                             │
  │   │   PAGE_UNKNOWN → switch on fetch_mode                               │
  │   │     NEW_PAGE/DEALLOCATED/IF_IN_BUFFER/RECOVERY → ok                │
  │   │     OLD_PAGE/PREVENT_DEALLOC → assert(false), unfix, return NULL   │
  │   │     MAYBE_DEALLOCATED → warning, unfix, return NULL                │
  │   │                                                                      │
  │   │ Performance stats recording                                         │
  │   │ return pgptr ─────────────────────────────────────────────► DONE    │
  │   └─────────────────────────────────────────────────────────────────────┘
```

### 14.4 pgbuf_unfix Flow Diagram

```
pgbuf_unfix(thread_p, pgptr)
  │
  ├── CAST_PGPTR_TO_BFPTR(bufptr, pgptr)
  │
  ├── pgbuf_unlatch_thrd_holder()
  │     holder->fix_count--
  │     if fix_count == 0: move holder to free list
  │
  ├── LOCKFREE FAST PATH:
  │   pgbuf_lockfree_unfix_ro()
  │     CAS: fcnt-- if READ && !waiters && fcnt > 1
  │     → return (success) ────────────────────────────────► DONE
  │     → fall through (need full path)
  │
  ├── PGBUF_BCB_LOCK(bufptr)
  │
  ├── pgbuf_unlatch_bcb_upon_unfix()
  │     CAS: fcnt--, latch_mode=NO_LATCH if fcnt==0
  │     │
  │     ├── [fcnt > 0] → release BCB mutex → DONE
  │     │
  │     ├── [fcnt == 0, no waiters]:
  │     │     LRU zone adjustment:
  │     │       VOID_ZONE   → 2Q/Aout placement logic
  │     │       LRU_1_ZONE  → stay, register hit, maybe private→shared
  │     │       LRU_2_ZONE  → boost to top if old enough
  │     │       LRU_3_ZONE  → always boost to top
  │     │       BOTTOM_FLAG → move to LRU bottom
  │     │     Dirty tracking: update oldest_unflush_lsa if dirtied
  │     │     Async flush trigger if ASYNC_FLUSH_REQ set
  │     │     → release BCB mutex → DONE
  │     │
  │     └── [fcnt == 0, waiters]:
  │           pgbuf_wakeup_reader_writer()
  │             Walk next_wait_thrd queue:
  │               READ waiters → CAS fcnt+=N, wake all consecutive readers
  │               WRITE waiter → CAS latch_mode=WRITE, wake first only
  │           → release BCB mutex → DONE
```

### 14.5 Thread State During pgbuf_fix

| Stage | hash_mutex | BCB mutex | LRU mutex | atomic_latch |
|-------|:---:|:---:|:---:|:---:|
| Lockfree fast path | - | - | - | CAS: fcnt++ |
| Hash Phase 1 | - | trylock | - | - |
| Hash Phase 2 | held | trylock→block | - | - |
| Post hash hit | - | held | - | - |
| BCB allocation | - | - | held (victim scan) | - |
| Latch grant | - | held→released | - | CAS: mode+fcnt |
| Latch block | - | released | - | CAS: waiter_exists |
| Return to caller | - | - | - | fcnt > 0 (held via atomic) |
| Unfix fast | - | - | - | CAS: fcnt-- |
| Unfix full | - | held→released | maybe (zone adjust) | CAS: fcnt-- |

---

*Document produced via deep-interview (8 rounds, 12.7% ambiguity) → ralplan (Architect + Critic consensus, 13 improvements applied) → autopilot execution.*
