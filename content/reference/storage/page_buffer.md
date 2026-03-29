# Page Buffer Manager — Comprehensive Analysis Report

> File: `src/storage/page_buffer.c` + `src/storage/page_buffer.h`
> Generated from: CUBRID engine, develop branch
> Analysis date: 2026-03-27

---

## 1. File Overview

### File Identity

| Property | Value |
|----------|-------|
| Source file | `src/storage/page_buffer.c` |
| Header file | `src/storage/page_buffer.h` |
| Source lines | 16,931 |
| Header lines | 499 |
| Language | C (compiled as C++17 via `c_to_cpp.sh`) |
| License | Apache 2.0 |

### Purpose and Role

`page_buffer.c` implements CUBRID's **buffer pool manager** — the central memory management layer for database pages. Every access to any on-disk page (heap, B-tree, log, temporary) must go through this module. Its responsibilities are:

1. **Page pinning (fix/unfix):** Provides `pgbuf_fix()` and `pgbuf_unfix()` to pin a page into memory and release it. A fixed page is guaranteed to remain in memory; an unfixed page may be evicted.
2. **LRU replacement policy:** Maintains a three-zone LRU replacement policy (hot, buffer, victim zones) across multiple parallel LRU lists (shared and private per-transaction).
3. **Dirty tracking:** Tracks which pages have been modified (`oldest_unflush_lsa`) so the WAL (Write-Ahead Logging) protocol can enforce durability.
4. **Flush coordination:** Flushes dirty pages to disk in cooperation with the log manager, the double-write buffer, and background daemon threads.
5. **Hash-based lookup:** Uses a large open hash table (1M buckets) to map `VPID` → `BCB` (Buffer Control Block).
6. **Latch management:** Enforces reader-writer latches on each BCB to coordinate concurrent page access between threads.
7. **TDE (Transparent Data Encryption) integration:** Stores and checks encryption algorithm flags per page.
8. **Ordered fix protocol:** Implements a deadlock-free page-fixing order for heap files via `pgbuf_ordered_fix()`.
9. **Recovery support:** Provides special fix modes for crash recovery, deallocation undo, and new-page logging.
10. **Page quota system:** Per-transaction private LRU lists with quota tracking to ensure fair buffer allocation under high concurrency.

### Build Modes

| Guard | Binary | Notes |
|-------|--------|-------|
| `SERVER_MODE` | `cub_server` | Full multi-threaded mode with mutexes, daemon threads, direct victim queues |
| `SA_MODE` | `cubridsa` | Standalone: server code runs in same process as client; mutexes stubbed out |
| `CS_MODE` | `cubridcs` | Client library: does **not** include `page_buffer.c` directly |

In `SA_MODE`, all `pthread_mutex_*` calls are replaced by no-ops (`#define pthread_mutex_lock(a) 0`), making the buffer pool single-threaded. Daemon threads (`pgbuf_Page_flush_daemon`, etc.) are `SERVER_MODE`-only.

---

## 2. Includes & Dependencies

### System Headers (in `page_buffer.c`)

```c
#include <stdlib.h>    // malloc, free, qsort
#include <stddef.h>    // offsetof
#include <string.h>    // memset, memcpy, memcmp
#include <assert.h>    // assert
#include <atomic>      // std::atomic (C++17)
```

### Internal Dependencies — `src/storage/`

| File | Usage |
|------|-------|
| `page_buffer.h` | Own header — enums, structs, public API declarations |
| `storage_common.h` | `VPID`, `VOLID`, `PAGEID`, `PAGE_PTR`, `PAGE_TYPE`, `FILEIO_PAGE`, `IO_PAGESIZE` |
| `file_io.h` | `fileio_read()`, `fileio_write()`, `fileio_get_volume_descriptor()`, `fileio_init_lsa_of_page()` |
| `disk_manager.h` | (via `page_buffer.h`) volume information, `DISK_ISVALID` |
| `double_write_buffer.hpp` | DWB flush coordination |
| `show_scan.h` | `pgbuf_start_scan()` — SHOW BUFFER STATUS |

### Cross-Module Dependencies — `src/transaction/`

| File | Usage |
|------|-------|
| `log_manager.h` | `log_Gl`, checkpoint LSA, `logpb_force_flush_pages()` |
| `log_append.hpp` | `log_append_undoredo_data2()` — TDE redo/undo logging |
| `log_impl.h` | `LOG_TDES`, `LOG_FIND_TDES()`, wait_msecs |
| `log_volids.hpp` | Volume ID classification helpers |
| `transaction_sr.h` | `logtb_is_interrupted()`, `logtb_find_wait_msecs()` |
| `mvcc.c` / `vacuum.c` | `VACUUM_IS_THREAD_VACUUM_WORKER()` macro — vacuum threads ignore LRU promotion |

### Cross-Module Dependencies — `src/base/`

| File | Usage |
|------|-------|
| `error_manager.h` | `er_set()`, `ASSERT_ERROR()`, `er_errid()` |
| `memory_alloc.h` | `db_private_alloc()`, `free_and_init()` |
| `system_parameter.h` | `prm_get_integer_value()`, `PRM_ID_PB_NBUFFERS`, `PRM_ID_PB_LRU_HOT_RATIO`, etc. |
| `perf_monitor.h` | `perfmon_inc_stat()`, `perfmon_pbx_fix()`, `perfmon_pbx_unfix()` |
| `critical_section.h` | CS macros used in hash chain locking |
| `memory_hash.h` | `mht_create()`, `mht_destroy()` — Aout hash tables |
| `environment_variable.h` | Environment config |
| `porting_inline.hpp` | `STATIC_INLINE`, `ALWAYS_INLINE` attribute macros |
| `tsc_timer.h` | High-resolution timing for performance counters |
| `resource_tracker.hpp` | Debug page fix tracking (`thread_p->get_pgbuf_tracker()`) |

### Cross-Module Dependencies — `src/thread/`

| File | Usage |
|------|-------|
| `thread_daemon.hpp` | `cubthread::daemon` — background daemon threads |
| `thread_entry_task.hpp` | `cubthread::entry_task` — flush daemon task base class |
| `thread_manager.hpp` | `cubthread::get_manager()->create_daemon()` |
| `thread_entry.hpp` | `THREAD_ENTRY`, `thread_p->private_lru_index`, `thread_p->m_holder_anchor` |

### Cross-Module Dependencies — Other

| File | Usage |
|------|-------|
| `lockfree_circular_queue.hpp` | `lockfree::circular_queue<T>` — lock-free queues for direct victims and flushed BCBs |
| `list_file.h` | `thread_get_sort_stats_active()` |
| `query_manager.h` | Query manager interaction |
| `xserver_interface.h` | Server-side interface |
| `btree_load.h` | B-tree load support |
| `boot_sr.h` | `BO_IS_FLUSH_DAEMON_AVAILABLE()` |
| `tde.h` | `TDE_ALGORITHM`, `tde_is_loaded()` |
| `scope_exit.hpp` | RAII scope exit helper |
| `numeric_opfunc.h` | Numeric helpers |
| `dbtype.h` | `DB_VALUE` types |
| `connection_error.h` | `SERVER_MODE`-only: connection error codes |
| `probes.h` | `ENABLE_SYSTEMTAP`: `CUBRID_PGBUF_HIT()`, `CUBRID_PGBUF_MISS()` |

### The `memory_wrapper.hpp` Rule

```c
// XXX: SHOULD BE THE LAST INCLUDE HEADER
#include "memory_wrapper.hpp"
```

This is faithfully observed in `page_buffer.c` — it is the last `#include` in the file.

### Reverse Dependencies — Who Includes `page_buffer.h`

```
src/transaction/log_manager.c
src/transaction/log_recovery.c
src/transaction/log_page_buffer.c
src/transaction/log_2pc.c
src/transaction/log_append.cpp
src/transaction/log_recovery_redo.hpp
src/transaction/lock_manager.c
src/transaction/mvcc.c
src/storage/file_io.c
src/storage/file_manager.c
src/storage/file_manager.h
src/storage/disk_manager.c
src/storage/heap_file.h
src/storage/overflow_file.c
src/storage/overflow_file.h
src/storage/slotted_page.c
src/storage/external_sort.c
src/storage/extendible_hash.c
src/query/vacuum.c
src/query/query_hash_scan.c
src/thread/thread_entry.cpp
src/communication/network_interface_sr.cpp
src/base/system_parameter.c
```

That is **24 source/header files** directly include `page_buffer.h`, making this one of the most widely depended-upon modules in the entire engine.

---

## 3. Preprocessor & Compilation

### Conditional Compilation Guards

| Guard | Effect |
|-------|--------|
| `SERVER_MODE` | Enables: pthread mutexes on BCB/LRU/hash/invalid-list, thread waiting queues (`next_wait_thrd`), daemon threads, direct victim queues, `is_flushing_victims`/`is_checkpoint` flags, `latch_last_thread`, BCB mutex monitor |
| `SA_MODE` | Stubs out all pthread operations with no-ops; single-threaded path |
| `NDEBUG` | Release build: uses `pgbuf_fix_release()` instead of `pgbuf_fix_debug()`; removes caller tracking from `fixed_at[]`; skips watcher magic checks |
| `!NDEBUG` | Debug build: every public function has a `_debug` variant taking `caller_file`, `caller_line`, `caller_func`; all macros in `page_buffer.h` delegate to `_debug` variants via `ARG_FILE_LINE_FUNC` |
| `CUBRID_DEBUG` | Extra-aggressive debugging: buffer guard bytes (`pgbuf_Guard[8]`), page scrambling on unfix, consistency checks, `pgbuf_dump()` / `pgbuf_dump_if_any_fixed()` |
| `ENABLE_SYSTEMTAP` | DTrace/SystemTap probe points: `CUBRID_PGBUF_HIT()`, `CUBRID_PGBUF_MISS()` |

### Important Macros Defined (Configuration)

```c
#define PGBUF_MINIMUM_BUFFERS       (MAX_NTRANS * 10)   // floor on pool size
#define PGBUF_DEFAULT_FIX_COUNT     7                   // initial holder entries per thread
#define PGBUF_NUM_ALLOC_HOLDER      10                  // holder entries per allocated array
#define PGBUF_FIX_COUNT_THRESHOLD   64                  // fix count above which page is "hot"
#define PGBUF_LRU_NBITS             16                  // bits for LRU list index in flags
#define PGBUF_LRU_LIST_MAX_COUNT    65536               // max distinct LRU lists
#define HASH_SIZE_BITS              20
#define PGBUF_HASH_SIZE             (1 << 20)           // 1,048,576 hash buckets
#define PGBUF_MAX_NEIGHBOR_PAGES    32                  // max pages in neighbor flush
#define PGBUF_MAX_PAGE_WATCHERS     64                  // max simultaneous watchers/page
#define PGBUF_MAX_PAGE_FIXED_BY_TRAN 64                // max pages a single thread may hold
#define PGBUF_FLUSH_VICTIM_BOOST_MULT 10               // max flush rate boost multiplier
#define PGBUF_CHKPT_MAX_FLUSH_RATE  1200               // pages/sec max during checkpoint
#define PGBUF_CHKPT_MIN_FLUSH_RATE  50                 // pages/sec min during checkpoint
#define PGBUF_CHKPT_BURST_PAGES     16                 // burst pages per checkpoint interval
#define PGBUF_PRIVATE_LRU_MIN_COUNT 4                  // floor on private LRU BCB count
#define PGBUF_PRIVATE_LRU_MAX_HARD_QUOTA 5000          // ceiling on private LRU quota
#define PGBUF_MIN_PAGES_IN_SHARED_LIST 1000            // minimum BCBs per shared list
#define PGBUF_TRAN_THRESHOLD_ACTIVITY (pgbuf_Pool.num_buffers / 4)
#define PGBUF_TRAN_MAX_ACTIVITY (10 * PGBUF_TRAN_THRESHOLD_ACTIVITY)
```

### BCB Flags (Bitmask, stored in `bcb->flags` as `volatile int`)

| Flag | Bit | Meaning |
|------|-----|---------|
| `PGBUF_BCB_DIRTY_FLAG` | 0x80000000 | Page has been modified since last flush |
| `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` | 0x40000000 | Flush is in progress; do not victimize |
| `PGBUF_BCB_VICTIM_DIRECT_FLAG` | 0x20000000 | BCB assigned as direct victim to a waiting thread |
| `PGBUF_BCB_INVALIDATE_DIRECT_VICTIM_FLAG` | 0x10000000 | Direct victim was invalidated (page was re-fixed) |
| `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG` | 0x08000000 | Move BCB to bottom on next unfix (set for deallocated pages) |
| `PGBUF_BCB_TO_VACUUM_FLAG` | 0x04000000 | Page should be vacuumed |
| `PGBUF_BCB_ASYNC_FLUSH_REQ` | 0x02000000 | Asynchronous flush requested; write latch holder must flush on unfix |

### LRU Zone Encoding (in `flags` field's upper bits)

```c
typedef enum {
  PGBUF_LRU_1_ZONE     = 1 << 16,  // hot zone — no victimization
  PGBUF_LRU_2_ZONE     = 2 << 16,  // buffer zone — boosting allowed
  PGBUF_LRU_3_ZONE     = 3 << 16,  // victim zone — clean pages evicted here
  PGBUF_INVALID_ZONE   = 1 << 18,  // newly allocated / on invalid list
  PGBUF_VOID_ZONE      = 2 << 18,  // transitional — between allocation and LRU insertion
} PGBUF_ZONE;
```

The lower 16 bits of `flags` store the LRU list index (0–65535). The mask `PGBUF_LRU_INDEX_MASK = 0x0000FFFF` extracts it.

### Atomic Latch Layout (`PGBUF_ATOMIC_LATCH`, a `std::atomic<uint64_t>`)

```
Bits 63–48 : PGBUF_LATCH_MODE  (latch_mode — uint16_t)
Bits 47–32 : waiter_exists     (uint16_t: 0 or 1)
Bits 31–0  : fcnt              (int32_t: fix count)
```

This 64-bit atomic word is manipulated with CAS loops in `set_latch()`, `add_fcnt()`, `set_latch_and_fcnt()`, `set_latch_and_add_fcnt()`, etc., allowing lock-free read-only fix/unfix paths.

### Fix/Avoid-Dealloc Counter Layout (`count_fix_and_avoid_dealloc`)

```c
#define PGBUF_BCB_COUNT_FIX_SHIFT_BITS  16
#define PGBUF_BCB_AVOID_DEALLOC_MASK    ((int) 0x0000FFFF)
// Upper 16 bits: fix count saturation counter (up to FIX_COUNT_THRESHOLD)
// Lower 16 bits: avoid-deallocation reference count
```

---

## 4. Data Structures & Types

### 4.1 `PGBUF_BCB` — Buffer Control Block

Defined at line 503. One per buffer frame. This is the heart of the page buffer manager.

```c
struct pgbuf_bcb
{
  pthread_mutex_t mutex;           // [SERVER_MODE] per-BCB mutex
  int owner_mutex;                 // [SERVER_MODE] thread index holding mutex (-1 if free)
  VPID vpid;                       // {volid, pageid} of the page currently in this buffer
  PGBUF_ATOMIC_LATCH atomic_latch; // std::atomic<uint64_t>: latch_mode | waiter_exists | fcnt
  volatile int flags;              // BCB flags (dirty, flushing, victim, etc.) + zone + LRU index
  THREAD_ENTRY *next_wait_thrd;    // [SERVER_MODE] linked list of threads waiting for this BCB
  THREAD_ENTRY *latch_last_thread; // [SERVER_MODE] last thread to acquire latch (debug)
  PGBUF_BCB *hash_next;            // next BCB in hash chain for same bucket
  PGBUF_BCB *prev_BCB;             // previous BCB in LRU doubly-linked list
  PGBUF_BCB *next_BCB;             // next BCB in LRU or invalid list
  int tick_lru_list;               // list tick when BCB was inserted — used to detect "old enough" for boost
  int tick_lru3;                   // position counter in LRU zone 3 — used to update victim hint
  volatile int count_fix_and_avoid_dealloc; // upper 16: hot-page fix count; lower 16: avoid-dealloc refcount
  int hit_age;                     // quota.adjust_age at last hit — used in quota adjustment
  LOG_LSA oldest_unflush_lsa;      // oldest LSA record not yet flushed (WAL tracking)
  PGBUF_IOPAGE_BUFFER *iopage_buffer; // pointer to the actual page data buffer
};
```

**Key design notes:**
- `vpid` is the identity of the currently resident page. It is `NULL_VPID` when the BCB is on the invalid list.
- `atomic_latch` encodes latch mode, waiter presence, and fix count in a single 64-bit atomic, enabling a **lock-free fast path** for read-only fixes (`pgbuf_lockfree_fix_ro()`).
- `flags` uses bitmask encoding that combines zone, LRU index, and status flags in a single 32-bit word.
- `oldest_unflush_lsa` is the WAL anchor — a BCB cannot be flushed until all log records up to this LSA are written to the log file.

### 4.2 `PGBUF_IOPAGE_BUFFER`

```c
struct pgbuf_iopage_buffer
{
  PGBUF_BCB *bcb;      // back-pointer to controlling BCB
  int dummy;           // alignment pad (32-bit platforms only)
  FILEIO_PAGE iopage;  // actual page data: {FILEIO_PAGE_RESERVED prv; char page[DB_PAGESIZE];}
};
```

The `FILEIO_PAGE_RESERVED prv` header within `iopage` stores: `pageid`, `volid`, `ptype`, `pflag` (TDE bits), LSA, and reserved bytes.

**Pointer arithmetic:** `PAGE_PTR` (the externally visible page pointer) points to `iopage.page[0]`. The macros:
```c
CAST_PGPTR_TO_BFPTR(bufptr, pgptr)   // pgptr → BCB via offsetof(PGBUF_IOPAGE_BUFFER, iopage.page)
CAST_BFPTR_TO_PGPTR(pgptr, bufptr)   // BCB → pgptr
CAST_PGPTR_TO_IOPGPTR(io_pgptr, pgptr) // pgptr → FILEIO_PAGE *
```

### 4.3 `PGBUF_BUFFER_POOL` — The Pool Singleton

```c
struct pgbuf_buffer_pool
{
  int num_buffers;                      // total BCB count (from PRM_ID_PB_NBUFFERS)
  PGBUF_BCB *BCB_table;                 // flat array of all BCBs
  PGBUF_BUFFER_HASH *buf_hash_table;    // hash table (1M buckets)
  PGBUF_BUFFER_LOCK *buf_lock_table;    // one entry per thread — serializes concurrent fetches of same page
  PGBUF_IOPAGE_BUFFER *iopage_table;    // flat array of all page buffers
  int num_LRU_list;                     // number of shared LRU lists
  float ratio_lru1;                     // fraction of each LRU for hot zone (PRM_ID_PB_LRU_HOT_RATIO)
  float ratio_lru2;                     // fraction of each LRU for buffer zone
  PGBUF_LRU_LIST *buf_LRU_list;         // array of all LRU lists (shared + private)
  PGBUF_AOUT_LIST buf_AOUT_list;        // 2Q Aout history list
  PGBUF_INVALID_LIST buf_invalid_list;  // free BCB list (never-used pages)
  PGBUF_VICTIM_CANDIDATE_LIST *victim_cand_list;  // candidate buffer for flush thread
  PGBUF_SEQ_FLUSHER seq_chkpt_flusher;  // checkpoint sequential flusher state
  PGBUF_PAGE_MONITOR monitor;           // counters: hits, misses, activity, victim requests
  PGBUF_PAGE_QUOTA quota;               // private LRU quota system
  PGBUF_HOLDER_ANCHOR *thrd_holder_info;   // per-thread holder anchor array
  PGBUF_HOLDER *thrd_reserved_holder;      // pre-allocated holder entries
  pthread_mutex_t free_holder_set_mutex;   // guards free_holder_set
  PGBUF_HOLDER_SET *free_holder_set;       // overflow holder set (malloc'd on demand)
  int free_index;                          // current free index in free_holder_set
  bool check_for_interrupts;               // set by log manager; triggers interrupt checks in fix
  bool is_flushing_victims;                // [SERVER_MODE] flush thread active
  bool is_checkpoint;                      // [SERVER_MODE] checkpoint active
  PGBUF_DIRECT_VICTIM direct_victims;      // [SERVER_MODE] direct victim assignment system
  lockfree::circular_queue<PGBUF_BCB *> *flushed_bcbs;  // post-flush queue
  lockfree::circular_queue<int> *private_lrus_with_victims;
  lockfree::circular_queue<int> *big_private_lrus_with_victims;
  lockfree::circular_queue<int> *shared_lrus_with_victims;
  PGBUF_STATUS *show_status;              // per-transaction stats for SHOW BUFFER STATUS
  PGBUF_STATUS_OLD show_status_old;
  PGBUF_STATUS_SNAPSHOT show_status_snapshot;
  pthread_mutex_t show_status_mutex;
};
```

**Singleton:** `static PGBUF_BUFFER_POOL pgbuf_Pool;` at line 836.

### 4.4 `PGBUF_LRU_LIST` — Tripartite LRU List

```c
struct pgbuf_lru_list
{
  pthread_mutex_t mutex;           // list-level mutex
  PGBUF_BCB *top;                  // most-recently-used end
  PGBUF_BCB *bottom;               // least-recently-used end
  PGBUF_BCB *bottom_1;             // last BCB in zone 1 (NULL if zone 1 empty)
  PGBUF_BCB *bottom_2;             // last BCB in zone 2 (NULL if zone 2 empty)
  PGBUF_BCB *volatile victim_hint; // hint for victim search start point (below = dirty)
  int count_lru1;                  // BCBs in zone 1
  int count_lru2;                  // BCBs in zone 2
  int count_lru3;                  // BCBs in zone 3 (victimizable)
  int count_vict_cand;             // clean BCBs in zone 3 (immediately victimizable)
  int threshold_lru1;              // target size for zone 1
  int threshold_lru2;              // target size for zone 2
  int quota;                       // private list BCB quota
  int tick_list;                   // incremented on every insertion/boost
  int tick_lru3;                   // incremented when BCBs fall to zone 3
  volatile int flags;              // PGBUF_LRU_VICTIM_LFCQ_FLAG
  int index;                       // this list's index in buf_LRU_list array
};
```

**Zone semantics:**
- **Zone 1 (hot):** Most recently used BCBs. No victimization. Vacuum threads and other "ignorable" unfixers do not promote here.
- **Zone 2 (buffer):** BCBs that have aged out of zone 1. Boosting back to top allowed if BCB is "old enough" (tick age test). No victimization.
- **Zone 3 (victim):** The victimization zone. Clean BCBs here are candidates for replacement. Dirty BCBs here must be flushed first.

### 4.5 `PGBUF_BUFFER_HASH`

```c
struct pgbuf_buffer_hash
{
  pthread_mutex_t hash_mutex;   // protects hash chain and buffer lock chain
  PGBUF_BCB *hash_next;         // head of hash chain for this bucket
  PGBUF_BUFFER_LOCK *lock_next; // head of buffer lock chain
};
```

**Size:** `PGBUF_HASH_SIZE = 1 << 20 = 1,048,576` buckets.

### 4.6 `PGBUF_BUFFER_LOCK`

```c
struct pgbuf_buffer_lock
{
  VPID vpid;                         // page being fetched from disk
  PGBUF_BUFFER_LOCK *lock_next;      // next lock in chain
  THREAD_ENTRY *next_wait_thrd;      // [SERVER_MODE] threads waiting for this page to load
};
```

Used to serialize multiple threads that simultaneously request the same page that is not yet in the buffer pool. One thread fetches from disk; others wait on this lock.

### 4.7 `PGBUF_HOLDER` — Per-Thread Page Fix Record

```c
struct pgbuf_holder
{
  int fix_count;                // number of times this thread has fixed this BCB
  PGBUF_BCB *bufptr;            // which BCB is held
  PGBUF_HOLDER *thrd_link;      // next holder in this thread's used list
  PGBUF_HOLDER *next_holder;    // next holder in thread's free list
  PGBUF_HOLDER_STAT perf_stat;  // dirty_before_hold, dirtied_by_holder, hold_has_write/read_latch
  char fixed_at[64*1024];       // [!NDEBUG] file:line of each fix call (up to 64KB)
  int fixed_at_size;            // [!NDEBUG] used bytes in fixed_at
  int watch_count;              // number of PGBUF_WATCHER structs attached
  PGBUF_WATCHER *first_watcher; // doubly-linked watcher list
  PGBUF_WATCHER *last_watcher;
};
```

### 4.8 `PGBUF_HOLDER_ANCHOR` — Per-Thread Holder List Head

```c
struct pgbuf_holder_anchor
{
  int num_free_cnt;           // free holder entry count
  int num_hold_cnt;           // used holder entry count
  PGBUF_HOLDER *thrd_free_list;  // free entries
  PGBUF_HOLDER *thrd_hold_list;  // entries currently holding a BCB
};
```

Each thread has one `PGBUF_HOLDER_ANCHOR` stored in `pgbuf_Pool.thrd_holder_info[thread_index]`.

### 4.9 `PGBUF_AOUT_LIST` — 2Q Algorithm "Aout" History

```c
struct pgbuf_aout_list
{
  pthread_mutex_t Aout_mutex;
  PGBUF_AOUT_BUF *Aout_top;     // newest evicted VPID
  PGBUF_AOUT_BUF *Aout_bottom;  // oldest evicted VPID
  PGBUF_AOUT_BUF *Aout_free;    // free node list
  PGBUF_AOUT_BUF *bufarray;     // pre-allocated node array
  int num_hashes;               // number of hash tables for fast lookup
  MHT_TABLE **aout_buf_ht;      // hash tables (indexed by pageid % num_hashes)
  int max_count;                // size limit (PRM_ID_PB_AOUT_RATIO * num_buffers)
};
```

The Aout list implements the **2Q** page replacement enhancement. When a page is victimized from the LRU, its VPID is added to the Aout FIFO. If the same page is re-requested and it is found in Aout, it is a "second chance hit" and gets placed at the top of its LRU (hot zone), bypassing the normal middle insertion. This prevents sequential scans from displacing hot pages.

### 4.10 `PGBUF_DIRECT_VICTIM` — SERVER_MODE Direct Victim Assignment

```c
struct pgbuf_direct_victim
{
  PGBUF_BCB **bcb_victims;    // per-thread slot for pre-assigned BCB (indexed by thread index)
  lockfree::circular_queue<THREAD_ENTRY *> *waiter_threads_high_priority;
  lockfree::circular_queue<THREAD_ENTRY *> *waiter_threads_low_priority;
};
```

When a thread cannot find a victim from the LRU lists, it enqueues itself in one of these queues and sleeps. The page flush daemon or maintenance daemon dequeues a waiting thread and assigns it a BCB directly (avoiding the overhead of the thread searching all LRU lists again).

### 4.11 `PGBUF_WATCHER` — Ordered Fix Watcher (Public API)

Defined in `page_buffer.h`:

```c
struct pgbuf_watcher
{
  PAGE_PTR pgptr;                 // the fixed page pointer
  PGBUF_WATCHER *next;            // next watcher on same holder
  PGBUF_WATCHER *prev;            // prev watcher on same holder
  PGBUF_ORDERED_GROUP group_id;   // VPID of heap file header (grouping key)
  unsigned latch_mode:7;
  unsigned page_was_unfixed:1;    // set if any refix occurred
  unsigned initial_rank:4;        // rank at init (PGBUF_ORDERED_RANK)
  unsigned curr_rank:4;           // rank after fix
  unsigned int magic;             // [!NDEBUG] PGBUF_WATCHER_MAGIC_NUMBER = 0x12345678
  char watched_at[128];           // [!NDEBUG] source location of watch
  char init_at[256];              // [!NDEBUG] source location of init
};
```

### 4.12 Other Significant Structs

| Struct | Purpose |
|--------|---------|
| `PGBUF_SEQ_FLUSHER` | Controls rate-limited sequential flush during checkpoint; tracks intervals, burst mode, control counters |
| `PGBUF_VICTIM_CANDIDATE_LIST` | `{PGBUF_BCB *bufptr; VPID vpid}` — list of dirty BCBs collected for flushing |
| `PGBUF_BATCH_FLUSH_HELPER` | Gathers up to 63 neighbor pages for sequential I/O coalescing |
| `PGBUF_FIX_PERF` | Local perf tracking during `pgbuf_fix()` — tick counts, timing, page type |
| `PGBUF_PAGE_MONITOR` | Global counters: `dirties_cnt`, `lru_hits[]`, `lru_activity[]`, `fix_req_cnt`, `pg_unfix_cnt`, `lru_victim_req_cnt` |
| `PGBUF_PAGE_QUOTA` | Per-LRU flush priority, private session counts, private/shared ratio, adjust timestamps |
| `PGBUF_STATUS` / `PGBUF_STATUS_SNAPSHOT` | Show-status counters: `num_hit`, `num_page_request`, `num_pages_written`, etc. |
| `PGBUF_HOLDER_INFO` | Ordered-fix helper: collects VPID, group_id, rank, watchers for a page being re-fixed |
| `PGBUF_DEALLOC_UNDO_DATA` | Undo data for `pgbuf_rv_dealloc_undo()` — stored in `FILEIO_PAGE_RESERVED` |

### 4.13 Public Enumerations (from `page_buffer.h`)

| Enum | Values |
|------|--------|
| `PAGE_FETCH_MODE` | `OLD_PAGE`, `NEW_PAGE`, `OLD_PAGE_IF_IN_BUFFER`, `OLD_PAGE_PREVENT_DEALLOC`, `OLD_PAGE_DEALLOCATED`, `OLD_PAGE_MAYBE_DEALLOCATED`, `RECOVERY_PAGE` |
| `PGBUF_LATCH_MODE` | `PGBUF_NO_LATCH=0`, `PGBUF_LATCH_READ=1`, `PGBUF_LATCH_WRITE=2`, `PGBUF_LATCH_FLUSH=3`, `PGBUF_LATCH_INVALID=4` |
| `PGBUF_LATCH_CONDITION` | `PGBUF_UNCONDITIONAL_LATCH`, `PGBUF_CONDITIONAL_LATCH` |
| `PGBUF_PROMOTE_CONDITION` | `PGBUF_PROMOTE_ONLY_READER`, `PGBUF_PROMOTE_SHARED_READER` |
| `PGBUF_ORDERED_RANK` | `PGBUF_ORDERED_HEAP_HDR=0`, `PGBUF_ORDERED_HEAP_NORMAL`, `PGBUF_ORDERED_HEAP_OVERFLOW`, `PGBUF_ORDERED_RANK_UNDEFINED` |
| `PGBUF_DEBUG_PAGE_VALIDATION_LEVEL` | `NO_PAGE_VALIDATION`, `VALIDATION_FETCH`, `VALIDATION_FREE`, `VALIDATION_ALL` |

---

## 5. Global & Static Variables

### The Buffer Pool Singleton

```c
static PGBUF_BUFFER_POOL pgbuf_Pool;         // line 836 — the single buffer pool instance
static PGBUF_BATCH_FLUSH_HELPER pgbuf_Flush_helper; // line 837 — neighbor flush scratch space
```

### Public External Variables

```c
HFID *pgbuf_ordered_null_hfid = NULL;        // null HFID sentinel for ordered fix
const VPID vpid_Null_vpid = { NULL_PAGEID, NULL_VOLID }; // null VPID constant
```

### Debug-Only Variables

```c
#if defined(CUBRID_DEBUG)
static char pgbuf_Guard[8] = { MEM_REGION_GUARD_MARK, ... }; // guard bytes for overrun detection
#endif
```

### Server-Mode Daemon Handles

```c
#if defined(SERVER_MODE)
static cubthread::daemon *pgbuf_Page_maintenance_daemon = NULL;
static cubthread::daemon *pgbuf_Page_flush_daemon       = NULL;
static cubthread::daemon *pgbuf_Page_post_flush_daemon  = NULL;
static cubthread::daemon *pgbuf_Flush_control_daemon    = NULL;
#endif
```

### Release-Mode Abort Line Tracker

```c
#if defined(NDEBUG)
static int pgbuf_Abort_release_line = 0;  // saves __LINE__ before abort() in release builds
#endif
```

### BCB Mutex Monitor

```c
#if defined(SERVER_MODE)
static bool pgbuf_Monitor_locks = false;  // enabled by PRM_ID_PB_MONITOR_LOCKS (always true in !NDEBUG)
#endif
```

---

## 6. Function Catalog

### 6.1 Public API Functions

---

#### `pgbuf_initialize(void) → int`
**Signature:** `int pgbuf_initialize (void);`
**Type:** Public API

Initializes the entire page buffer pool. Reads `PRM_ID_PB_NBUFFERS` for pool size (minimum `MAX_NTRANS * 10`). Sets LRU zone ratios from `PRM_ID_PB_LRU_HOT_RATIO` and `PRM_ID_PB_LRU_BUFFER_RATIO`. Calls in order: `pgbuf_initialize_page_quota_parameters()`, `pgbuf_initialize_bcb_table()`, `pgbuf_initialize_hash_table()`, `pgbuf_initialize_lock_table()`, `pgbuf_initialize_lru_list()`, `pgbuf_initialize_invalid_list()`, `pgbuf_initialize_aout_list()`, `pgbuf_initialize_thrd_holder()`, `pgbuf_initialize_page_quota()`, `pgbuf_initialize_page_monitor()`. Allocates victim candidate list, direct victim queues, lock-free circular queues, and show_status array. On any error, calls `pgbuf_finalize()` to clean up and returns `ER_FAILED`.

---

#### `pgbuf_finalize(void) → void`
**Signature:** `void pgbuf_finalize (void);`
**Type:** Public API

Destroys all buffer pool resources. Destroys hash mutexes, BCB mutexes, LRU mutexes, invalid-list mutex, free-holder-set mutex. Frees all `malloc()`'d arrays. Deletes C++ lock-free queue objects. Called both during normal shutdown and as cleanup on initialization failure.

---

#### `pgbuf_fix(thread_p, vpid, fetch_mode, requestmode, condition) → PAGE_PTR`
**Macro + function:** In debug builds expands to `pgbuf_fix_debug(..., ARG_FILE_LINE_FUNC)`. In release builds calls `pgbuf_fix_release(...)`.
**Type:** Public API — most frequently called function in the engine

The central page-pinning operation. Full algorithm described in Section 7.1.

**Parameters:**
- `vpid` — {volid, pageid} identifying the desired page
- `fetch_mode` — `OLD_PAGE` (existing), `NEW_PAGE` (freshly allocated), `RECOVERY_PAGE`, etc.
- `requestmode` — `PGBUF_LATCH_READ` or `PGBUF_LATCH_WRITE`
- `condition` — `PGBUF_UNCONDITIONAL_LATCH` (may block) or `PGBUF_CONDITIONAL_LATCH` (fails immediately if unavailable)

Returns `PAGE_PTR` on success, `NULL` on error (error set via `er_set()`).

---

#### `pgbuf_unfix(thread_p, pgptr) → void`
**Macro:** In debug expands to `pgbuf_unfix_debug(..., ARG_FILE_LINE_FUNC)`.
**Type:** Public API

Releases one fix on a page. Decrements fix count. If fix count reaches zero, the page is eligible for LRU management. Calls `pgbuf_unlatch_thrd_holder()` to update holder accounting, then `pgbuf_unlatch_bcb_upon_unfix()` to handle LRU positioning and wake up any waiters. In `CUBRID_DEBUG` mode, checks for overruns, dirty-without-log violations, and consistency.

---

#### `pgbuf_unfix_all(thread_p) → void`
**Type:** Public API

Emergency unfix of all pages held by `thread_p`. Called at request termination. In debug mode, logs each remaining held page. In release mode, calls `pgbuf_unfix_and_init()` for each.

---

#### `pgbuf_fix_with_retry(thread_p, vpid, fetch_mode, request_mode, retry) → PAGE_PTR`
**Type:** Public API

Wrapper around `pgbuf_fix()` that retries up to `retry` times on latch timeout errors (`ER_LK_PAGE_TIMEOUT`, `ER_PAGE_LATCH_TIMEDOUT`). Used in code paths that cannot block indefinitely.

---

#### `pgbuf_ordered_fix(thread_p, req_vpid, fetch_mode, requestmode, req_watcher) → int`
**Macro + function.**
**Type:** Public API

Deadlock-free ordered page fix for heap files. Uses `PGBUF_WATCHER` and `PGBUF_ORDERED_RANK` to maintain a canonical ordering: heap header > heap normal > heap overflow. If fixing the requested page would violate the order (because a lower-ranked page is already held), unfixes all conflicting pages and refixes in the correct order. Calls `pgbuf_get_groupid_and_unfix()` to determine the heap group and whether reordering is needed.

---

#### `pgbuf_promote_read_latch(thread_p, pgptr_p, condition) → int`
**Macro + function.**
**Type:** Public API

Promotes a READ latch to a WRITE latch on an already-fixed page. Two conditions:
- `PGBUF_PROMOTE_ONLY_READER`: only succeeds if the current thread is the sole reader.
- `PGBUF_PROMOTE_SHARED_READER`: succeeds even if multiple readers hold the page, by releasing the read latch, waiting as first blocker (with `wait_for_latch_promote` flag), and re-acquiring as writer.

Returns `ER_PAGE_LATCH_PROMOTE_FAIL` (non-fatal) if promotion cannot be done cleanly.

---

#### `pgbuf_flush(thread_p, pgptr, free_page) → void`
**Type:** Public API

Flushes `pgptr` to disk (only if dirty) following WAL, then optionally unfixes. Calls `pgbuf_flush_with_wal()`. The comment warns against general use because the caller cannot react to flush failure.

---

#### `pgbuf_flush_with_wal(thread_p, pgptr) → PAGE_PTR`
**Type:** Public API

Flushes a fixed, write-latched page to disk after ensuring the WAL is satisfied. Calls `pgbuf_bcb_safe_flush_force_unlock()`. Returns `NULL` on failure.

---

#### `pgbuf_flush_if_requested(thread_p, page) → void`
**Type:** Public API

Called periodically by threads that hold a write latch for a long time (e.g., log manager). If `PGBUF_BCB_ASYNC_FLUSH_REQ` is set on the BCB, flushes immediately.

---

#### `pgbuf_flush_all(thread_p, volid) → int`
**Type:** Public API

Flushes all dirty pages for `volid` (or all volumes if `volid == NULL_VOLID`). Used by log/recovery manager.

---

#### `pgbuf_flush_all_unfixed(thread_p, volid) → int`
**Type:** Public API

Like `pgbuf_flush_all()` but skips fixed pages.

---

#### `pgbuf_flush_all_unfixed_and_set_lsa_as_null(thread_p, volid) → int`
**Type:** Public API

Like `pgbuf_flush_all_unfixed()` but also resets page LSA to null before flushing. Used during database restore/creation.

---

#### `pgbuf_flush_victim_candidates(thread_p, flush_ratio, time_tracker, stop) → int`
**Type:** Public API (called by page flush daemon)

The main flushing loop. Collects dirty victim candidates from LRU zone 3 via `pgbuf_get_victim_candidates_from_lru()`, sorts them by VPID for sequential I/O if `PRM_ID_PB_SEQUENTIAL_VICTIM_FLUSH` is enabled, then flushes each using `pgbuf_bcb_flush_with_wal()` with neighbor coalescing (`pgbuf_flush_page_and_neighbors_fb()`). Applies a dynamic flush rate boost based on the miss rate: up to `PGBUF_FLUSH_VICTIM_BOOST_MULT = 10×` the configured ratio when the miss rate is high.

---

#### `pgbuf_flush_checkpoint(thread_p, flush_upto_lsa, prev_chkpt_redo_lsa, smallest_lsa, flushed_page_cnt) → int`
**Type:** Public API (called by checkpoint thread)

Checkpoint flush. Scans all BCBs to find dirty pages with `oldest_unflush_lsa <= flush_upto_lsa`. Builds a sorted candidate list and calls `pgbuf_flush_chkpt_seq_list()` → `pgbuf_flush_seq_list()` for rate-controlled flushing. Outputs the smallest remaining unflushed LSA.

---

#### `pgbuf_invalidate(thread_p, pgptr) → int`
**Type:** Public API

Invalidates a fixed page. If fix count > 1, just unfixes. If fix count == 1, flushes if dirty, unfixes, then removes from hash and puts BCB on invalid list. This is a performance hint — frees the BCB frame for immediate reuse. Must be called for permanent pages only after commit decision.

---

#### `pgbuf_invalidate_all(thread_p, volid) → int`
**Type:** Public API

Invalidates all unfixed pages for `volid`. Scans the entire BCB table. Flushes dirty ones first.

---

#### `pgbuf_set_dirty(thread_p, pgptr, free_page) → void`
**Macro + function.**
**Type:** Public API

Marks a page as dirty (calls `pgbuf_set_dirty_buffer_ptr()`). Optionally unfixes. The underlying `pgbuf_set_dirty_buffer_ptr()` sets `PGBUF_BCB_DIRTY_FLAG` and increments `monitor.dirties_cnt`.

---

#### `pgbuf_get_lsa(pgptr) → LOG_LSA *`
**Type:** Public API

Returns pointer to `iopage.prv.lsa` — the page's current LSA. No mutex needed (page is fixed).

---

#### `pgbuf_set_lsa(thread_p, pgptr, lsa_ptr) → const LOG_LSA *`
**Macro + function.**
**Type:** Public API

Sets the page LSA and, if this is the first LSA assignment since last flush, records `lsa_ptr` as `oldest_unflush_lsa`. In release builds, also forces `pgbuf_set_dirty_buffer_ptr()` as a safety net for missing dirty marks.

---

#### `pgbuf_copy_to_area / pgbuf_copy_from_area`
**Type:** Public API

Copies page content to/from a caller-provided buffer without requiring the caller to hold the page fixed for the duration. If the page is already in the buffer pool, copies directly from/to the BCB and releases the BCB mutex. Otherwise optionally fetches the page.

---

#### `pgbuf_set_tde_algorithm(thread_p, pgptr, tde_algo, skip_logging) → void`
#### `pgbuf_get_tde_algorithm(pgptr) → TDE_ALGORITHM`
#### `pgbuf_rv_set_tde_algorithm(thread_p, rcv) → int`
**Type:** Public API

Manage Transparent Data Encryption algorithm for a page. Algorithm stored in `iopage.prv.pflag` bits (`FILEIO_PAGE_FLAG_ENCRYPTED_AES`, `FILEIO_PAGE_FLAG_ENCRYPTED_ARIA`). The `rv_` variant is the recovery handler for redo/undo.

---

#### `pgbuf_peek_stats(...)` / `pgbuf_daemons_get_stats(...)`
**Type:** Public API

Exposes buffer pool statistics for monitoring (SHOW BUFFER STATUS).

---

#### `pgbuf_adjust_quotas(thread_p) → void`
#### `pgbuf_assign_private_lru(thread_p) → int`
#### `pgbuf_release_private_lru(thread_p, private_idx) → int`
**Type:** Public API

Quota management for per-transaction private LRU lists. `pgbuf_assign_private_lru()` gives a thread its own private LRU index. `pgbuf_release_private_lru()` releases it back. `pgbuf_adjust_quotas()` periodically recalculates quotas based on LRU activity.

---

#### `pgbuf_simple_fix(thread_p, vpid, need_fix) → PAGE_PTR`
#### `pgbuf_simple_unfix(thread_p, pgptr) → void`
**Type:** Public API (temporary file access only)

Lock-free, latch-free pin/unpin for **temporary file** pages only. Increments/decrements `fcnt` directly without entering the full fix/unfix path. **Cannot be mixed with `pgbuf_fix()`/`pgbuf_unfix()` on the same page.**

---

#### Recovery API Functions

```c
pgbuf_log_new_page(thread_p, page_new, data_size, ptype_new)  → void
pgbuf_log_redo_new_page(thread_p, page_new, data_size, ptype_new) → void
pgbuf_rv_new_page_redo(thread_p, rcv) → int
pgbuf_rv_new_page_undo(thread_p, rcv) → int
pgbuf_dealloc_page(thread_p, page_dealloc) → void
pgbuf_rv_dealloc_redo(thread_p, rcv) → int
pgbuf_rv_dealloc_undo(thread_p, rcv) → int
pgbuf_rv_dealloc_undo_compensate(thread_p, rcv) → int
pgbuf_rv_flush_page(thread_p, rcv) → int
```

These implement the recoverable page alloc/dealloc protocol. `pgbuf_log_new_page()` writes an undo-redo log record for new page initialization; on undo, `pgbuf_rv_new_page_undo()` marks the page as `PAGE_UNKNOWN`. `pgbuf_dealloc_page()` transitions a page to the deallocated state with appropriate log records.

---

### 6.2 Static Internal Functions (Selected Key Functions)

---

#### `pgbuf_latch_bcb_upon_fix(thread_p, bufptr, request_mode, buf_lock_acquired, condition, is_latch_wait) → int`
**Type:** Static inline helper

Called from `pgbuf_fix()` after the BCB is found (or newly allocated). Uses a CAS loop on `bufptr->atomic_latch` to attempt granting the latch. Three outcomes:
1. **Idle page** (latch_mode == NO_LATCH, fcnt == 0): grants immediately, allocates holder entry.
2. **Compatible latch** (READ on READ, or same thread write): increments fcnt, updates holder.
3. **Incompatible latch** (WRITE contention, or CONDITIONAL fails): either returns error or calls `pgbuf_block_bcb()` to sleep.

---

#### `pgbuf_unlatch_bcb_upon_unfix(thread_p, bufptr, holder_status) → int`
**Type:** Static inline

Core unfix logic. CAS-decrements `fcnt` in `atomic_latch`. If fcnt reaches zero, sets latch_mode to `NO_LATCH`. Then:
- If `PGBUF_BCB_MOVE_TO_LRU_BOTTOM_FLAG`: calls `pgbuf_move_bcb_to_bottom_lru()`.
- If blocked readers/writers exist: calls `pgbuf_wakeup_reader_writer()`.
- Otherwise: handles zone-specific LRU logic (zone 1 stays, zone 2 may boost, zone 3 always boosts, void zone calls `pgbuf_unlatch_void_zone_bcb()`).

---

#### `pgbuf_block_bcb(thread_p, bufptr, request_mode, request_fcnt, as_promote) → int`
**Type:** Static

Suspends the calling thread on `bufptr`'s wait queue. Adds thread to `bufptr->next_wait_thrd` linked list, sets `thread_p->request_latch_mode`, then calls `pgbuf_timed_sleep()`. On wake-up, allocates a holder entry.

---

#### `pgbuf_claim_bcb_for_fix(thread_p, vpid, fetch_mode, hash_anchor, perf, try_again, already_locked) → PGBUF_BCB *`
**Type:** Static

The "miss path" in `pgbuf_fix()`. When the page is not found in the hash table:
1. Calls `pgbuf_allocate_bcb()` to get a free/victimized BCB frame.
2. If Aout list is enabled, checks if VPID is in Aout history (second-chance hit).
3. Reads the page from disk using `fileio_read()` (via `pgbuf_victimize_bcb()`).
4. Updates performance counters (`perf_page_found`, etc.).

---

#### `pgbuf_allocate_bcb(thread_p, src_vpid) → PGBUF_BCB *`
**Type:** Static

Allocates a BCB for a new page. First tries the **invalid list** (truly free BCBs), then calls `pgbuf_get_victim()` to victimize an LRU candidate. Registers the requesting thread in the direct victim queue if all searches fail, and sleeps waiting for a victim assignment.

---

#### `pgbuf_victimize_bcb(thread_p, bufptr) → int`
**Type:** Static

Prepares a selected victim BCB for reuse. Removes it from its current hash chain, flushes if dirty, invalidates the page content, and resets the BCB to an initial clean state.

---

#### `pgbuf_get_victim(thread_p) → PGBUF_BCB *`
**Type:** Static

Main victim selection function. Searches in this order:
1. Own private LRU list (if over quota).
2. Other private LRU lists (via `pgbuf_lfcq_get_victim_from_private_lru()`).
3. Shared LRU lists (via `pgbuf_lfcq_get_victim_from_shared_lru()`).
4. Own private LRU even if under quota (last resort).

Increments `monitor.lru_victim_req_cnt` (used to compute miss rate).

---

#### `pgbuf_get_victim_from_lru_list(thread_p, lru_idx) → PGBUF_BCB *`
**Type:** Static

Scans LRU zone 3 from the `victim_hint` position backward (toward the bottom). Skips dirty, fixed, and "avoid victim" BCBs. Returns the first victimizable BCB with its mutex held.

---

#### `pgbuf_bcb_flush_with_wal(thread_p, bufptr, is_page_flush_thread, is_bcb_locked) → int`
**Type:** Static inline

Performs the actual page flush. Protocol:
1. Sets `PGBUF_BCB_FLUSHING_TO_DISK_FLAG`.
2. Calls `log_append_get_nxlsa()` to get current log end; compares against `oldest_unflush_lsa`.
3. If log is behind: wakes log flush daemon (or calls `logpb_force_flush_pages()`).
4. Calls `dwb_set_data_on_next_slot()` + `fileio_write()` via the double-write buffer.
5. Clears dirty flag and flushing flag, updates `oldest_unflush_lsa` to null.
6. Adds BCB to `flushed_bcbs` queue for post-flush processing.

---

#### `pgbuf_bcb_safe_flush_internal(thread_p, bufptr, synchronous, locked) → int`
**Type:** Static

Safe flush that handles the concurrent write-latch case. If the page is not write-latched (or the caller holds the write latch), flushes immediately. If write-latched by another thread: sets `PGBUF_BCB_ASYNC_FLUSH_REQ` and optionally blocks (if `synchronous == true`) by calling `pgbuf_block_bcb(PGBUF_LATCH_FLUSH)`.

---

#### `pgbuf_search_hash_chain(thread_p, hash_anchor, vpid) → PGBUF_BCB *`
**Type:** Static inline

Two-phase hash lookup. Phase 1 (optimistic): scans the hash chain without the hash mutex, uses `PGBUF_BCB_TRYLOCK()` on each matching BCB. Phase 2 (under hash mutex): if trylock fails, acquires `hash_anchor->hash_mutex` and retries. Returns the BCB with its mutex held, or NULL if not found (leaving hash mutex held for the caller to handle the miss).

---

#### `pgbuf_lockfree_fix_ro(thread_p, vpid, fetch_mode) → PAGE_PTR`
**Type:** Static inline

**The hot path for read-only fixes.** Scans the hash chain without any BCB mutex. Uses `compare_exchange_strong` on `atomic_latch` to atomically increment `fcnt` only if the page is already in READ mode (or NO_LATCH transitioning to READ). If the CAS succeeds, allocates a holder entry and returns. Falls back to the normal path if the page is write-latched or not in the buffer. This avoids acquiring the per-BCB mutex for the common case of concurrent read fixes on a popular page.

---

#### `pgbuf_lockfree_unfix_ro(thread_p, bufptr) → bool`
**Type:** Static inline

Complementary to `pgbuf_lockfree_fix_ro()`. If the page is READ-latched (no write latch, no waiters), atomically decrements `fcnt` via CAS. Returns `true` (fast path taken, no mutex needed) if successful.

---

#### `pgbuf_lru_boost_bcb(thread_p, bcb) → void`
**Type:** Static

Moves a BCB from zone 2 or zone 3 to the top of its LRU list (zone 1). Acquires the LRU list mutex, removes from current position, inserts at top, and adjusts zone boundaries. Never boosts from zone 1.

---

#### `pgbuf_lru_fall_bcb_to_zone_3(thread_p, bcb, lru_list) → void`
**Type:** Static inline

Moves a BCB from zone 1 or 2 to zone 3 (victimization zone). Updates `tick_lru3`, adds to victim candidate count if clean.

---

#### `pgbuf_lru_adjust_zones(thread_p, lru_list, min_one) → void`
**Type:** Static inline

Rebalances zone 1 and zone 2 sizes relative to their thresholds. If zone 1 is over threshold, falls one BCB to zone 2. If zone 2 is over threshold, falls one BCB to zone 3. `min_one` ensures at least one BCB remains in zone 1.

---

#### `pgbuf_compute_lru_vict_target(lru_sum_flush_priority) → void`
**Type:** Static

Computes per-LRU flush priority weights for the flush daemon. LRUs with more victim candidates get proportionally higher priority.

---

#### `pgbuf_adjust_quotas(thread_p) → void`
**Type:** Public (called by maintenance daemon)

Runs periodically (every 100ms). Computes activity per private LRU, adjusts quotas based on usage, destroys inactive private LRUs, and updates `monitor.victim_rich`.

---

#### `pgbuf_wakeup_reader_writer(thread_p, bufptr) → void`
**Type:** Static inline (SERVER_MODE only)

Wakes up waiting threads on `bufptr->next_wait_thrd`. Grants READ latches to multiple consecutive readers simultaneously. Stops at the first WRITE waiter. Removes NO_LATCH (timed-out) entries from the queue.

---

#### `pgbuf_timed_sleep(thread_p, bufptr) → int`
**Type:** Static (SERVER_MODE only)

Implements timed wait for latch. Uses `thread_suspend_timeout_wakeup_and_unlock_entry()` with a timeout derived from `pgbuf_latch_timeout` (default 300 seconds). On timeout with `LK_INFINITE_WAIT`, sets `ER_LK_UNILATERALLY_ABORTED`. With finite wait, sets `ER_LK_PAGE_TIMEOUT`.

---

#### `pgbuf_hash_func_mirror(vpid) → unsigned int`
**Type:** Static inline

The internal hash function. Reverses the lower 8 bits of `volid` and XORs with `pageid`, masked to `HASH_SIZE_BITS = 20` bits. This ensures pages from different volumes hash differently even when pageids coincide.

---

#### `pgbuf_flags_mask_sanity_check() → void`
**Type:** Static

Called at startup in `pgbuf_initialize()`. Asserts that all BCB flag bit positions and zone masks are consistent with each other (prevents silent corruption from future bit layout changes).

---

### 6.3 Daemon Thread Functions (SERVER_MODE only)

| Function | Period | Purpose |
|----------|--------|---------|
| `pgbuf_page_maintenance_execute()` | 100 ms | `pgbuf_adjust_quotas()` + `pgbuf_direct_victims_maintenance()` |
| `pgbuf_page_flush_daemon_task::execute()` | `PRM_ID_PAGE_BG_FLUSH_INTERVAL_MSECS` | Main flush loop calling `pgbuf_flush_victim_candidates()` |
| `pgbuf_page_post_flush_execute()` | 1/10/100 ms (adaptive) | `pgbuf_assign_flushed_pages()` — post-flush BCB processing |
| `pgbuf_flush_control_daemon_task::execute()` | 50 ms | `fileio_flush_control_add_tokens()` — I/O rate limiting |

---

## 7. Key Algorithms & Logic Flows

### 7.1 `pgbuf_fix()` — Page Pin Algorithm

```
pgbuf_fix(thread_p, vpid, fetch_mode, request_mode, condition):

1. VALIDATE
   - request_mode must be READ or WRITE
   - condition must be UNCONDITIONAL or CONDITIONAL
   - increment monitor.fix_req_cnt

2. INTERRUPT CHECK
   - if logtb_is_interrupted(): return NULL with ER_INTERRUPTED

3. FAST PATH (READ-ONLY, UNCONDITIONAL)
   - if request_mode==READ and fetch_mode in {OLD_PAGE, OLD_PAGE_PREVENT_DEALLOC, OLD_PAGE_MAYBE_DEALLOCATED}:
     - attempt pgbuf_lockfree_fix_ro() — CAS on atomic_latch, no BCB mutex
     - if succeeds: goto fast_path (skip hash lookup entirely)

4. HASH LOOKUP
   - hash_anchor = &buf_hash_table[PGBUF_HASH_VALUE(vpid)]
   - bufptr = pgbuf_search_hash_chain(thread_p, hash_anchor, vpid)
   - returns BCB with bufptr->mutex held, or NULL with hash_anchor->hash_mutex held

5a. HIT PATH (bufptr != NULL)
   - invalidate direct victim flag if needed
   - goto LATCH_PASS

5b. MISS PATH (bufptr == NULL)
   - if OLD_PAGE_IF_IN_BUFFER: return NULL immediately
   - lock_page() — serialize concurrent fetches of same page
   - pgbuf_claim_bcb_for_fix():
     - pgbuf_get_bcb_from_invalid_list() OR pgbuf_get_victim()
     - if Aout hit: add to top of LRU (second chance)
     - fileio_read() to load page from disk
     - if second chance: add to top, else add to middle of LRU
   - buf_lock_acquired = true

6. REGISTER FIX
   - pgbuf_bcb_register_fix(bufptr)  (increments count_fix_and_avoid_dealloc upper bits)
   - pgbuf_set_bcb_page_vpid(bufptr) (verifies page VPID in iopage.prv matches)
   - pgbuf_check_bcb_page_vpid(bufptr, maybe_deallocated)

7. LATCH_PASS
   - pgbuf_latch_bcb_upon_fix(thread_p, bufptr, request_mode, buf_lock_acquired, condition, &is_latch_wait)
   - if condition==CONDITIONAL and latch unavailable: return NULL with ER_LK_PAGE_TIMEOUT
   - if UNCONDITIONAL and contention: pgbuf_block_bcb() → sleep

8. HASH CHAIN CONNECTION (if buf_lock_acquired)
   - pgbuf_insert_into_hash_chain(hash_anchor, bufptr)
   - pgbuf_unlock_page(hash_anchor, vpid, false)

9. fast_path:
   - validate page type vs fetch_mode (NEW_PAGE, deallocated checks)
   - if OLD_PAGE_PREVENT_DEALLOC: unregister avoid_deallocation
   - record performance stats
   - return PAGE_PTR
```

### 7.2 `pgbuf_unfix()` — Page Unpin Algorithm

```
pgbuf_unfix(thread_p, pgptr):

1. CAST: pgptr → bufptr (BCB)

2. DEBUG VALIDATION
   - pgbuf_is_valid_page_ptr(), watcher magic checks, dirty-without-log check

3. PERF TRACKING
   - get page type for stat

4. UNLATCH HOLDER
   - pgbuf_unlatch_thrd_holder():
     - decrements holder->fix_count
     - if fix_count == 0: removes holder entry from thread's hold list
     - collects perf stats into holder_perf_stat

5. FAST PATH (READ latch, no waiters)
   - if pgbuf_lockfree_unfix_ro(): CAS decrement fcnt; return immediately

6. SLOW PATH
   - PGBUF_BCB_LOCK(bufptr)
   - pgbuf_unlatch_bcb_upon_unfix():
     - CAS decrement fcnt in atomic_latch
     - if fcnt == 0: latch_mode = NO_LATCH
     - if MOVE_TO_LRU_BOTTOM: pgbuf_move_bcb_to_bottom_lru()
     - else if waiters: pgbuf_wakeup_reader_writer()
     - else: zone-specific LRU handling:
       * VOID_ZONE: pgbuf_unlatch_void_zone_bcb() — decides which LRU to add to
       * LRU_1_ZONE: stay in place; maybe move private→shared if over quota
       * LRU_2_ZONE: boost to top if bcb is old enough
       * LRU_3_ZONE: always boost to top
   - BCB_UNLOCK (happens inside pgbuf_unlatch_bcb_upon_unfix)
```

### 7.3 LRU Zone Maintenance and Zone Thresholds

Each LRU list maintains three zones by tracking three pointers: `bottom_1` (last BCB in zone 1), `bottom_2` (last BCB in zone 2), and `bottom` (last BCB in zone 3).

The zone sizes are controlled by:
- `threshold_lru1 = quota * ratio_lru1` (hot zone target)
- `threshold_lru2 = quota * ratio_lru2` (buffer zone target)

When a BCB is added to the top:
- Zone 1 grows → if over threshold, `pgbuf_lru_adjust_zone1()` moves `bottom_1` BCB to zone 2.
- Zone 2 grows → if over threshold, `pgbuf_lru_adjust_zone2()` moves `bottom_2` BCB to zone 3.

This waterfall mechanism maintains steady-state proportions.

### 7.4 Victim Selection

```
pgbuf_get_victim(thread_p):
  1. If thread has private LRU AND it is over quota:
     search own private list for clean zone-3 BCB
  2. Consume from big_private_lrus_with_victims queue (large private lists)
  3. Consume from private_lrus_with_victims queue
  4. Consume from shared_lrus_with_victims queue
  5. If still nothing and victim_rich: loop steps 1-4
  6. Last resort: search own private even if under quota
  7. If all fail: enqueue self in direct victim queue, sleep

pgbuf_get_victim_from_lru_list(thread_p, lru_idx):
  - Acquire lru_list->mutex
  - Start from victim_hint, scan toward bottom
  - Skip: fixed (fcnt > 0), dirty, flushing, direct_victim, avoid_dealloc BCBs
  - On success: PGBUF_BCB_LOCK(victim), advance victim_hint past it, return
```

### 7.5 Dirty Page Flush — WAL Protocol

```
pgbuf_bcb_flush_with_wal(thread_p, bufptr, is_page_flush_thread, is_bcb_locked):
  1. Mark PGBUF_BCB_FLUSHING_TO_DISK_FLAG (prevents concurrent flush/victimization)
  2. BCB_UNLOCK(bufptr) — release BCB mutex during I/O
  3. Compute need_wal_lsa = oldest_unflush_lsa
  4. Get current log LSA (log_append_get_nxlsa)
  5. If need_wal_lsa > current_log_lsa:
     - wake log flush daemon or call logpb_force_flush_pages()
     - wait for log to catch up
  6. TDE encryption if required (tde_encrypt_data_page())
  7. Double-write buffer: dwb_set_data_on_next_slot()
  8. fileio_write(volid, pageid, iopage)
  9. Increment PSTAT_PB_NUM_IOWRITES
  10. BCB_LOCK(bufptr)
  11. pgbuf_bcb_mark_was_flushed():
      - clear PGBUF_BCB_DIRTY_FLAG
      - clear PGBUF_BCB_FLUSHING_TO_DISK_FLAG
      - oldest_unflush_lsa = NULL_LSA
      - decrement monitor.dirties_cnt
  12. Wake flush waiters (threads waiting with PGBUF_LATCH_FLUSH)
  13. Enqueue to flushed_bcbs for post-flush processing
```

### 7.6 2Q Page Replacement

When a page is victimized (evicted from LRU), its VPID is added to the **Aout FIFO**. If the same page is subsequently requested and found in Aout:
- It is a "second-chance" page — it was recently evicted but is now needed again.
- It gets inserted at the **top** of the LRU (zone 1 position), bypassing the normal "new page goes to middle" policy.
- This prevents sequential scan pages from permanently displacing pages that were recently active but temporarily not needed.

Pages not in Aout are added to the **middle** of the LRU (boundary of zones 1 and 2), so truly cold pages go directly to zone 2 and can be victimized after aging.

### 7.7 Buffer Pool Initialization Sequence

```
pgbuf_initialize():
  1. pgbuf_flags_mask_sanity_check()
  2. pgbuf_initialize_page_quota_parameters()   — compute num_private_LRU_list
  3. pgbuf_initialize_bcb_table()               — malloc BCB_table + iopage_table
  4. pgbuf_initialize_hash_table()              — malloc buf_hash_table (1M entries)
  5. pgbuf_initialize_lock_table()              — malloc buf_lock_table (1 entry/thread)
  6. pgbuf_initialize_lru_list()                — malloc buf_LRU_list (shared+private)
  7. pgbuf_initialize_invalid_list()            — chain all BCBs into invalid list
  8. pgbuf_initialize_aout_list()               — malloc Aout bufarray + hash tables
  9. pgbuf_initialize_thrd_holder()             — malloc thrd_holder_info + reserved_holder
  10. pgbuf_initialize_page_quota()             — init per-LRU quota arrays
  11. pgbuf_initialize_page_monitor()           — init activity/hit counters
  12. malloc victim_cand_list (num_buffers entries)
  13. pgbuf_initialize_seq_flusher() for checkpoint flusher
  14. [SERVER_MODE] malloc direct_victims.bcb_victims (one slot per thread)
  15. [SERVER_MODE] new lockfree::circular_queue (waiter_threads_high/low_priority)
  16. [SERVER_MODE] new lockfree::circular_queue (flushed_bcbs, 8192 entries)
  17. new lockfree::circular_queue for private/big_private/shared_lrus_with_victims
  18. malloc show_status (MAX_NTRANS + 1 entries)
```

### 7.8 Checkpoint Sequential Flush

`pgbuf_flush_checkpoint()` drives checkpoint flushing. It first builds a sorted list of dirty BCBs with `oldest_unflush_lsa <= flush_upto_lsa`, then calls `pgbuf_flush_chkpt_seq_list()` → `pgbuf_flush_seq_list()`. The sequential flusher maintains a rate controller: it divides each second into intervals (`interval_msec`) and flushes a proportional quota per interval. Burst mode flushes all-at-once per interval; non-burst mode space-times each page write with a computed sleep to achieve a uniform flush rate (between `PGBUF_CHKPT_MIN_FLUSH_RATE=50` and `PGBUF_CHKPT_MAX_FLUSH_RATE=1200` pages/sec).

---

## 8. Concurrency & Thread Safety

### 8.1 Locking Hierarchy

The locking hierarchy must be respected to prevent deadlocks:

```
LRU list mutex  →  BCB mutex  →  hash anchor mutex  →  invalid list mutex
```

Most code follows: acquire LRU mutex → BCB mutex → release in reverse order. The key exception is `pgbuf_search_hash_chain()`, which uses a two-phase optimistic-then-lock approach.

### 8.2 Hash Anchor Mutex (`hash_anchor->hash_mutex`)

Protects the `hash_next` chain (hash chain) and `lock_next` chain (buffer lock records) for one hash bucket. Held very briefly: during hash chain search (slow path), BCB insertion, BCB deletion, buffer lock acquisition/release.

### 8.3 BCB Mutex (`bufptr->mutex`)

Per-BCB `pthread_mutex_t`. Protects: VPID, flags, oldest_unflush_lsa, next_wait_thrd, count_fix_and_avoid_dealloc. **Not held during I/O** — the flush path releases BCB mutex before `fileio_write()` and re-acquires after.

The BCB mutex monitoring system (`pgbuf_Monitor_locks = true` in debug builds) tracks which thread holds each BCB mutex and detects double-locks, cross-locks, and leaks at the call-site level.

### 8.4 LRU List Mutex (`lru_list->mutex`)

Per-list `pthread_mutex_t`. Protects all list pointer manipulations (top, bottom, bottom_1, bottom_2, victim_hint, zone counters). Must not be held while acquiring BCB mutex (ordering violation).

### 8.5 Invalid List Mutex (`buf_invalid_list.invalid_mutex`)

Global mutex protecting the free BCB list. Held only briefly during head-pop and head-push.

### 8.6 Atomic Latch (`bufptr->atomic_latch`)

64-bit `std::atomic<uint64_t>` encoding `{latch_mode, waiter_exists, fcnt}`. CAS loops implement:
- Lock-free read fix/unfix (READ pages with no waiters)
- Waiter flag setting (to signal sleeping threads)
- Fix count increment/decrement

Memory ordering: `memory_order_acq_rel` on success, `memory_order_acquire` on failure for all CAS operations.

### 8.7 Lock-Free Queues

`lockfree::circular_queue<T>` is used for:
- `direct_victims.waiter_threads_high/low_priority` — threads waiting for victim BCBs
- `flushed_bcbs` — BCBs post-flush, awaiting `pgbuf_assign_flushed_pages()` processing
- `private/shared_lrus_with_victims` — LRU indices with available victim candidates

These queues avoid mutex contention on the common victimization path.

### 8.8 Concurrent Fix/Unfix Handling

Multiple threads may hold a page with `PGBUF_LATCH_READ` simultaneously. Only one thread at a time may hold `PGBUF_LATCH_WRITE`. The transition rules enforced by `pgbuf_latch_bcb_upon_fix()`:
- READ on NO_LATCH: grant immediately.
- READ on READ: grant if no writers waiting.
- WRITE on NO_LATCH: grant immediately.
- WRITE on READ or WRITE: block (unless CONDITIONAL_LATCH).
- Upgrade (READ → WRITE, same thread): handled by `pgbuf_promote_read_latch()`.

### 8.9 Flush Thread Coordination

The page flush daemon (`pgbuf_Page_flush_daemon`) runs `pgbuf_flush_victim_candidates()` which sets `pgbuf_Pool.is_flushing_victims = true`. Checkpoint thread sets `pgbuf_Pool.is_checkpoint = true`. During checkpoint, flush boost is disabled to avoid interference.

Threads waiting with `PGBUF_LATCH_FLUSH` in the BCB wait queue are woken after flush completes in `pgbuf_wake_flush_waiters()`.

---

## 9. Memory Management

### 9.1 Buffer Memory Allocation

```c
// BCB table: flat array, no padding
pgbuf_Pool.BCB_table = malloc(num_buffers * sizeof(PGBUF_BCB));

// IO page table: each entry is BCB_ptr + alignment + FILEIO_PAGE
// Size per entry: offsetof(PGBUF_IOPAGE_BUFFER, iopage) + IO_PAGESIZE [+ guard in CUBRID_DEBUG]
pgbuf_Pool.iopage_table = malloc(num_buffers * PGBUF_IOPAGE_BUFFER_SIZE);
```

The `iopage_table` is allocated as a flat byte array. `PGBUF_FIND_IOPAGE_PTR(i)` uses `(char *)base + PGBUF_IOPAGE_BUFFER_SIZE * i` arithmetic.

### 9.2 Page Alignment

`PGBUF_IOPAGE_BUFFER` is designed so that `iopage` (the `FILEIO_PAGE`) is 8-byte aligned on 32-bit platforms via the `int dummy` padding member. On 64-bit Linux/Windows, the struct layout naturally achieves alignment without the dummy.

### 9.3 Holder Memory

Thread holder entries (`PGBUF_HOLDER`) are pre-allocated in `thrd_reserved_holder` (7 entries per thread). When a thread exhausts its pre-allocated entries, it takes entries from `free_holder_set` under `free_holder_set_mutex`. New `PGBUF_HOLDER_SET` arrays (10 entries each) are allocated from the OS on demand. Entries are never returned to the global pool — they stay per-thread.

### 9.4 Aout List Memory

All Aout list nodes are pre-allocated as `bufarray[max_count]`. No dynamic allocation during operation. `max_count = min(num_buffers * aout_ratio, 32768)`.

### 9.5 CUBRID Memory Rules

All heap allocations use `malloc()` directly (not `db_private_alloc()` because this is server-global, not transaction-scoped). All frees use `free_and_init(ptr)` which both frees and zeroes the pointer, preventing use-after-free.

---

## 10. Error Handling

### 10.1 Error Codes Used

| Error Code | Where | Condition |
|------------|-------|-----------|
| `ER_PB_BAD_PAGEID` | `pgbuf_fix()`, `pgbuf_is_valid_page()` | `pageid < 0` or page not allocated |
| `ER_PB_UNFIXED_PAGEPTR` | `pgbuf_unfix()`, `pgbuf_is_valid_page_ptr()` | `fcnt <= 0` on unfix attempt |
| `ER_PB_UNKNOWN_PAGEPTR` | `pgbuf_is_valid_page_ptr()` | Pointer not within any BCB range |
| `ER_PB_ALL_BUFFERS_DIRTY` | `pgbuf_allocate_bcb()` | Cannot find any victim (all dirty) |
| `ER_PAGE_LATCH_TIMEDOUT` | `pgbuf_timed_sleep()` | Latch wait exceeded timeout |
| `ER_PAGE_LATCH_ABORTED` | `pgbuf_fix_with_retry()` | Retry limit exceeded |
| `ER_PAGE_LATCH_PROMOTE_FAIL` | `pgbuf_promote_read_latch()` | Cannot promote (non-fatal) |
| `ER_LK_PAGE_TIMEOUT` | `pgbuf_latch_bcb_upon_fix()`, `pgbuf_timed_sleep()` | Finite wait timeout |
| `ER_LK_UNILATERALLY_ABORTED` | `pgbuf_timed_sleep()` | Infinite wait timeout — aborts transaction |
| `ER_INTERRUPTED` | `pgbuf_fix()`, `pgbuf_is_log_check_for_interrupts()` | Transaction interrupted |
| `ER_OUT_OF_VIRTUAL_MEMORY` | All `malloc()` call sites | Allocation failure |
| `ER_LOG_CHECKPOINT_SKIP_INVALID_PAGE` | `pgbuf_set_lsa()` | LSA set before checkpoint LSA (assertion) |

### 10.2 Invalid Page Detection

`pgbuf_is_valid_page()` calls `disk_is_page_sector_reserved()` to verify the page is within an allocated sector on its volume. `pgbuf_check_bcb_page_vpid()` validates that `bufptr->vpid` matches `iopage.prv.{pageid,volid}`. These checks run at the `PGBUF_DEBUG_PAGE_VALIDATION_FETCH` or `VALIDATION_ALL` level.

### 10.3 Corruption Detection (CUBRID_DEBUG)

Guard bytes (`pgbuf_Guard[8]`) are appended after each page's data area (via `PGBUF_FIND_BUFFER_GUARD`). Checked in `pgbuf_unfix()`. If overwritten, logs the overrun.

`pgbuf_scramble()` fills invalidated pages with `MEM_REGION_GUARD_MARK` bytes to detect use-after-free.

`pgbuf_is_consistent()` validates BCB field invariants (latch mode vs. fix count, etc.).

### 10.4 BCB State Invariants (enforced by asserts)

- `oldest_unflush_lsa` must be null when BCB is not dirty.
- `fcnt >= 0` always.
- A BCB in `PGBUF_VOID_ZONE` may not be in any LRU list.
- `PGBUF_BCB_FLUSHING_TO_DISK_FLAG` must not overlap with `PGBUF_BCB_VICTIM_DIRECT_FLAG`.

---

## 11. Integration Points

### 11.1 `file_io.c` — Disk I/O

- `fileio_read(thread_p, volid, pageid, iopage)` — called in `pgbuf_victimize_bcb()` to load page from disk
- `fileio_write(thread_p, vdes, iopage, pageid, ...)` — called in `pgbuf_bcb_flush_with_wal()` to flush to disk
- `fileio_get_volume_descriptor(volid)` — maps volid to OS file descriptor
- `fileio_get_volume_label(volid, PEEK)` — for error messages
- `fileio_init_lsa_of_page()`, `fileio_set_page_lsa()` — page header LSA management
- `fileio_flush_control_add_tokens()` — I/O rate limiting for the flush control daemon

### 11.2 `log_manager.c` / `log_append.cpp` — WAL Protocol

- `log_append_get_nxlsa()` — get current log tail LSA before flush
- `logpb_force_flush_pages(thread_p)` — force log flush synchronously (SA_MODE)
- `log_wakeup_log_flush_daemon()` — wake log flush daemon (SERVER_MODE)
- `log_Gl.chkpt_redo_lsa` / `log_Gl.chkpt_lsa_lock` — checkpoint LSA for validation in `pgbuf_set_lsa()`
- `log_append_undoredo_data2()` — TDE algorithm change logging

### 11.3 `double_write_buffer.hpp` — DWB

- `dwb_set_data_on_next_slot()` — stage page in DWB before writing
- DWB ensures partial writes cannot corrupt pages (crash-safe page write)

### 11.4 Log Manager Checkpoint Integration

`pgbuf_flush_checkpoint()` is called by the log manager's checkpoint thread. It is the primary mechanism for enforcing durability checkpoints. The function outputs `smallest_lsa` — the smallest `oldest_unflush_lsa` among all remaining dirty pages, which becomes the new checkpoint redo LSA.

### 11.5 Vacuum Integration

- `VACUUM_IS_THREAD_VACUUM_WORKER(thread_p)` — vacuum worker threads do NOT promote BCBs to hot zone (they use `PGBUF_SHOULD_IGNORE_UNFIX` and `PGBUF_VACUUM_SHOULD_IGNORE_UNFIX` macros)
- `pgbuf_notify_vacuum_follows()` — marks a BCB with `PGBUF_BCB_TO_VACUUM_FLAG` to hint that vacuum will follow shortly

### 11.6 Callers of `pgbuf_fix()`

All page-level modules call `pgbuf_fix()`:

| Module | Files |
|--------|-------|
| Heap manager | `src/storage/heap_file.c` |
| B-tree | `src/storage/btree.c`, `btree_load.c` |
| Slotted page | `src/storage/slotted_page.c` |
| File manager | `src/storage/file_manager.c` |
| Disk manager | `src/storage/disk_manager.c` |
| Overflow | `src/storage/overflow_file.c` |
| External sort | `src/storage/external_sort.c` |
| Extendible hash | `src/storage/extendible_hash.c` |
| Log manager | `src/transaction/log_manager.c`, `log_recovery.c` |
| Lock manager | `src/transaction/lock_manager.c` |
| MVCC | `src/transaction/mvcc.c` |
| Vacuum | `src/query/vacuum.c` |

---

## 12. Complexity & Metrics

### 12.1 Function Counts

| Category | Count |
|----------|-------|
| Public API functions | ~55 |
| Static helper functions | ~80 |
| Static inline functions | ~45 |
| Daemon/C++ class functions | 8 |
| **Total** | **~190** |

### 12.2 Largest Functions (estimated by logical complexity)

| Function | Approx. Lines | Complexity Notes |
|----------|---------------|------------------|
| `pgbuf_fix_debug/release` | ~430 | Main fix with 7 major paths; perf tracking, fetch mode variants, VPID validation |
| `pgbuf_flush_victim_candidates` | ~350 | Flush loop with rate control, boost calc, sequential sort, neighbor flush |
| `pgbuf_unlatch_bcb_upon_unfix` | ~220 | CAS loop, 6 zone cases, private/shared handling, wakeup |
| `pgbuf_latch_bcb_upon_fix` | ~200 | CAS loop, 5 latch grant cases, holder management |
| `pgbuf_flush_seq_list` | ~200 | Rate-controlled flush loop with burst/non-burst, time limits |
| `pgbuf_flush_checkpoint` | ~180 | BCB scan, sorted list build, seq flusher invocation |
| `pgbuf_unfix_debug` | ~170 | CUBRID_DEBUG checks, consistency check, scramble path |
| `pgbuf_promote_read_latch` | ~150 | CAS promotion with block/unblock and holder rebuild |
| `pgbuf_ordered_fix` | ~400+ | Watcher management, group detection, reorder logic |
| `pgbuf_adjust_quotas` | ~200 | Activity computation, quota adjustment, LRU destruction |
| `pgbuf_get_victim` | ~130 | 4-phase search, lfcq interaction, direct victim fallback |

### 12.3 Complexity Hotspots

1. **`pgbuf_fix()`** — The hottest code path. Called for every page access from every module. The lockfree_fix_ro fast path is critical for read scalability.

2. **Atomic latch CAS loops** — The `set_latch*` / `add_fcnt` / `get_impl` inline functions each execute a spin-loop CAS. Contention on popular pages causes CPU spinning.

3. **LRU zone adjustment** — Every fix on a zone-2 or zone-3 page triggers `pgbuf_lru_boost_bcb()` which acquires the LRU list mutex. This is a scalability bottleneck under heavy concurrent access to warm pages.

4. **Victim search** — Under memory pressure, `pgbuf_get_victim()` may loop through multiple LRU lists. The lock-free circular queue consumption-based approach reduces mutex contention but cannot eliminate the BCB-level mutex on victimized pages.

5. **`pgbuf_ordered_fix()`** — Complex deadlock-avoidance logic requiring conditional unfix and re-fix of multiple heap pages. The worst case is O(n) unfix/fix operations where n is the number of heap pages held by the thread.

---

## 13. Notable Patterns & Idioms

### 13.1 Debug/Release Function Pairs

Every public function that needs caller tracking follows the pattern:

```c
// In page_buffer.h:
#if !defined(NDEBUG)
#define pgbuf_fix(...) pgbuf_fix_debug(..., ARG_FILE_LINE_FUNC)
extern PAGE_PTR pgbuf_fix_debug(THREAD_ENTRY *, const VPID *, ..., const char *, int, const char *);
#else
#define pgbuf_fix(...) pgbuf_fix_release(...)
extern PAGE_PTR pgbuf_fix_release(THREAD_ENTRY *, const VPID *, ...);
#endif
```

`ARG_FILE_LINE_FUNC` expands to `__FILE__, __LINE__, __func__`. This pattern is used for `pgbuf_fix`, `pgbuf_unfix`, `pgbuf_set_dirty`, `pgbuf_set_lsa`, `pgbuf_ordered_fix/unfix`, `pgbuf_promote_read_latch`, `pgbuf_invalidate`, `pgbuf_invalidate_all`, `pgbuf_replace_watcher`, `pgbuf_attach_watcher`.

### 13.2 STATIC_INLINE with ALWAYS_INLINE

```c
STATIC_INLINE bool pgbuf_is_bcb_victimizable(PGBUF_BCB *bcb, bool has_mutex_lock)
  __attribute__ ((ALWAYS_INLINE));
```

Many functions are both declared `static inline` and marked with `__attribute__((ALWAYS_INLINE))` to ensure the compiler never emits a non-inlined copy. This is critical for the hot-path functions like `pgbuf_bcb_is_dirty()`, `pgbuf_bcb_get_zone()`, `pgbuf_lru_list_from_bcb()`.

### 13.3 CAS Loop Pattern

All atomic latch mutations use the same pattern:

```c
PGBUF_ATOMIC_LATCH_IMPL impl, new_impl;
do {
  impl.raw = latch->load(std::memory_order_acquire);
  new_impl = impl;
  // modify new_impl
} while (!latch->compare_exchange_weak(impl.raw, new_impl.raw,
                                        std::memory_order_acq_rel,
                                        std::memory_order_acquire));
```

`compare_exchange_weak` is used in the spin loops (more efficient on platforms with LL/SC), while `compare_exchange_strong` is used when spurious failure must be avoided (e.g., in `pgbuf_promote_read_latch`).

### 13.4 `free_and_init()` Pattern

All heap frees use `free_and_init(ptr)` which sets `ptr = NULL` after freeing, preventing double-free. This is a CUBRID-wide anti-pattern guard. Bare `free()` is **never** used.

### 13.5 `PGBUF_BCB_LOCK`/`PGBUF_BCB_UNLOCK` Macros

```c
// SERVER_MODE with monitoring:
#define PGBUF_BCB_LOCK(bcb) \
  (pgbuf_Monitor_locks ? pgbuf_bcbmon_lock(bcb, __LINE__) \
                       : (void) pthread_mutex_lock(&(bcb)->mutex))
// SA_MODE: no-ops
#define PGBUF_BCB_LOCK(bcb)
```

This pattern unifies all BCB locking and makes it trivially possible to add monitoring or skip locking depending on build mode.

### 13.6 Zone Encoding in a Single Integer

The BCB's current zone and LRU list index are encoded in the same `volatile int flags` field:

```
flags = PGBUF_MAKE_ZONE(lru_index, zone_constant)
```

where `zone_constant` occupies the upper bits (above bit 17) and `lru_index` occupies the lower 16 bits. `PGBUF_GET_ZONE(flags)` and `PGBUF_GET_LRU_INDEX(flags)` extract the parts. This avoids a second word of memory and a separate atomic operation.

### 13.7 Scope Exit Pattern

```c
#include "scope_exit.hpp"
auto unlock_BCB = cubmem::scope_exit ([bufptr] { PGBUF_BCB_UNLOCK (bufptr); });
// ... later: unlock_BCB.release(); // cancel the scope exit
```

Used in `pgbuf_latch_bcb_upon_fix()` to ensure the BCB mutex is unlocked on all exit paths including early returns, while retaining the ability to cancel the unlock when the function intentionally passes locked ownership to the caller.

### 13.8 Watcher Doubly-Linked List (Ordered Fix)

`PGBUF_WATCHER` structs are arranged in a doubly-linked list within a `PGBUF_HOLDER`. Each watcher has `next`/`prev` pointers. The `PGBUF_INIT_WATCHER` and `PGBUF_CLEAR_WATCHER` macros manage initialization with optional debug magic number checking. This supports the heap file's need to fix multiple pages in a defined order.

### 13.9 `pgbuf_ordered_null_hfid` Sentinel

`PGBUF_ORDERED_NULL_HFID` is a global sentinel HFID pointer (`pgbuf_ordered_null_hfid = NULL`) used in ordered fix APIs to indicate "no heap file grouping." The macro `PGBUF_WATCHER_SET_GROUP(w, hfid)` checks for this and for null/invalid HFIDs before setting the group_id, preventing spurious deadlock-avoidance reordering on non-heap pages.

### 13.10 `PGBUF_ABORT_RELEASE()` Macro

```c
#if defined(NDEBUG)
static int pgbuf_Abort_release_line = 0;
#define PGBUF_ABORT_RELEASE() do { pgbuf_Abort_release_line = __LINE__; abort(); } while (false)
#else
#define PGBUF_ABORT_RELEASE() assert(false)
#endif
```

In release builds, saves the line number to a global before aborting. This survives in a core dump and allows post-mortem identification of the abort site even when compiler optimization has obscured the call stack.

### 13.11 SystemTap Probe Points

```c
#if defined(ENABLE_SYSTEMTAP)
  CUBRID_PGBUF_HIT();   // fired on buffer pool hit
  CUBRID_PGBUF_MISS();  // fired on buffer pool miss
#endif
```

These hooks allow production tracing of buffer pool behavior without recompilation, using DTrace or SystemTap.

### 13.12 `PGBUF_IS_AUXILIARY_VOLUME(volid)` Guard

```c
#define PGBUF_IS_AUXILIARY_VOLUME(volid) ((volid) < LOG_DBFIRST_VOLID ? true : false)
```

Used in LSA and dirty-checking logic to exclude volumes like copydb/backupdb targets from normal WAL enforcement.

---

## Appendix A: System Parameter References

| Parameter ID | Type | Default | Usage |
|--------------|------|---------|-------|
| `PRM_ID_PB_NBUFFERS` | int | computed | Pool size |
| `PRM_ID_PB_LRU_HOT_RATIO` | float | 0.4 | Zone 1 fraction |
| `PRM_ID_PB_LRU_BUFFER_RATIO` | float | 0.4 | Zone 2 fraction |
| `PRM_ID_PB_NUM_LRU_CHAINS` | int | MAX_NTRANS | Shared LRU count |
| `PRM_ID_PB_AOUT_RATIO` | float | 0.3 | Aout list size / pool |
| `PRM_ID_PB_BUFFER_FLUSH_RATIO` | float | 0.01 | Base flush ratio |
| `PRM_ID_PB_NEIGHBOR_FLUSH_PAGES` | int | 7 | Neighbor flush window |
| `PRM_ID_PB_NEIGHBOR_FLUSH_NONDIRTY` | bool | false | Flush non-dirty neighbors |
| `PRM_ID_PB_SEQUENTIAL_VICTIM_FLUSH` | bool | true | Sort before flush |
| `PRM_ID_PB_MONITOR_LOCKS` | bool | false (release) / true (debug) | BCB mutex monitoring |
| `PRM_ID_PAGE_LATCH_TIMEOUT` | int | 300 (sec) | Latch wait timeout |
| `PRM_ID_PAGE_BG_FLUSH_INTERVAL_MSECS` | int | 100 | Flush daemon period |
| `PRM_ID_LOG_PGBUF_VICTIM_FLUSH` | bool | false | Verbose flush logging |
| `PRM_ID_LOG_CHKPT_DETAILED` | bool | false | Detailed checkpoint logging |

---

## Appendix B: Key Public Macros (from `page_buffer.h`)

```c
pgbuf_unfix_and_init(thread_p, pgptr)           // unfix then set pgptr = NULL
pgbuf_unfix_and_init_after_check(thread_p, pgptr) // null-safe version
pgbuf_ordered_unfix_and_init(thread_p, page, pg_watcher)
pgbuf_set_dirty_and_free(thread_p, pgptr)       // set dirty + unfix + NULL
PGBUF_INIT_WATCHER(w, rank, hfid)               // initialize watcher struct
PGBUF_CLEAR_WATCHER(w)                          // clear watcher (set NULL ptrs)
PGBUF_IS_CLEAN_WATCHER(w)                       // check watcher is unused
PGBUF_IS_ORDERED_PAGETYPE(ptype)                // true for PAGE_HEAP or PAGE_OVERFLOW
PGBUF_IS_PAGE_CHANGED(pgptr, ref_lsa)           // check if LSA changed since ref_lsa
PGBUF_PAGE_STATE_MSG(name)                      // log format string: VPID + LSA
PGBUF_PAGE_MODIFY_MSG(name)                     // log format: VPID + prev/curr LSA
PGBUF_WATCHER_SET_GROUP(w, hfid)                // set watcher group from HFID
PGBUF_WATCHER_COPY_GROUP(w_dst, w_src)          // copy group between watchers
```
