# CUBRID Concurrency Architecture

A comprehensive analysis of CUBRID's transaction and concurrency internals: locking, latching, MVCC, WAL/recovery, buffer pool, and thread management.

**Audience:** CUBRID core developers and contributors
**Codebase:** CUBRID 11.5.x (`develop` branch, 2026-03-29)
**Scope:** ~120K lines across `src/transaction/`, `src/storage/`, `src/thread/`, `src/base/`, `src/query/`

---

## Table of Contents

1. [Introduction & ACID Mapping](#1-introduction--acid-mapping)
2. [Lock Manager](#2-lock-manager)
3. [Page Buffer & Latching](#3-page-buffer--latching)
4. [MVCC & Vacuum](#4-mvcc--vacuum)
5. [WAL, Recovery & Commit](#5-wal-recovery--commit)
6. [Buffer Pool Management](#6-buffer-pool-management)
7. [Concurrency Infrastructure](#7-concurrency-infrastructure)
8. [Cross-Cutting Flows](#8-cross-cutting-flows)
9. [Summary & Key Design Decisions](#9-summary--key-design-decisions)

---

## 1. Introduction & ACID Mapping

CUBRID employs a hybrid concurrency control strategy that combines pessimistic object-level locking with optimistic MVCC snapshot isolation. This dual approach allows readers to proceed without blocking writers in most cases, while writers coordinate through a granular lock hierarchy. Underneath, a Write-Ahead Logging (WAL) protocol ensures crash recovery, and a multi-zone buffer pool manages page-level concurrency through latches.

The concurrency subsystems are layered. At the lowest level, lock-free data structures and atomic operations provide contention-free access to shared internal tables. Above that, critical sections (reader-writer locks) protect coarse-grained shared state like the transaction table and the wait-for-graph. The buffer pool uses page-level latches (short-duration, fine-grained) to protect pages during access. At the logical level, the lock manager provides transaction-duration object locks, while MVCC snapshots enable non-blocking reads. The WAL ties everything together for durability and crash recovery.

The following diagram shows how the six major concurrency subsystems relate:

```
                    +-----------------------+
                    |   Client Transaction  |
                    +-----------+-----------+
                                |
               +----------------+----------------+
               |                                 |
      +--------v--------+              +--------v--------+
      |  Lock Manager   |              |      MVCC       |
      | (object locks)  |<------------>| (snapshots,     |
      | non2pl, WFG     |  isolation   |  visibility)    |
      +--------+--------+              +--------+--------+
               |                                 |
               |  page access                    |  version chain
               |                                 |  (prev_version_lsa)
      +--------v--------+              +--------v--------+
      |   Buffer Pool   |              |       WAL       |
      | (latching, LRU, |<------------>| (log records,   |
      |  dirty tracking)|  WAL proto   |  recovery)      |
      +--------+--------+              +--------+--------+
               |                                 |
               +----------------+----------------+
                                |
                    +-----------v-----------+
                    | Concurrency Infra     |
                    | (THREAD_ENTRY, csect, |
                    |  lock-free structs)   |
                    +-----------------------+
```

CUBRID supports four isolation levels: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, and SERIALIZABLE. In practice, most deployments use READ COMMITTED or REPEATABLE READ, where MVCC snapshots handle read isolation and the lock manager handles write isolation. The transaction lifecycle is managed through `log_tran_table.c`, which maps transaction indices to their MVCC state, lock holdings, and log chain pointers. Each active transaction has an entry in the global transaction table, protected by `CSECT_TRAN_TABLE`.

### ACID-to-Implementation Mapping

| Property | Primary Subsystem | Mechanism | Key Files |
|----------|------------------|-----------|-----------|
| **Atomicity** | WAL | Undo logging: every modification is logged with undo data before execution. On abort, undo records are replayed in reverse order via compensation log records (CLR). System operations use nested top-level transactions (`LOG_SYSOP_END`) to make sub-operations independently atomic. | `log_manager.c`, `log_recovery.c` |
| **Consistency** | Lock Manager | Two-phase locking (with non-2PL extensions) prevents conflicting concurrent modifications. Schema locks (`SCH_S_LOCK`, `SCH_M_LOCK`) protect DDL operations against concurrent DML. Unique constraint violations are detected within the lock-protected critical region of index insertion. | `lock_manager.c`, `lock_table.c` |
| **Isolation** | MVCC + Lock Manager | MVCC snapshots provide read-committed and repeatable-read isolation without read locks. The snapshot captures active transaction IDs at query start, then `mvcc_satisfies_snapshot()` determines record visibility. Object-level locks prevent write-write conflicts. The `non2pl` list enables early lock release while maintaining logical isolation through MVCC visibility. | `mvcc.c`, `lock_manager.c` |
| **Durability** | WAL + Buffer Pool | WAL protocol: log records are forced to disk before commit returns. The commit path calls `logpb_flush_pages()` which either flushes synchronously or batches via group commit. Checkpoint records track the oldest dirty page LSN (`redo_lsa`), enabling efficient crash recovery by bounding the redo start point. The buffer pool enforces the WAL invariant: no dirty page is flushed to disk before its log records are flushed. | `log_page_buffer.c`, `page_buffer.c` |

---

## 2. Lock Manager

The lock manager (`src/transaction/lock_manager.c`, 9,939 lines) implements object-level locking with a 12-mode lock hierarchy, deadlock detection via wait-for-graph analysis, and lock escalation for high-contention scenarios. It operates at the logical level, protecting OID-identified objects (instances and classes) for the duration of transactions.

### 2.1 Lock Resource Model

Every lockable object is represented by an `LK_RES` structure (`lock_manager.h:174-192`):

```
LK_RES (Lock Resource)
+---------------------------+
| key: LK_RES_KEY           |  -- {type, oid, class_oid}
| total_holders_mode: LOCK  |  -- aggregate of all holders
| total_waiters_mode: LOCK  |  -- aggregate of all waiters
| holder: LK_ENTRY*         |  -- linked list of lock holders
| waiter: LK_ENTRY*         |  -- linked list of lock waiters
| non2pl: LK_ENTRY*         |  -- non-2PL deferred entries
| res_mutex: pthread_mutex_t|  -- per-resource synchronization
| del_id: UINT64            |  -- epoch ID for lock-free reclamation
+---------------------------+
```

The `key` is a composite of `{type, oid, class_oid}` where `type` is one of `LOCK_RESOURCE_INSTANCE`, `LOCK_RESOURCE_CLASS`, or `LOCK_RESOURCE_ROOT_CLASS`. This three-part key allows the lock manager to distinguish between locking an instance, locking the class itself, and locking the class hierarchy root. The `total_holders_mode` is the aggregate lock mode of all entries on the `holder` list, computed as the supremum of individual holder modes using the conversion matrix. Similarly, `total_waiters_mode` is the aggregate of all waiter requests.

Each transaction's lock on a resource is tracked by an `LK_ENTRY` (`lock_manager.h:78-101`):

```
LK_ENTRY (Lock Entry)
+---------------------------+
| res_head: LK_RES*         |  -- back-pointer to resource
| thrd_entry: THREAD_ENTRY* |  -- owning thread
| tran_index: int           |  -- transaction table index
| granted_mode: LOCK        |  -- currently held mode (NULL if waiting)
| blocked_mode: LOCK        |  -- requested mode (if waiting)
| count: int                |  -- re-entry count
| next: LK_ENTRY*           |  -- next in holder/waiter list
| tran_next: LK_ENTRY*      |  -- next in transaction's lock chain
| tran_prev: LK_ENTRY*      |  -- prev in transaction's lock chain
| class_entry: LK_ENTRY*    |  -- parent class lock (for escalation)
| ngranules: int            |  -- count of finer-grained locks
| instant_lock_count: int   |  -- short-lived lock count
+---------------------------+
```

The `tran_next`/`tran_prev` doubly-linked list is anchored in the per-transaction `LK_TRAN_LOCK` structure within the global lock data. This chain enables efficient iteration over all locks held by a transaction, which is critical for bulk unlock on commit or abort. The `class_entry` back-pointer connects instance locks to their parent class lock, enabling the escalation logic to count granules and decide when to promote.

The global lock table is stored in `LK_GLOBAL_DATA` (`lock_manager.c:359-399`). Critically, the resource hash table is a **lock-free hashmap**:

```cpp
using lk_hashmap_type = cubthread::lockfree_hashmap<LK_RES_KEY, LK_RES>;
lk_hashmap_type m_obj_hash_table;  // lock_manager.c:355
```

This means lock resource lookup is O(1) average without any global lock contention. The lock-free hashmap uses hazard-pointer-based reclamation (see Section 7) to safely deallocate unused `LK_RES` entries. The choice between old-style (`lf_hash_table_cpp`) and new-style (`lockfree::hashmap`) implementations is controlled by the `PRM_ID_ENABLE_NEW_LFHASH` system parameter. The global data also includes `LK_TRAN_LOCK *tran_lock_table`, which is an array indexed by `tran_index` providing fast access to each transaction's lock metadata, and `LF_FREELIST obj_free_entry_list` for lock-free reuse of `LK_ENTRY` structures.

### 2.2 Lock Modes and Compatibility

CUBRID supports 12 lock modes defined in `lock_table.c`. The modes form a partial order that supports both instance-level and class-level (intention) locking:

| Mode | Symbol | Purpose | Typical Usage |
|------|--------|---------|---------------|
| NULL_LOCK | - | No lock | Initial state |
| SCH_S_LOCK | Schema-S | Schema read protection | SELECT (DDL protection) |
| IS_LOCK | Intent-S | Intent to read instances | Class lock for SELECT |
| S_LOCK | Shared | Read lock | Explicit table lock |
| IX_LOCK | Intent-X | Intent to modify instances | Class lock for UPDATE/INSERT |
| BU_LOCK | Bulk-Update | Bulk update operations | Bulk loading |
| SIX_LOCK | S+IX | Shared + intent to modify | Table scan with instance updates |
| U_LOCK | Update | Read with upgrade intent | SELECT ... FOR UPDATE |
| X_LOCK | Exclusive | Write lock | UPDATE, DELETE on instances |
| SCH_M_LOCK | Schema-M | Schema exclusive | DDL (ALTER TABLE, DROP) |

The compatibility matrix (`lock_Comp[requested][held]`, `lock_table.c:67`) is a 12x12 boolean table. Key compatibility rules: `S` is compatible with `S` and `IS` but conflicts with `IX`, `X`, and `SCH_M`. `IX` is compatible with `IX` and `IS` but conflicts with `S`, `X`, and `SCH_M`. `X` conflicts with everything except `NULL_LOCK`. `SCH_M` conflicts with all modes including `SCH_S`. The `BU_LOCK` is special: it is compatible only with `BU_LOCK` and `NULL_LOCK`, providing full exclusion during bulk loading.

The conversion matrix (`lock_Conv[requested][current]`, `lock_table.c`) defines lock strengthening when a transaction re-requests a lock. For example: `S` + `IX` converts to `SIX`; `IS` + `IX` converts to `IX`; `S` + `X` converts to `X`. This handles the common pattern where a transaction holds a shared lock and later needs to write.

### 2.3 Lock Acquisition and Release

The primary entry point is `lock_object()` (`lock_manager.h:202`):

```c
int lock_object(THREAD_ENTRY *thread_p, const OID *oid,
                const OID *class_oid, LOCK lock, int cond_flag);
```

The `cond_flag` parameter controls blocking behavior: `LK_UNCOND_LOCK` blocks the caller until the lock is granted, while `LK_COND_LOCK` returns immediately with `LK_NOTGRANTED` if the lock cannot be granted. The timeout variant `lock_object_wait_msecs()` allows a bounded wait.

The detailed acquisition flow:

1. **Hash lookup**: Compute bucket index from OID via `LK_OBJ_LOCK_HASH(oid, htsize)`. Call `m_obj_hash_table.find_or_insert()` to find the existing `LK_RES` or create a new one atomically. This step is lock-free.

2. **Acquire res_mutex**: The per-resource `pthread_mutex_t` serializes all subsequent operations on this resource's holder, waiter, and non2pl lists. This is fine-grained: different resources can be modified concurrently.

3. **Check for existing lock**: Scan the `holder` list for an entry from the same `tran_index`. If found, check if the existing `granted_mode` already covers the request (using the lock mode partial order). If so, increment `count` and return `LK_GRANTED` without modifying the lists.

4. **Compatibility check**: Compare the `requested` mode against `total_holders_mode` using `lock_Comp[requested][total_holders_mode]`. If compatible, also check `total_waiters_mode` (to prevent starvation of waiting transactions).

5. **If compatible**: Add `LK_ENTRY` to the `holder` list. Update `total_holders_mode` using `lock_Conv`. Release `res_mutex`. Return `LK_GRANTED`.

6. **If incompatible**: Add `LK_ENTRY` to the `waiter` list. Set `blocked_mode` to the requested mode, `granted_mode` to `NULL_LOCK`. Update `total_waiters_mode`. Set the thread's `lockwait` pointer to the entry and `lockwait_state` to `LOCK_SUSPENDED`. Release `res_mutex`. **Suspend the thread** by calling `pthread_cond_wait()` on `thread_p->wakeup_cond`.

7. **On wakeup**: Check `lockwait_state`. Possible outcomes:
   - `LOCK_RESUMED`: Lock was granted by the releasing transaction
   - `LOCK_RESUMED_TIMEOUT`: Wait timed out, return `LK_NOTGRANTED_DUE_TIMEOUT`
   - `LOCK_RESUMED_DEADLOCK_TIMEOUT`: Selected as deadlock victim, return `LK_NOTGRANTED_DUE_ABORTED`

Lock release (`lock_unlock_object`, `lock_manager.h:209`) reverses this: it removes the entry from the `holder` list, recalculates `total_holders_mode`, and then traverses the `waiter` list to find entries whose `blocked_mode` is now compatible with the new `total_holders_mode`. Compatible waiters are moved from the `waiter` list to the `holder` list, their `granted_mode` is set, and their threads are signaled via `pthread_cond_signal()` on their `wakeup_cond`. When a transaction commits or aborts, `lock_unlock_all()` iterates the transaction's lock chain (`tran_next`) and releases all held locks in bulk.

### 2.4 The Non-2PL List

The `non2pl` field in `LK_RES` is architecturally significant. Standard two-phase locking (2PL) requires that all locks are held until transaction commit — the "shrinking phase" only begins at commit. CUBRID relaxes this for certain operations: a transaction can release a lock early but defer the visibility of the unlock until commit. The released entry moves to the `non2pl` list rather than being freed immediately.

Functions managing this list include `lock_add_non2pl_lock`, `lock_remove_non2pl`, `lock_update_non2pl_list`, `lock_initialize_entry_as_non2pl`, `lock_insert_into_tran_non2pl_list`, and `lock_delete_from_tran_non2pl_list` (all in `lock_manager.c`). The `non2pl` mechanism appears 159 times in the lock manager, reflecting its pervasive use. Entries on the `non2pl` list still occupy `LK_ENTRY` structures and are linked in the resource's chain, but they do not affect the `total_holders_mode` computation, meaning other transactions can acquire conflicting locks. The entry is fully cleaned up only when the owning transaction commits.

This deviation from strict 2PL is MVCC-friendly: physical locks can be released early to reduce blocking, while logical isolation is maintained through MVCC snapshot visibility. A concurrent transaction acquiring a conflicting lock after the early release will still see the correct data version through its MVCC snapshot.

### 2.5 Lock Escalation

When a transaction acquires more than `KEY_LOCK_ESCALATION_THRESHOLD` (10) instance locks under the same class, the lock manager escalates to a class-level lock (`lock_manager.c:173`). The `KEY_LOCK_ESCALATION` enum (`lock_manager.h:68-73`) tracks the escalation state:

- `NO_KEY_LOCK_ESCALATION`: Under threshold, normal instance locking
- `NEED_KEY_LOCK_ESCALATION`: Threshold reached, escalation pending on next lock request
- `KEY_LOCK_ESCALATED`: Class lock acquired, individual instance locks released

The `LK_ENTRY.class_entry` field back-points to the parent class lock entry, and `ngranules` counts the number of finer-grained locks held under it. When `ngranules` exceeds the threshold, the lock manager acquires a class-level `S_LOCK` or `X_LOCK` (depending on the instance lock modes being held) and releases the individual instance locks. This reduces memory consumption (fewer `LK_ENTRY` allocations) and lock manager overhead, at the cost of potentially broader blocking scope.

### 2.6 Deadlock Detection

Deadlocks are detected by a background daemon that constructs a wait-for-graph (WFG) and searches for cycles. The WFG subsystem lives in `src/transaction/wait_for_graph.c` (2,434 lines).

**WFG Data Structures**: Each transaction is a node (`WFG_NODE`, lines 66-82) with a `status` field tracking DFS traversal state (`WFG_NOT_VISITED`, `WFG_ON_STACK`, `WFG_OFF_STACK`, `WFG_RE_ON_STACK`, `WFG_ON_TG_CYCLE`). Edges (`WFG_EDGE`, lines 56-63) are directed: `waiter_tran_index` → `holder_tran_index`, meaning "waiter waits for holder." Each node has linked lists of outgoing holder edges and incoming waiter edges.

**Graph Construction**: When `lock_object()` suspends a thread (step 6 above), the lock manager records the wait relationship. The WFG daemon (triggered periodically) calls `wfg_insert_out_edges()` to add edges from each waiting transaction to all transactions holding conflicting locks on the same resource.

**Cycle Detection Algorithm**: The WFG uses a non-recursive depth-first search with an explicit stack (`WFG_STACK`, lines 85-90). Each stack frame contains a `wait_tran_index` and a `current_holder_edge_p` (the edge being traversed). The algorithm pushes a waiting transaction, then iterates through its holder edges. If an edge points to a node already `WFG_ON_STACK`, a cycle is detected. The stack is bounded and cycles are collected up to a maximum of 100 cycles with 10 per group (lines 42-43).

**Victim Selection**: Among transactions in a detected cycle, the youngest transaction (based on `tran_index`) is selected as the victim. It is aborted to break the cycle. The victim's thread receives `LOCK_RESUMED_DEADLOCK_TIMEOUT` as the wakeup reason from `lockwait_state`. The WFG daemon is protected by `CSECT_WFG` critical section and runs lazily (not maintaining the graph in real-time), which reduces overhead at the cost of slightly delayed deadlock detection.

### 2.7 Interactions

**Lock Manager x MVCC**: Object locks prevent write-write conflicts; MVCC snapshots handle read-write concurrency. A `SELECT` acquires no instance locks (only class-level `SCH_S_LOCK` for DDL protection). An `UPDATE` acquires `IX_LOCK` on the class and `X_LOCK` on the instance. The `non2pl` list bridges these subsystems: locks released early still appear in MVCC's visibility checks until commit.

**Lock Manager x Lock-Free Infrastructure**: The lock resource hash table (`m_obj_hash_table`) is a `cubthread::lockfree_hashmap`. Each thread uses its `THREAD_ENTRY->tran_entries[THREAD_TS_OBJ_LOCK_RES]` descriptor for hazard-pointer-based safe reclamation of retired `LK_RES` entries. Lock entries use `THREAD_TS_OBJ_LOCK_ENT` for the same purpose. This allows concurrent lock lookups and entry allocation without global table locks (see Section 7 for the lock-free hashmap internals).

**Lock Manager x Buffer Pool**: The lock manager does not directly interact with the buffer pool during lock acquisition. However, when a transaction modifies a page (after acquiring an `X_LOCK`), the buffer pool's page latch (`PGBUF_LATCH_WRITE`) provides physical-level mutual exclusion on the page content while the object lock provides logical-level transaction-duration isolation.

---

## 3. Page Buffer & Latching

Low-level concurrency within the storage engine is managed through latches (lightweight locks on buffer pool pages) and critical sections (reader-writer locks on shared data structures). Unlike object locks, latches are held only for the duration of a page access operation, not for the entire transaction. They protect physical data structures (pages, internal tables) rather than logical objects.

### 3.1 Critical Sections

CUBRID uses `SYNC_CRITICAL_SECTION` (`src/thread/critical_section.h:110-124`) as its primary reader-writer synchronization primitive for shared internal data structures:

```
SYNC_CRITICAL_SECTION
+------------------------------+
| name: const char*            |  -- human-readable name for debugging
| cs_index: int                |  -- static index (e.g., CSECT_WFG)
| lock: pthread_mutex_t        |  -- monitor lock (protects state transitions)
| rwlock: int                  |  -- >0: #readers, <0: writer, 0: free
| waiting_readers: unsigned    |  -- count of suspended readers
| waiting_writers: unsigned    |  -- count of suspended writers
| readers_ok: pthread_cond_t   |  -- condition variable for reader wakeup
| waiting_writers_queue: FIFO  |  -- writer thread queue (THREAD_ENTRY chain)
| waiting_promoters_queue: FIFO|  -- reader-to-writer upgrade queue
| owner: thread_id_t           |  -- thread ID of current writer
| tran_index: int              |  -- transaction of current writer
| stats: SYNC_STATS*           |  -- performance statistics
+------------------------------+
```

The system defines 18 static critical sections (`critical_section.c:76-95`) protecting different shared resources:

| Index | Name | Protects |
|-------|------|----------|
| 0 | `CSECT_WFG` | Wait-for-graph construction and cycle detection |
| 1 | `CSECT_LOG` | Log manager state and log page buffer |
| 2 | `CSECT_LOCATOR_SR_CLASSNAME_TABLE` | Class name lookup table |
| 3 | `CSECT_QPROC_QUERY_TABLE` | Query processing plan cache |
| 4 | `CSECT_QPROC_LIST_CACHE` | Query result list cache |
| 8 | `CSECT_TRAN_TABLE` | Transaction table modifications |

**Reader path** (`csect_enter_as_reader`): Acquire the monitor `lock`. If `rwlock >= 0` and there are no waiting writers (preventing reader starvation of writers), atomically increment `rwlock` and release the monitor. Otherwise, increment `waiting_readers` and block on the `readers_ok` condition variable. On wakeup, decrement `waiting_readers`, increment `rwlock`, and proceed.

**Writer path** (`csect_enter`): Acquire the monitor `lock`. If `rwlock == 0` and no writers are queued, set `rwlock = -1`, record the `owner` thread ID, release the monitor. Otherwise, increment `waiting_writers` and enqueue the `THREAD_ENTRY` on `waiting_writers_queue` (FIFO ordering). Block on the thread's own condition variable. FIFO ordering prevents writer starvation and ensures fairness.

**Promotion** (`csect_promote`): A reader can upgrade to writer status. The thread is added to `waiting_promoters_queue`. When all other readers exit, the promoter is granted exclusive access. This avoids a full release-reacquire cycle which could lose the reader's position.

**Exit**: On reader exit, decrement `rwlock`. If `rwlock` reaches 0 and there are waiting writers or promoters, signal the first one. On writer exit, set `rwlock = 0`, then signal either the first waiting promoter, the first waiting writer, or broadcast to all waiting readers (priority: promoters > writers > readers).

The critical section tracker (`src/thread/critical_section_tracker.hpp:32-68`, namespace `cubsync`) is a debug facility that monitors acquisition order across different critical sections to detect potential deadlock-prone orderings. Each `THREAD_ENTRY` has a `m_csect_tracker` that records which critical sections are held and in what order, detecting violations at runtime during development builds.

### 3.2 Buffer Pool Latch Modes

Page-level latches protect buffer pool pages during access. The latch modes (`page_buffer.h:190-197`):

| Mode | Value | Semantics | Holding Duration |
|------|-------|-----------|-----------------|
| `PGBUF_NO_LATCH` | 0 | Page not accessible | N/A |
| `PGBUF_LATCH_READ` | 1 | Shared: multiple concurrent readers | Microseconds to milliseconds |
| `PGBUF_LATCH_WRITE` | 2 | Exclusive: single writer | Microseconds to milliseconds |
| `PGBUF_LATCH_FLUSH` | 3 | Flush-exclusive: during page writeback | Duration of I/O |

Latch state is stored atomically in the BCB (Buffer Control Block) via `PGBUF_ATOMIC_LATCH` (`page_buffer.c:491-500`) — a 64-bit value packed with `latch_mode` (2 bits), `waiter_exists` (1 bit), and `fcnt` (fix count, 29 bits). In the uncontended case, latch acquisition is a single CAS (compare-and-swap) operation on this 64-bit field, avoiding any mutex. When contended (CAS fails), the thread falls back to the BCB's `mutex` and enqueues on `next_wait_thrd`. The latch condition can also be `PGBUF_CONDITIONAL_LATCH` (non-blocking, returns failure immediately) or `PGBUF_UNCONDITIONAL_LATCH` (blocking, waits until granted).

### 3.3 Ordered Latch Protocol

Multi-page operations (heap scans, index traversals, overflow page access) must fix multiple pages simultaneously. Without ordering, two threads fixing pages in opposite order create a latch deadlock. CUBRID prevents this with a rank-based ordered latch protocol.

**PGBUF_ORDERED_RANK** (`page_buffer.h:222-229`) defines a hierarchy of page types:

```c
PGBUF_ORDERED_HEAP_HDR = 0      // Heap header page (highest priority, fix first)
PGBUF_ORDERED_HEAP_NORMAL = 1   // Normal heap data page
PGBUF_ORDERED_HEAP_OVERFLOW = 2 // Overflow page (lowest priority, fix last)
PGBUF_ORDERED_RANK_UNDEFINED    // For pages not in any group
```

**PGBUF_WATCHER** (`page_buffer.h:234-249`) tracks each page fix with its rank and group:

```
PGBUF_WATCHER
+---------------------------+
| pgptr: PAGE_PTR           |  -- pointer to fixed page
| group_id: VPID            |  -- heap header VPID (groups related pages)
| latch_mode: 7 bits        |  -- current latch mode
| page_was_unfixed: 1 bit   |  -- set if page was temporarily unfixed
| initial_rank: 4 bits      |  -- rank at initialization
| curr_rank: 4 bits         |  -- rank after fix (may differ)
| next/prev: PGBUF_WATCHER* |  -- doubly-linked watcher chain
+---------------------------+
```

The `pgbuf_ordered_fix()` function (`page_buffer.c`, 9 call sites) enforces the ordering rule: within the same group (identified by the heap header page's VPID), pages must be fixed in ascending rank order (HDR before NORMAL before OVERFLOW). Within the same rank, pages are fixed in ascending VPID order.

When a thread needs to fix a page that violates the current ordering — for example, it holds a NORMAL page and now needs the HDR page — `pgbuf_ordered_fix()` must temporarily unfix the higher-ranked page, fix the lower-ranked page, then refix the higher-ranked page. The `page_was_unfixed` flag is set to indicate that the page may have been modified by another thread during the unfix window, requiring the caller to re-validate any cached state from the page.

This protocol eliminates latch deadlocks for heap page operations at the cost of occasional unfix/refix cycles. The 9 call sites include heap scan operations, overflow page access, and heap file header operations.

### 3.4 Interactions

**Latching x Buffer Pool**: Every `pgbuf_fix()` call acquires a latch; `pgbuf_unfix()` releases it. The BCB's atomic latch field is the fast path. When contended, the thread suspends on the BCB's `next_wait_thrd` queue. The fix count (`fcnt`) tracks how many threads hold the page, enabling shared read access without mutual exclusion.

**Latching x WAL**: The log manager uses `CSECT_LOG` critical section to serialize log state modifications. During `logpb_flush_all_append_pages()`, the `LOG_CS` write lock is held to prevent concurrent log page modifications during flush. This is the coarsest latch in the WAL subsystem. Individual log page buffer slots have their own latches for concurrent append by different transactions.

**Latching x Lock Manager**: The lock manager's per-resource `res_mutex` is a fine-grained latch protecting holder/waiter list modifications. The WFG deadlock detector acquires `CSECT_WFG` to serialize graph construction and cycle detection. No latch ordering is defined between `res_mutex` instances (deadlock is avoided by acquiring only one at a time) or between `res_mutex` and page latches (they are never held simultaneously in the lock manager code path).

---

## 4. MVCC & Vacuum

CUBRID implements Multi-Version Concurrency Control (MVCC) to allow readers to access consistent snapshots without acquiring read locks. The MVCC subsystem spans `src/transaction/mvcc.c` (722 lines), supporting files (`mvcc_active_tran.hpp`, `mvcc_table.hpp`), and interacts deeply with the heap storage layer for version chain management. The vacuum subsystem (`src/query/vacuum.c`, 8,470 lines) runs as a background daemon to reclaim obsolete versions.

### 4.1 MVCC Record Header

Every heap record carries an MVCC header (`mvcc.h:38-46`):

```
MVCC_REC_HEADER (30 bytes)
+------------------------------+
| mvcc_flag: 8 bits            |  -- which fields are valid
| repid: 24 bits               |  -- representation ID
| chn: int                     |  -- cache coherency number
| mvcc_ins_id: MVCCID          |  -- inserter's transaction MVCCID
| mvcc_del_id: MVCCID          |  -- deleter's transaction MVCCID
| prev_version_lsa: LOG_LSA    |  -- WAL address of previous version
+------------------------------+
```

The `mvcc_flag` field controls which optional fields are present:
- `OR_MVCC_FLAG_VALID_INSID`: Insert MVCCID is set (set when the record is first created)
- `OR_MVCC_FLAG_VALID_DELID`: Delete MVCCID is set (set when the record is logically deleted or updated)
- `OR_MVCC_FLAG_VALID_PREV_VERSION`: A previous version exists in the WAL (set when the record has been updated, pointing to the pre-update version)

This two-timestamp design (insert + delete) means each record has a visibility window: it is visible to transactions whose snapshot falls between `mvcc_ins_id` (must be committed and visible) and `mvcc_del_id` (must not be committed and visible). An INSERT sets only `mvcc_ins_id`. A DELETE sets `mvcc_del_id` on the existing record. An UPDATE logically deletes the old version (sets `mvcc_del_id`) and inserts a new version (with `mvcc_ins_id` = current transaction and `prev_version_lsa` pointing to the undo log record containing the old version's data).

### 4.2 Snapshot Model

An MVCC snapshot (`mvcc.h:169-192`) captures a point-in-time view of which transactions are active:

```
MVCC_SNAPSHOT
+-----------------------------------+
| lowest_active_mvccid: MVCCID     |  -- floor: anything below is committed
| highest_completed_mvccid: MVCCID |  -- ceiling: anything above is in-flight
| m_active_mvccs: mvcc_active_tran |  -- set of active transaction IDs
| snapshot_fnc: function pointer   |  -- visibility check function
| valid: bool                      |  -- snapshot validity flag
+-----------------------------------+
```

The `snapshot_fnc` function pointer allows different visibility semantics: `mvcc_satisfies_snapshot` for normal reads, `mvcc_satisfies_delete` for delete operations, `mvcc_satisfies_vacuum` for garbage collection decisions, and `mvcc_satisfies_dirty` for dirty reads.

**Active transaction tracking** uses a hybrid bit-area approach (`mvcc_active_tran.hpp:31-60`). For the dense range of active MVCCIDs near the current global MVCCID, a bit array indexed relative to `bit_area_start_mvccid` provides O(1) `is_active(mvccid)` checks. The bit area is sized at `500 * 64 bits = 32,000 bits`, covering the most recent 32,000 MVCCIDs. Long-running transactions whose MVCCIDs fall outside the bit-area window are tracked in a separate sparse long-transaction array. This hybrid design ensures O(1) lookups for the common case (recent transactions) while supporting arbitrary-age long-running transactions.

**Snapshot construction** is lazy: `logtb_get_mvcc_snapshot()` (`log_tran_table.c`) builds the snapshot on the first query within a transaction, not at `BEGIN TRANSACTION`. The global `mvcc_trans_status` (`mvcc_table.hpp:40-62`) uses `std::atomic<version_type> m_version` for lock-free snapshot reads. The algorithm: a reading transaction atomically captures the version counter, then copies the active transaction bit-area and the `lowest_active_mvccid` / `highest_completed_mvccid` bounds, then re-reads the version counter. If the version has not changed, the snapshot is consistent. If it changed (concurrent commit occurred during the copy), the process retries. This ensures snapshot atomicity without locks. After the first query, subsequent queries in the same transaction reuse the existing snapshot (for REPEATABLE READ) or rebuild it (for READ COMMITTED).

### 4.3 Visibility Checking

The core visibility function is `mvcc_satisfies_snapshot()` (`mvcc.c:156`), which returns one of three results:

- `TOO_OLD_FOR_SNAPSHOT`: Record was deleted before the snapshot — invisible, skip
- `SNAPSHOT_SATISFIED`: Record is visible to this snapshot — return to client
- `TOO_NEW_FOR_SNAPSHOT`: Record was inserted after the snapshot — invisible, but the previous version (via `prev_version_lsa`) might be visible

The decision tree handles 9 cases (`mvcc.c:155-269`):

1. **No ins_id flag set** → Record has no MVCC metadata, treat as committed and visible → `SNAPSHOT_SATISFIED`
2. **ins_id is own transaction** → Own inserts are always visible. Check del_id:
   - If del_id is set AND is own transaction → own delete, `TOO_OLD_FOR_SNAPSHOT`
   - If del_id is not own → `SNAPSHOT_SATISFIED`
3. **ins_id is active** (checked via `m_active_mvccs.is_active()`) → Inserter hasn't committed → `TOO_NEW_FOR_SNAPSHOT`
4. **ins_id > highest_completed_mvccid** → Inserted after snapshot was taken → `TOO_NEW_FOR_SNAPSHOT`
5. **ins_id is committed and visible**. Now check del_id:
   - If no del_id flag → not deleted → `SNAPSHOT_SATISFIED`
   - If del_id is own transaction → own delete, `TOO_OLD_FOR_SNAPSHOT`
   - If del_id is active → deleter hasn't committed → `SNAPSHOT_SATISFIED` (deletion not yet visible)
   - If del_id > highest_completed_mvccid → deleted after snapshot → `SNAPSHOT_SATISFIED`
   - If del_id is committed and < lowest_active → deletion is visible → `TOO_OLD_FOR_SNAPSHOT`

The `mvcc_satisfies_vacuum()` function determines whether a record can be garbage-collected. It returns `VACUUM_RECORD_REMOVE` (physically remove the record), `VACUUM_RECORD_DELETE_INSID_PREV_VER` (clear the insert MVCCID and prev_version_lsa but keep the record), or `VACUUM_RECORD_CANNOT_VACUUM` (record is still needed by active transactions).

### 4.4 Version Chain via prev_version_lsa

This is CUBRID's most architecturally distinctive MVCC mechanism. Unlike PostgreSQL (which stores old tuple versions in the heap, creating HOT chains), or MySQL/InnoDB (which uses a separate undo tablespace with rollback segments), CUBRID stores version deltas directly in the WAL and chains them via the `prev_version_lsa` field in the MVCC record header.

**Write path** (during UPDATE):
1. The new version is written to the heap with `mvcc_ins_id` = current transaction's MVCCID
2. The old record's data is logged as the undo portion of a `LOG_MVCC_UNDOREDO_DATA` WAL record
3. The new record's `prev_version_lsa` is set to the LSA of that WAL record (`heap_update_set_prev_version()`, `heap_file.c:893`)
4. The old record's `mvcc_del_id` is set to the current transaction's MVCCID (marking it as logically deleted)

**Read path** (when current version is not visible):
1. Reader calls `mvcc_satisfies_snapshot()` → returns `TOO_NEW_FOR_SNAPSHOT`
2. Check `prev_version_lsa` — if not `NULL_LSA`, a previous version exists
3. Call `heap_get_visible_version_from_log()` (`heap_file.c:890`)
4. Fix the log page in the buffer pool: `pgbuf_fix(log_page_vpid, PGBUF_LATCH_READ)`
5. Read the `LOG_MVCC_UNDOREDO_DATA` record at the specified offset
6. Extract the undo data to reconstruct the previous version's content and MVCC header
7. Unfix the log page
8. Check visibility of the reconstructed version via `mvcc_satisfies_snapshot()`
9. If visible: return this version. If `TOO_NEW_FOR_SNAPSHOT` and it has its own `prev_version_lsa`: recurse (follow the chain further). If `TOO_OLD_FOR_SNAPSHOT` or no prev version: no visible version exists.

**Design tradeoffs**: This approach avoids a separate version store and leverages the existing WAL infrastructure for version storage. It means version data is automatically included in log archives and backups. However, old-version reads require log page I/O (potentially reading from archived log volumes), and version chain length depends on vacuum activity. Without timely vacuum, chains grow unboundedly, degrading read performance for long-running transactions that need old versions.

### 4.5 Vacuum Subsystem

The vacuum subsystem (`src/query/vacuum.c`, 8,470 lines) runs as a master-worker daemon system that reclaims obsolete MVCC versions, removing delete IDs, clearing `prev_version_lsa` pointers, and physically removing fully deleted records.

**Master-worker architecture**:

```
vacuum_master_task::execute()     -- master daemon (line ~2988)
  |
  +-- Scans WAL log blocks (identified by blockid) for MVCC operations
  +-- For each block: creates VACUUM_DATA_ENTRY with {blockid, start_lsa, MVCCID range}
  +-- Compares oldest_visible threshold (min of all active snapshots)
  +-- Enqueues eligible blocks for worker threads
  |
  v
vacuum_process_log_block()        -- worker entry point (line ~3205)
  |
  +-- Reads log records from the assigned log block
  +-- For each MVCC record: calls mvcc_satisfies_vacuum(oldest_visible)
  +-- Dispatches to appropriate action based on result
  |
  v
vacuum_heap_page()                -- heap page cleanup (line ~1568)
  |
  +-- Fixes the heap page (PGBUF_LATCH_WRITE)
  +-- For each record: applies vacuum action
  +-- vacuum_heap_record_insid_and_prev_version() (line ~2186, static)
       Clears insert MVCCID and prev_version_lsa from records
       that no longer need version chain traversal
  +-- Unfixes the page (dirty if modified)
```

**Vacuum safety invariant**: The master daemon computes the `oldest_visible` threshold as the minimum MVCCID across all active transactions' snapshots (or the global `lowest_active_mvccid` if no snapshots exist). Only records whose MVCC IDs are older than `oldest_visible` are eligible for vacuum. Critically, `prev_version_lsa` is **never** cleared while `del_id >= oldest_visible` (`mvcc_satisfies_vacuum()`, `mvcc.c:323-327`), because some active transaction may need to follow the chain to find a visible older version.

**WAL interaction**: The master daemon tracks progress using `blockid`-indexed consumption of WAL log blocks. Each `VACUUM_DATA_ENTRY` represents one log block worth of MVCC operations to process. Vacuum data entries are persisted in `VACUUM_DATA_PAGE` structures linked as a list of pages, providing crash-safe progress tracking. After a crash, vacuum resumes from the last persisted entry, and since all vacuum operations are themselves WAL-logged, they are idempotent during recovery.

**Record compaction**: When vacuum determines a record's `mvcc_ins_id` is no longer needed (all active transactions can see this version as committed), it clears the INSID and `prev_version_lsa` from the record, compacting the MVCC header. This reduces the per-record overhead and terminates the version chain at this point, preventing further chain traversal for this record.

### 4.6 Interactions

**MVCC x Lock Manager**: MVCC provides non-blocking reads; the lock manager prevents write-write conflicts. A `SELECT` acquires no object locks (only a snapshot). An `UPDATE` acquires both an `X_LOCK` and writes a new MVCC version. The `non2pl` list (Section 2.4) bridges these: locks released early remain logically in effect through MVCC visibility rules.

**MVCC x WAL**: The `prev_version_lsa` field creates a physical chain from heap records into WAL log pages. MVCC undo data is embedded in `LOG_MVCC_UNDOREDO_DATA` records. Version reconstruction requires fixing log pages in the buffer pool. The vacuum daemon's log block consumption depends on WAL archive availability — if archive logs are deleted before vacuum processes them, some version cleanup may be deferred.

**MVCC x Buffer Pool**: `heap_get_visible_version_from_log()` calls `pgbuf_fix()` on log pages to read previous versions. This means MVCC version traversal competes for buffer pool slots and latches with normal data page access. Under heavy long-transaction workloads, version chain traversal can amplify buffer pool pressure.

---

## 5. WAL, Recovery & Commit

The Write-Ahead Logging subsystem ensures durability and supports crash recovery. It spans `src/transaction/log_manager.c` (15,278 lines), `log_page_buffer.c` (11,598 lines), `log_recovery.c` (6,500 lines), and several supporting files.

### 5.1 Log Record Structure

Every log record begins with `LOG_RECORD_HEADER` (`log_record.hpp:146-153`):

```
LOG_RECORD_HEADER
+---------------------------+
| prev_tranlsa: LOG_LSA     |  -- previous record for same transaction (undo chain)
| back_lsa: LOG_LSA         |  -- backward log address (physical backward link)
| forw_lsa: LOG_LSA         |  -- forward log address (physical forward link)
| trid: TRANID              |  -- transaction identifier
| type: LOG_RECTYPE         |  -- record type (52 types)
+---------------------------+
```

The `prev_tranlsa` field creates a per-transaction backward chain through the log. During undo (abort or recovery), this chain is followed backwards to find all records that need to be undone for a specific transaction, without scanning the entire log. The `back_lsa`/`forw_lsa` fields form the physical log chain, linking all records regardless of transaction.

The `LOG_RECTYPE` enum (`log_record.hpp:35-141`) defines 52 record types organized into categories:

**Data modification records**: `LOG_UNDOREDO_DATA` (2), `LOG_UNDO_DATA` (3), `LOG_REDO_DATA` (4) carry the actual before/after images of modified data. For MVCC operations, the variants `LOG_MVCC_UNDOREDO_DATA` (46), `LOG_MVCC_UNDO_DATA` (47), `LOG_MVCC_REDO_DATA` (48) additionally carry MVCC metadata (insert/delete MVCCIDs).

**Transaction boundary records**: `LOG_COMMIT` (17) marks successful completion; `LOG_ABORT` (22) marks rollback. These records trigger lock release and MVCC ID advancement.

**System operation records**: `LOG_SYSOP_END` (20) with sub-types for nested top-level operations. System operations (like B-tree structure modifications) are made independently atomic — they commit regardless of whether the enclosing transaction commits.

**Compensation records**: `LOG_COMPENSATE` (8) is written during undo to record the physical effect of undoing a prior record. If a crash occurs during undo, the compensation record prevents re-undoing the same change (idempotent recovery).

**Checkpoint records**: `LOG_START_CHKPT` (25) and `LOG_END_CHKPT` (26) bracket a checkpoint, recording the transaction table snapshot and dirty page information.

The LSN (Log Sequence Number) is `LOG_LSA` — a `{pageid, offset}` pair that uniquely and monotonically identifies a position in the log stream. LSAs are used for ordering (which modification came first), recovery start points, and the WAL protocol (page LSA vs. log flush LSA comparison).

### 5.2 Group Commit

CUBRID supports group commit to batch multiple transaction commits into a single log flush, amortizing the I/O cost across concurrent transactions.

**Policy layer** (`logpb_flush_pages`, `log_page_buffer.c:3976`): This function decides how to handle a commit flush request based on two configuration parameters:

| `log_async_commit` | `log_group_commit_interval_msecs > 0` | Behavior |
|-|-|-|
| false | false | **Synchronous normal**: wake log flush daemon, wait for completion |
| false | true | **Group commit**: wait on `gc_cond` (batched with concurrent committers) |
| true | false | **Async normal**: wake daemon, return immediately |
| true | true | **Async group**: return immediately, no daemon wake |

The `LOG_GROUP_COMMIT_INFO` structure (`log_impl.h:339-348`) contains a `gc_mutex` and `gc_cond` condition variable. The synchronous group commit path works as follows: the committing thread checks if the current `nxio_lsa` (next I/O LSA, the point up to which the log has been flushed to disk) is already past its commit record's LSA. If not, it waits on `gc_cond` with a timeout. The log flush daemon thread (`logpb_flush_all_append_pages`) periodically flushes all pending log pages and broadcasts `gc_cond`, waking all waiting committers. This batching reduces the number of `fsync()` calls from one-per-commit to one-per-flush-interval. The check macro `LOG_IS_GROUP_COMMIT_ACTIVE()` (`log_impl.h:124-125`) tests whether the interval parameter is greater than zero.

**Mechanism layer** (`logpb_flush_all_append_pages`, `log_page_buffer.c:3228`): The physical flush function acquires the `LOG_CS` write lock (preventing concurrent log appends during flush), writes all dirty log pages from the `LOG_FLUSH_INFO.toflush` sorted array to disk via the I/O subsystem, advances `nxio_lsa`, and releases the lock. The `toflush` array is sorted by page ID to enable sequential I/O.

### 5.3 Checkpoint

Checkpoints (`logpb_checkpoint`, `log_page_buffer.c:6873`) record a consistent snapshot of the transaction system state to bound recovery time. Without checkpoints, recovery would need to scan the entire log from the beginning.

The checkpoint record (`LOG_REC_CHKPT`, `log_record.hpp:344-350`) contains:
- `redo_lsa`: The oldest LSN of any dirty page in the buffer pool. Recovery need not redo anything before this point, because all prior modifications have already been flushed to disk.
- `ntrans`: Number of active transactions at checkpoint time
- Per-transaction state (`LOG_INFO_CHKPT_TRANS`, 14 fields per transaction): `trid`, `state`, `head_lsa` (first log record), `tail_lsa` (last log record), `undo_nxlsa` (next undo record), `savept_lsa` (latest savepoint), and more

The checkpoint algorithm:
1. Write `LOG_START_CHKPT` record (marks the beginning of checkpoint)
2. Capture `log_Gl.chkpt_redo_lsa` — the oldest dirty page LSN tracked by the buffer pool (minimum `oldest_unflush_lsa` across all dirty BCBs)
3. Snapshot all active transactions from the transaction table, recording their state and LSA pointers
4. Write `LOG_END_CHKPT` record with the complete snapshot data
5. Flush log pages to ensure the checkpoint records are durable
6. Update the log header with the new checkpoint LSA

Checkpoints run periodically (configurable interval) and are triggered by the checkpoint daemon thread.

### 5.4 Crash Recovery

Recovery (`log_recovery.c`) follows the ARIES-style three-phase protocol:

**Analysis** (`log_recovery_analysis`, line 92): Scans the log forward from the last checkpoint LSA to the end of the log. Rebuilds the transaction table (which transactions were active, their state, and their undo chain pointers). Identifies the redo starting point as the minimum of the checkpoint's `redo_lsa` and all active transactions' earliest LSAs.

**Redo** (`log_recovery_redo`, line 101): Replays all logged operations from the redo starting point forward. For each log record, the redo function fixes the affected page, compares the page's LSN with the log record's LSA. If `page_lsa < log_lsa`, the page is stale and the redo is applied. If `page_lsa >= log_lsa`, the page is already up-to-date (the modification was flushed before the crash) and the redo is skipped. This idempotent comparison is the core of ARIES-style redo.

CUBRID supports **parallel redo** via the `redo_parallel` class (`log_recovery_redo_parallel.hpp:56-206`). This class creates a worker pool with a pre-allocated pool of 1 million reusable job objects (`PARALLEL_REDO_REUSABLE_JOBS_COUNT = ONE_M`). Redo jobs are distributed by VPID, ensuring that all redo operations for the same page are serialized within a single worker while different pages are processed in parallel across workers. The `min_unapplied_log_lsa_monitoring` component (`lines 131-183`) tracks the lowest not-yet-applied LSN across all workers to prevent premature log archive deletion — the archive must be retained until all workers have processed past it.

**Undo** (`log_recovery_undo`, line 109): Processes each transaction in `TRAN_UNACTIVE_ABORTED` state. For each, walks the undo chain backwards via `prev_tranlsa`, applying the registered recovery function for each undo record (`log_rv_undo_record`, line 162). For each undo: fix the page, call the recovery handler (e.g., `RVHF_MVCC_INSERT` reversal), write a `LOG_COMPENSATE` record, mark the page dirty, and unfix. The compensation record ensures that if a crash occurs during undo, the already-undone operations are not re-undone. `lock_reacquire_crash_locks()` rebuilds the lock table during recovery to prevent conflicts between undo operations and any concurrent access in HA configurations.

### 5.5 Two-Phase Commit

CUBRID supports distributed transactions via 2PC (`log_2pc.c`, 2,457 lines). During the prepare phase, the coordinator writes `LOG_2PC_PREPARE`, which forces the transaction's log records to disk and transitions the transaction to a prepared state (locks held, outcome undecided). The coordinator collects votes from participants, then writes `LOG_2PC_COMMIT_DECISION` or `LOG_2PC_ABORT_DECISION`. During crash recovery, prepared-but-not-decided transactions are treated as in-doubt: their locks are reacquired and held until an external coordinator resolves the outcome.

### 5.6 Interactions

**WAL x Buffer Pool**: The WAL protocol invariant: a dirty page cannot be flushed to disk until all log records modifying that page have been flushed first. The BCB's `oldest_unflush_lsa` field tracks this. During `pgbuf_flush_checkpoint()`, pages are only flushed if their log records (up to `oldest_unflush_lsa`) have already been written to disk (i.e., `nxio_lsa >= oldest_unflush_lsa`). The checkpoint's `redo_lsa` is derived from the minimum `oldest_unflush_lsa` across all dirty BCBs.

**WAL x MVCC**: MVCC undo data for updates is embedded in `LOG_MVCC_UNDOREDO_DATA` records. The `prev_version_lsa` pointer in `MVCC_REC_HEADER` addresses into the WAL, creating a version chain. Recovery must preserve the integrity of these chains — redo re-establishes the heap state including MVCC headers, and undo reverses MVCC operations using the logged insert/delete MVCCIDs.

**WAL x Lock Manager**: During recovery redo, the lock manager is inactive — redo is purely physical (page-level), applied based on LSN comparison. During undo, locks are reacquired (`lock_reacquire_crash_locks`) to prevent conflicts in HA configurations where other servers might access the recovering database.

---

## 6. Buffer Pool Management

The buffer pool (`src/storage/page_buffer.c`, 16,931 lines) manages the cache of disk pages in memory, implementing page replacement, dirty page tracking, and flush coordination.

### 6.1 BCB Architecture

Each buffer slot is managed by a Buffer Control Block (BCB, `page_buffer.c:503-535`):

```
PGBUF_BCB (Buffer Control Block)
+----------------------------------+
| mutex: pthread_mutex_t           |  -- BCB-level mutex (fallback path)
| vpid: VPID                      |  -- volume + page ID of cached page
| atomic_latch: PGBUF_ATOMIC_LATCH|  -- 64-bit: latch_mode(2) + waiter(1) + fcnt(29)
| flags: volatile int             |  -- dirty, flushing, etc.
| next_wait_thrd: THREAD_ENTRY*   |  -- latch wait queue head
| hash_next: PGBUF_BCB*           |  -- hash chain for VPID lookup
| prev_BCB / next_BCB: PGBUF_BCB* |  -- LRU doubly-linked chain
| tick_lru_list: int              |  -- age when inserted into LRU
| tick_lru3: int                  |  -- position tracking in LRU zone 3
| count_fix_and_avoid_dealloc: int|  -- hot page detection + dealloc prevention
| hit_age: int                    |  -- age of last access (for quota)
| oldest_unflush_lsa: LOG_LSA     |  -- oldest unflushed modification LSN
| iopage_buffer: PGBUF_IOPAGE_BUF*|  -- pointer to actual page data
+----------------------------------+
```

The `atomic_latch` is a 64-bit packed value (`page_buffer.c:491-500`) containing `latch_mode` (2 bits: NO_LATCH/READ/WRITE/FLUSH), `waiter_exists` (1 bit: true if any thread is waiting for a latch), and `fcnt` (29 bits: the number of threads currently holding the page). This packing enables single-CAS latch operations in the uncontended case. For reads, the CAS increments `fcnt` and sets mode to READ. For writes, it checks `fcnt == 0` and sets mode to WRITE. If the CAS fails (contention), the thread acquires the BCB's `mutex` and enqueues on `next_wait_thrd`.

The `count_fix_and_avoid_dealloc` field serves a dual purpose: the upper bits count the "hot" fix frequency (used for LRU zone boosting decisions), while the lower bits prevent page deallocation while another thread has a reference. The `hit_age` field records the LRU tick at the time of the last hit, used with the LRU list's global `tick_list` counter to determine page temperature for zone assignment.

### 6.2 Page Fix/Unfix Protocol

`pgbuf_fix()` (`page_buffer.h:275`) is the primary page access function:

1. **Hash lookup**: Find BCB by VPID in the hash table. The hash table uses separate chaining with `hash_next` pointers.
2. **Buffer hit** (BCB found):
   - Attempt CAS on `atomic_latch`:
     - Read latch: if current mode is NO_LATCH or READ, increment `fcnt` and set READ
     - Write latch: if `fcnt == 0`, set mode to WRITE and `fcnt = 1`
   - If CAS succeeds: return page pointer immediately (fast path)
   - If CAS fails (contented): acquire BCB `mutex`, enqueue `THREAD_ENTRY` on `next_wait_thrd`, wait on thread's `wakeup_cond`, and retry after wakeup
3. **Buffer miss** (BCB not found):
   - Select a victim BCB from the LRU (see Section 6.3)
   - If victim is dirty: flush it to disk first (WAL protocol check included)
   - Read the requested page from disk into the victim's buffer
   - Insert new VPID-to-BCB mapping in the hash table
   - Acquire the requested latch on the new BCB
4. **Return**: Pointer to the page within the BCB's `iopage_buffer`

The fetch mode (`PAGE_FETCH_MODE`) influences the fix behavior: `OLD_PAGE` expects the page to exist on disk or in buffer; `NEW_PAGE` is for newly allocated pages; `OLD_PAGE_IF_IN_BUFFER` only succeeds if the page is already cached (avoids I/O); `RECOVERY_PAGE` is used during crash recovery where pages may be in any state.

`pgbuf_unfix()` releases the latch by CAS-decrementing `fcnt`. If `fcnt` reaches 0 and `waiter_exists` is set, it wakes the first waiter from the BCB's `next_wait_thrd` queue. `pgbuf_set_dirty()` marks the BCB dirty and updates `oldest_unflush_lsa` to track the WAL protocol requirement.

### 6.3 Three-Zone LRU

CUBRID uses a three-zone LRU replacement policy (`page_buffer.c:577-615`) that classifies pages by access recency:

```
+------- top (most recently used) -------+
|                                         |
|  LRU_1_ZONE: Hot pages (~25%)           |
|  Recently accessed, protected from      |
|  eviction. Newly fixed pages enter here.|
|                                         |
+-- bottom_1 ----------------------------+
|                                         |
|  LRU_2_ZONE: Warm pages (~50%)          |
|  Moderately recent. Candidates for      |
|  demotion to cold zone.                 |
|                                         |
+-- bottom_2 ----------------------------+
|                                         |
|  LRU_3_ZONE: Cold pages (remainder)     |
|  Victim candidates for replacement.     |
|  Victim search starts from bottom.      |
|                                         |
+------- bottom (least recently used) ---+
```

Each `pgbuf_lru_list` (`page_buffer.c:577-615`) tracks zone boundaries with `bottom_1` and `bottom_2` pointers separating the three zones, zone counts (`count_lru1`, `count_lru2`, `count_lru3`), and configurable thresholds. Zone ratios are tunable via `PGBUF_LRU_ZONE_MIN_RATIO` and `PGBUF_LRU_ZONE_MAX_RATIO` (`page_buffer.c:339-340`). The default thresholds are approximately 25% for LRU1 and 50% for LRU2, with the remainder in LRU3.

**Zone transitions**: A newly fixed page enters at the **top** of LRU1 (most recently used). A page hit (re-fix of a cached page) boosts the page back to the top of LRU1. As pages age without hits, they naturally migrate: when LRU1 exceeds its threshold, `bottom_1` advances upward and pages fall into LRU2. Similarly, LRU2 overflow pushes pages into LRU3. Victim selection starts from the `bottom` of LRU3 and works upward, choosing the first page with `fcnt == 0` (not fixed) and no `avoid_dealloc` flag.

A `victim_hint` pointer accelerates victim search by tracking where the last victim was found, avoiding repeated scanning of the same cold pages. The `tick_list` counter is incremented when BCBs are added or boosted, providing an age metric. `tick_lru3` tracks positions within the cold zone for efficient hint updates.

**Concurrency**: Each LRU list is protected by its own `pthread_mutex_t`, allowing concurrent page replacement decisions across different LRU lists (the buffer pool maintains multiple LRU lists to reduce contention). Zone transitions acquire this mutex briefly to update the boundary pointers and counts.

### 6.4 WAL Protocol Enforcement

When `pgbuf_set_dirty()` is called, the BCB's `oldest_unflush_lsa` is updated if the current LSN is older than the stored one. During flush operations:

1. Compare `log_Gl.append.nxio_lsa` (log flush point) with `page.oldest_unflush_lsa`
2. If `nxio_lsa >= oldest_unflush_lsa`: all log records for this page are on disk, safe to flush
3. If `nxio_lsa < oldest_unflush_lsa`: log must be flushed first — call `logpb_flush_all_append_pages()` to write pending log pages, then proceed with the data page flush

The checkpoint algorithm reads the minimum `oldest_unflush_lsa` across all dirty BCBs to set the `redo_lsa` in the checkpoint record. This determines where recovery will start the redo phase.

### 6.5 Flush Daemons

Several daemon threads manage page flushing asynchronously:
- `pgbuf_page_flush_daemon_task`: Periodically flushes dirty pages when the dirty ratio exceeds a configured threshold, writing them to disk in VPID order for sequential I/O
- `pgbuf_page_maintenance_daemon_task`: LRU maintenance — adjusts zone boundaries, updates victim hints, handles quota management for private LRU lists
- `pgbuf_page_post_flush_daemon_task`: Post-flush cleanup — resets BCB dirty flags and updates LRU positions after successful writes
- `pgbuf_flush_control_daemon_task`: Monitors the dirty page ratio and triggers emergency flushes if it exceeds the high-water mark

### 6.6 Interactions

**Buffer Pool x WAL**: The `oldest_unflush_lsa` field in BCBs bridges these subsystems. Checkpoint reads the minimum across all dirty BCBs to determine `redo_lsa`. Flush daemons check WAL flush progress before writing dirty pages. The log flush daemon's broadcast on `gc_cond` indirectly unblocks page flush operations that were waiting for log records to be written.

**Buffer Pool x Latching**: Page latches (Section 3.2) are stored in BCBs. The ordered latch protocol (Section 3.3) prevents deadlocks during multi-page operations. The LRU's per-list mutex serializes chain modifications. Page latch waits are tracked in `THREAD_ENTRY->m_pgbuf_tracker` for leak detection.

**Buffer Pool x MVCC**: Version chain traversal via `prev_version_lsa` requires fixing log pages in the buffer pool. These log page fixes compete with data page access for buffer slots and latches. Under heavy long-transaction workloads with deep version chains, this can amplify buffer pool pressure and increase I/O.

---

## 7. Concurrency Infrastructure

The thread management and lock-free data structure subsystems provide the foundation upon which all other concurrency mechanisms are built.

### 7.1 THREAD_ENTRY

The `cubthread::entry` class (`src/thread/thread_entry.hpp:195-386`), typedef'd as `THREAD_ENTRY`, is the per-thread context carrier passed as the first parameter to virtually every function in the CUBRID server. It is the universal context object:

```
THREAD_ENTRY (cubthread::entry) - Key fields:
+------------------------------------------+
| index: int                               |  -- thread pool index
| type: thread_type                        |  -- TT_WORKER, TT_DAEMON, TT_VACUUM, etc.
| tran_index: int                          |  -- current transaction table index
| m_status: status                         |  -- TS_DEAD/TS_FREE/TS_RUN/TS_WAIT/TS_CHECK
|                                          |
| -- Synchronization --                    |
| th_entry_lock: pthread_mutex_t           |  -- per-thread latch
| wakeup_cond: pthread_cond_t              |  -- wakeup condition variable
|                                          |
| -- Lock wait state --                    |
| lockwait: void*                          |  -- LK_ENTRY* if waiting for object lock
| lockwait_stime: INT64                    |  -- wait start timestamp (ms)
| lockwait_msecs: int                      |  -- lock timeout value (ms)
| lockwait_state: int                      |  -- LOCK_SUSPENDED/RESUMED/DEADLOCK/etc.
| tran_next_wait: entry*                   |  -- WFG construction chain
|                                          |
| -- Lock-free transaction descriptors --  |
| tran_entries[THREAD_TS_COUNT]: lf_tran*  |  -- one per lock-free structure type
| m_lf_tran_index: lockfree::tran::index   |  -- global lock-free transaction index
|                                          |
| -- Resource tracking --                  |
| m_pgbuf_tracker: pgbuf_tracker&          |  -- page fix/unfix balance checking
| m_csect_tracker: critical_section_tracker&| -- critical section order validation
| m_error: cuberr::context                 |  -- per-thread error state
+------------------------------------------+
```

The lock wait states (`thread_entry.hpp:177-188`) encode the reason a thread was woken from a lock wait. They form a state machine: `LOCK_SUSPENDED` → (one of) `LOCK_RESUMED` (granted), `LOCK_RESUMED_TIMEOUT` (timed out), `LOCK_RESUMED_DEADLOCK_TIMEOUT` (deadlock victim), `LOCK_RESUMED_ABORTED` (transaction aborted due to deadlock), `LOCK_RESUMED_INTERRUPT` (interrupted by another mechanism).

The `tran_entries[]` array is indexed by `THREAD_TS_*` constants, each representing a different lock-free data structure type that needs independent hazard-pointer tracking. For example, `THREAD_TS_OBJ_LOCK_RES` is for the lock manager's `LK_RES` hash table, `THREAD_TS_OBJ_LOCK_ENT` is for `LK_ENTRY` freelist operations. Each entry is a `lockfree::tran::descriptor` (see Section 7.3) that tracks the thread's current epoch and retired node list for that structure type.

### 7.2 Thread Pools and Daemons

The thread manager (`src/thread/thread_manager.hpp`) manages two categories of threads:

**Worker pools** (`thread_worker_pool.hpp`): Template-based worker pools that execute queued tasks. A pool maintains a set of worker threads that dequeue tasks from a shared work queue and execute them. Worker pools are used for: parallel redo recovery (with 1M reusable job objects for high throughput), vacuum workers (processing log blocks), and general client request processing. Each worker thread has its own `THREAD_ENTRY` with an assigned `tran_index`.

**Daemon threads**: Single-purpose background threads for periodic or event-driven tasks. Key daemons include: log flush daemon (flushes log pages on commit), checkpoint daemon (periodic checkpoint), vacuum master daemon (scans WAL for vacuum work), page flush daemons (flush dirty data pages), page maintenance daemon (LRU management), and the deadlock detection daemon (WFG cycle detection). Daemons are typically single-threaded and wake on either a timer or an explicit signal from a worker thread.

### 7.3 Lock-Free Data Structures

CUBRID's lock-free infrastructure (8+ files in `src/base/lockfree_*.hpp`) provides concurrent data structures without traditional locking, used for performance-critical internal tables.

**lockfree::hashmap** (`lockfree_hashmap.hpp:40-171`): A bucket-chained hash map where `find()`, `insert()`, and `erase()` use CAS (Compare-And-Swap) operations on bucket chain pointers. The `address_marker<T>` template wraps pointers with a tag bit to distinguish deleted entries from live ones in the chain, solving the ABA problem. Inserted entries are allocated from a `freelist<T>`, and erased entries are retired to the calling thread's descriptor for deferred reclamation. The hash map supports both find-only (read path, no allocation) and find-or-insert (read-modify-write path, allocates on miss) operations.

**Two-layer architecture**: The core implementation lives in `lockfree::hashmap` (namespace `lockfree` in `src/base/lockfree_hashmap.hpp`). A thread-aware wrapper `cubthread::lockfree_hashmap` (`src/thread/thread_lockfree_hash_map.hpp:34-87`) adds integration with `THREAD_ENTRY`: it extracts the appropriate `tran_entries[]` descriptor from the thread context and delegates to the core implementation. The wrapper also supports a hybrid mode via `PRM_ID_ENABLE_NEW_LFHASH`, allowing runtime selection between the modern `lockfree::hashmap` and a legacy `lf_hash_table_cpp` implementation. The lock manager, catalog manager, and several other subsystems use the `cubthread::lockfree_hashmap` wrapper.

**Hazard-pointer-based reclamation**: The lock-free transaction system ensures safe memory reclamation without garbage collection:

```
lockfree::tran::system            -- global: manages transaction index pool
  |                                  (bitmap-based index allocation)
  +-- lockfree::tran::table       -- per-system: maps indices to descriptors
       |
       +-- lockfree::tran::descriptor  -- per-thread: epoch + retired list
            |
            +-- reclaimable_node  -- base class for reclaimable entries
                                     (virtual reclaim() method)
```

Each thread's `descriptor` (`lockfree_transaction_descriptor.hpp:51-97`) maintains a `m_tranid` (the thread's current epoch, fetched from the global counter at `start_tran()`), a linked list of retired nodes (`m_retired_head`/`m_retired_tail`), and statistics. The reclamation algorithm: when `retire_node()` is called, the node's `m_retire_tranid` is set to the current epoch and it is appended to the retired list. When `reclaim_retired_list()` runs, it computes the minimum `m_tranid` across all active descriptors in the system. Any retired node whose `m_retire_tranid` is less than this minimum is guaranteed to be unobservable by any concurrent thread, and its `reclaim()` method is called (which typically returns it to the freelist).

**lockfree::freelist** (`lockfree_freelist.hpp:42-101`): Provides memory pooling for lock-free structures. The primary pool is an atomic LIFO `m_available_list` (CAS-based push/pop). A `m_backbuffer` pre-allocated reserve list reduces CAS contention during allocation spikes: when the available list is exhausted, `swap_backbuffer()` atomically installs the reserve as the new available list, and a new reserve is allocated in the background. The `alloc_backbuffer()` function pre-populates blocks of entries (configurable `block_size` and `initial_block_count`), amortizing allocation costs.

### 7.4 Interactions

**Thread Model x Lock Manager**: The lock manager's hash table is a `cubthread::lockfree_hashmap<LK_RES_KEY, LK_RES>` (line 355). Each thread uses `tran_entries[THREAD_TS_OBJ_LOCK_RES]` for hazard-pointer-protected `LK_RES` access, and `tran_entries[THREAD_TS_OBJ_LOCK_ENT]` for `LK_ENTRY` freelist operations. Lock wait/resume flows through `THREAD_ENTRY->lockwait` and `wakeup_cond`.

**Thread Model x MVCC**: `THREAD_ENTRY->tran_index` maps to the transaction table entry, which holds the MVCC snapshot. The lock-free transaction descriptors in `tran_entries[]` are separate from MVCC transactions — they share the thread context but serve different purposes (memory safety for internal data structures vs. data visibility for application queries).

**Thread Model x WAL**: The log flush daemon is a dedicated daemon thread. Committing worker threads signal it via `log_wakeup_log_flush_daemon()`. The parallel redo recovery system uses the `cubthread::entry_workpool` worker pool template with its own daemon threads and task queue.

**Thread Model x Buffer Pool**: `THREAD_ENTRY->m_pgbuf_tracker` monitors page fix/unfix balance per thread, detecting leaks in debug builds. The BCB's `next_wait_thrd` queue links `THREAD_ENTRY` nodes for latch wait ordering. Each thread's critical section tracker (`m_csect_tracker`) validates that critical sections are acquired in a consistent order to prevent deadlocks.

---

## 8. Cross-Cutting Flows

This section traces four end-to-end operations across multiple subsystems, showing how the concurrency components interact at runtime.

### 8.1 Flow: SELECT Under MVCC Snapshot

A read-only `SELECT` statement demonstrates MVCC snapshot isolation without object locking.

```
Client: SELECT * FROM t WHERE id = 100

1. Transaction context [Transaction Table]
   Thread already has tran_index assigned in THREAD_ENTRY.
   If first query: transaction is in TRAN_ACTIVE state.

2. Snapshot acquisition [MVCC + Transaction Table]
   logtb_get_mvcc_snapshot(thread_p)               -- log_tran_table.c
     → Atomically read mvcc_trans_status.m_version
     → Copy active transaction bit-area into local snapshot
     → Copy lowest_active_mvccid, highest_completed_mvccid
     → Re-read m_version to verify consistency (retry if changed)
     → Set snapshot.valid = true
     → Set snapshot.snapshot_fnc = mvcc_satisfies_snapshot

3. Class lock [Lock Manager]
   lock_object(thread_p, &class_oid, NULL, SCH_S_LOCK, LK_UNCOND_LOCK)
     → Protects against concurrent DDL (ALTER TABLE, DROP TABLE)
     → Lock-free hashmap lookup for LK_RES
     → SCH_S is compatible with IS/IX/S/SIX (any DML lock)
     → Returns LK_GRANTED immediately

4. Heap page fix [Buffer Pool + Latching]
   pgbuf_fix(thread_p, heap_vpid, OLD_PAGE, PGBUF_LATCH_READ, UNCONDITIONAL)
     → Hash lookup: find BCB by VPID
     → CAS on atomic_latch: increment fcnt, set READ mode
     → If CAS fails (writer holds page): wait on next_wait_thrd
     → Return page pointer

5. Record visibility check [MVCC]
   For each record on the page:
     mvcc_satisfies_snapshot(&rec_header, &snapshot)
       → Check mvcc_ins_id against snapshot bounds
       → Check mvcc_del_id against snapshot bounds
       → If SNAPSHOT_SATISFIED: add record to result set
       → If TOO_NEW_FOR_SNAPSHOT: follow prev_version_lsa (Flow 8.4)
       → If TOO_OLD_FOR_SNAPSHOT: skip record (already deleted)

6. Page release [Buffer Pool]
   pgbuf_unfix(thread_p, page)
     → CAS on atomic_latch: decrement fcnt
     → If fcnt reaches 0 and waiter_exists: wake first waiter

No instance object locks acquired. No log records written.
Snapshot provides consistent read without blocking any writers.
```

### 8.2 Flow: UPDATE With WAL and Locking

An `UPDATE` demonstrates the full concurrency stack: locking, MVCC versioning, WAL logging, and buffer pool latching.

```
Client: UPDATE t SET val = 'new' WHERE id = 100

1. Class lock [Lock Manager]
   lock_object(thread_p, &class_oid, NULL, IX_LOCK, LK_UNCOND_LOCK)
     → Intent-exclusive on the class (allows concurrent IX/IS, blocks S/X/SCH_M)
     → Lock-free hashmap find_or_insert for class LK_RES
     → If compatible: add to holder list, return LK_GRANTED

2. Instance lock [Lock Manager]
   lock_object(thread_p, &instance_oid, &class_oid, X_LOCK, LK_UNCOND_LOCK)
     → Exclusive on the row being updated
     → Lock-free hashmap find_or_insert for instance LK_RES
     → pthread_mutex_lock(&res->res_mutex)
     → Check lock_Comp[X_LOCK][res->total_holders_mode]
     → If incompatible: add to waiter list, set LOCK_SUSPENDED
       → Thread blocks on wakeup_cond
       → WFG daemon may detect cycle → victim gets LOCK_RESUMED_DEADLOCK_TIMEOUT
     → If compatible: add to holder, return LK_GRANTED
     → Lock escalation check: if ngranules > 10, consider escalation

3. Page fix [Buffer Pool + Latching]
   pgbuf_fix(thread_p, heap_vpid, OLD_PAGE, PGBUF_LATCH_WRITE, UNCONDITIONAL)
     → CAS: set mode to WRITE if fcnt == 0
     → If contended: wait on BCB mutex + next_wait_thrd
     → Exclusive access to the page

4. MVCC visibility check [MVCC]
   mvcc_satisfies_snapshot(&rec_header, &snapshot) → must be SNAPSHOT_SATISFIED
   Verify no concurrent delete: check mvcc_del_id is not set

5. WAL logging [WAL]
   log_append_undoredo_data(thread_p, RVHF_MVCC_UPDATE, ...)
     → Create LOG_MVCC_UNDOREDO_DATA record containing:
       - Undo data: complete old record content + old MVCC header
       - Redo data: new record content + new MVCC header
       - MVCC metadata: old ins_id, del_id, prev_version_lsa
     → Append to log page buffer (advance log append LSA)
     → The LSA of this record becomes the new prev_version_lsa

6. Heap modification [Storage + MVCC]
   Write new record content to heap page:
     → Set new MVCC_REC_HEADER:
       - mvcc_ins_id = current transaction's MVCCID
       - prev_version_lsa = LSA of the undo log record from step 5
       - mvcc_flag |= OR_MVCC_FLAG_VALID_PREV_VERSION
     → Set old record's mvcc_del_id = current MVCCID
       (marks old version as logically deleted for this transaction)
   heap_update_set_prev_version(thread_p, ...)  -- heap_file.c:893

7. Dirty marking [Buffer Pool + WAL protocol]
   pgbuf_set_dirty(thread_p, page, FREE)
     → Set BCB dirty flag
     → Update BCB oldest_unflush_lsa = max(current, log_append_lsa)
     → pgbuf_unfix: release WRITE latch (page remains dirty in buffer)

8. Commit [WAL + Lock Manager + MVCC]
   log_commit(thread_p)
     → Write LOG_COMMIT record to log buffer
     → logpb_flush_pages(thread_p, flush_lsa):
       - Check group commit configuration
       - If group commit: wait on gc_cond for batch flush
       - If sync: wake log flush daemon, wait for nxio_lsa >= flush_lsa
     → logtb_complete_mvcc(thread_p):
       - Clear transaction from mvcc_trans_status active bit
       - Advance highest_completed_mvccid
       - Increment m_version (atomic) — invalidates concurrent snapshot copies
   lock_unlock_all(thread_p)
     → Walk tran_next chain
     → For each LK_ENTRY: remove from holder, recalculate total_holders_mode
     → Wake compatible waiters
     → Some entries may move to non2pl list before final cleanup
```

### 8.3 Flow: Crash Recovery

After a crash, recovery reconstructs a consistent state from the WAL.

```
Server restart detects incomplete shutdown (log header flag)

1. Find last checkpoint [WAL]
   Read log header from log volume → last checkpoint LSA
   Read LOG_END_CHKPT record at checkpoint LSA:
     → Extract redo_lsa (oldest dirty page LSN at checkpoint time)
     → Extract ntrans active transactions
     → For each: read LOG_INFO_CHKPT_TRANS with trid, state, head_lsa,
       tail_lsa, undo_nxlsa, savept_lsa

2. Analysis phase [WAL + Transaction Table]
   log_recovery_analysis(start_redo_lsa, end_redo_lsa)
     → Scan log forward from checkpoint LSA to end-of-log
     → For each LOG_COMMIT: mark transaction as committed
     → For each LOG_ABORT: mark transaction as aborted
     → For transactions still active at end-of-log: mark as needing undo
     → Compute redo start point: min(checkpoint.redo_lsa, earliest active lsa)
     → Compute redo end point: end-of-log LSA

3. Redo phase [WAL + Buffer Pool] (parallel)
   log_recovery_redo(start_redo_lsa, end_redo_lsa)
     → Create redo_parallel instance:
       - Allocate worker pool with N threads
       - Pre-allocate 1M reusable redo_job objects (ONE_M constant)
     → For each log record from start_redo_lsa forward:
       a. Create redo_job with VPID, LSA, and record data
       b. add(job): hash VPID to worker index for serial per-page execution
       c. Worker thread dequeues job:
          - pgbuf_fix(vpid, RECOVERY_PAGE, PGBUF_LATCH_WRITE)
          - Compare: if page_lsa < log_record_lsa → page is stale, apply redo
          - Compare: if page_lsa >= log_record_lsa → skip (already current)
          - Call registered redo function (RVHF_*, RVBT_*, RVDK_*, etc.)
          - pgbuf_set_dirty() + pgbuf_unfix()
     → min_unapplied_log_lsa_monitoring:
       Track lowest not-yet-applied LSN across all workers
       Prevent premature log archive deletion
     → wait_for_termination_and_stop_execution(): join all workers

4. Undo phase [WAL + Lock Manager]
   log_recovery_undo()
     → For each transaction in TRAN_UNACTIVE_ABORTED state:
       a. Start at undo_nxlsa (next record to undo for this transaction)
       b. Read log record, follow prev_tranlsa chain backwards:
          - pgbuf_fix the affected page (PGBUF_LATCH_WRITE)
          - Call registered undo function to reverse the change
          - Write LOG_COMPENSATE record (prevents re-undo on second crash)
          - pgbuf_set_dirty() + pgbuf_unfix()
       c. Continue until head_lsa reached (all records undone)
     → lock_reacquire_crash_locks():
       Rebuild lock table for in-doubt 2PC transactions

5. System ready
   All committed operations are redone. All aborted operations are undone.
   Transaction table is clean. Vacuum daemon resumes from persisted state.
   Normal client connections accepted.
```

### 8.4 Flow: MVCC Version Traversal via prev_version_lsa

When a reader's snapshot cannot see the current version of a record, CUBRID traverses the version chain through the WAL to find a visible older version. This is the most architecturally distinctive cross-cutting flow in CUBRID.

```
Scenario:
  Reader transaction T2 with snapshot S (highest_completed = MVCCID 50)
  Record R current version: mvcc_ins_id = 60 (T3, not visible to S)
  R.prev_version_lsa = {pageid=1000, offset=128}

1. Initial visibility check [Heap + MVCC]
   heap_get(thread_p, &oid, recdes, snapshot, ...)
     → pgbuf_fix(heap_page, PGBUF_LATCH_READ)
     → Read current MVCC_REC_HEADER from record R:
       mvcc_ins_id=60, mvcc_del_id=none, prev_version_lsa={1000,128}
     → mvcc_satisfies_snapshot(rec_header, snapshot):
       - ins_id=60 > highest_completed=50 → TOO_NEW_FOR_SNAPSHOT
     → prev_version_lsa is not NULL_LSA → version chain exists
     → Call heap_get_visible_version_from_log()

2. First chain hop [WAL + Buffer Pool]
   heap_get_visible_version_from_log(thread_p, &oid, recdes, snapshot)
     → Follow prev_version_lsa = {pageid=1000, offset=128}
     → pgbuf_fix(thread_p, log_page_vpid={vol=log, page=1000},
                 OLD_PAGE, PGBUF_LATCH_READ, UNCONDITIONAL)
       - May require I/O if log page is not in buffer (possibly archived)
     → Read LOG_MVCC_UNDOREDO_DATA record at offset 128:
       - Extract undo data → reconstruct previous record content
       - Extract previous MVCC_REC_HEADER:
         mvcc_ins_id=40, mvcc_del_id=60, prev_version_lsa={800,64}
     → pgbuf_unfix(thread_p, log_page)

3. Visibility check on reconstructed version [MVCC]
   mvcc_satisfies_snapshot(prev_header, snapshot):
     - ins_id=40: 40 < lowest_active? If yes → committed before snapshot
     - del_id=60: 60 > highest_completed=50 → deletion not visible to S
     → SNAPSHOT_SATISFIED (inserted before snapshot, deletion not yet visible)
     → Return this version to the reader

   [Alternative: if this version was also TOO_NEW_FOR_SNAPSHOT,
    and its prev_version_lsa={800,64} is not NULL_LSA,
    recurse to step 2 with the next LSA in the chain]

4. Chain termination conditions:
   → prev_version_lsa is NULL_LSA → end of chain, no visible version
   → Version is SNAPSHOT_SATISFIED → return it
   → Version is TOO_OLD_FOR_SNAPSHOT → skip to next (or end)

Example 3-version chain:
  Heap: R(ins=60, del=none, prev→{1000,128})   -- too new for T2
  Log@{1000,128}: V2(ins=40, del=60, prev→{800,64})  -- visible to T2!
  Log@{800,64}:   V1(ins=10, del=40, prev→NULL)  -- too old for T2
```

**Performance characteristics**:
- Each chain hop requires fixing a log page in the buffer pool, acquiring a read latch, reading the record, then unfixing. If the log page is not cached (common for old versions), this triggers a disk read.
- Archived log volumes may need to be opened and read, adding significant latency compared to reading from the active log.
- Chain length is bounded by vacuum activity: the vacuum daemon clears `prev_version_lsa` pointers when no active transaction needs the old version, terminating the chain.
- In the worst case (vacuum lag + long-running transactions), chains can span many log pages and multiple archive volumes, degrading read performance significantly.

**Comparison with other RDBMS**: PostgreSQL stores old versions in the heap itself (in-place, creating HOT chains for heap-only tuples), requiring periodic VACUUM to reclaim dead tuples. MySQL/InnoDB uses a separate undo tablespace with rollback segments. CUBRID's WAL-embedded approach avoids both a separate version store and heap bloat, but couples old-version read performance to WAL I/O throughput and archive availability.

---

## 9. Summary & Key Design Decisions

### CUBRID-Specific Design Choices

| Decision | Rationale | Tradeoff |
|----------|-----------|----------|
| **Hybrid locking + MVCC** | Readers never block writers (MVCC); writers never conflict undetected (locks) | Complexity of maintaining both lock state and version visibility simultaneously |
| **prev_version_lsa in WAL** | Avoids separate version store; leverages existing WAL infrastructure; version data included in backups/archives automatically | Old-version reads require log page I/O; chain length depends on vacuum; coupling between recovery and version storage |
| **Lock-free hashmap for LK_RES** | Eliminates global lock table contention for concurrent lock lookups; O(1) average find | Hazard-pointer reclamation adds per-thread overhead; hybrid old/new implementation adds code complexity |
| **Non-2PL list** | Early lock release enables higher concurrency while MVCC maintains logical isolation | Adds complexity to lock release and commit processing; 159 references in lock_manager.c |
| **Three-zone LRU** | Separates hot/warm/cold pages for more intelligent victim selection; configurable ratios | Zone boundary management adds overhead; requires tuning for workload |
| **Ordered latch protocol** | Eliminates latch deadlocks for multi-page heap operations without global ordering | Requires unfix/refix cycles when page access order violates rank; callers must handle page_was_unfixed |
| **Parallel redo recovery** | Reduces crash recovery time by distributing redo across CPU cores; VPID-based partitioning | 1M pre-allocated job objects use memory; VPID distribution may cause load imbalance |
| **Group commit** | Amortizes log flush I/O across concurrent committing transactions | Adds commit latency for individual transactions; configurable interval tradeoff |
| **Vacuum daemon** | Background reclamation keeps version chains short; master-worker separation for scalability | Vacuum lag creates longer chains; WAL archive dependency; 8,470 lines of complex daemon code |
| **Atomic BCB latch** | CAS-based latch acquisition avoids mutex overhead in the uncontended case | Falls back to mutex + wait queue when contended; requires 64-bit CAS support |
| **THREAD_ENTRY as universal context** | All per-thread state in one object; enables efficient function dispatch without global lookups | Large struct (200+ fields); every function takes thread_p as first parameter |

### Architecture Themes

1. **Pragmatic hybrid synchronization**: CUBRID does not commit to a single concurrency philosophy. It combines pessimistic locking (write-write conflicts), optimistic MVCC (read-write), lock-free data structures (hot internal tables), and traditional mutexes (coarse-grained coordination). Each mechanism is used where its performance characteristics match the access pattern.

2. **WAL as backbone**: The WAL serves triple duty: durability (crash recovery via redo/undo), version storage (prev_version_lsa chains for MVCC), and vacuum work tracking (log block consumption by vacuum daemon). This consolidation simplifies the system architecture but creates coupling between these concerns.

3. **Per-thread context**: `THREAD_ENTRY` carries all per-thread state — transaction index, lock wait state, error context, hazard pointer descriptors, resource trackers. This is the glue that connects all subsystems without requiring global lookups or thread-local storage indirection.

4. **Layered synchronization granularity**: From finest to coarsest — CAS atomics (BCB latches, lock-free hashmap operations) → per-resource mutexes (LK_RES.res_mutex) → reader-writer critical sections (CSECT_LOG, CSECT_WFG, CSECT_TRAN_TABLE) → transaction-level locks (S, X, IX, SIX). Each layer has a different contention profile, holding duration, and failure mode.

5. **Background daemons for maintenance**: Critical maintenance tasks (log flushing, checkpointing, vacuum, page flushing, deadlock detection) run as daemon threads rather than being piggy-backed on client operations. This keeps client request latency predictable and allows maintenance to be scheduled and throttled independently.

### Key Files Reference

| Subsystem | Primary File | Lines | Entry Points |
|-----------|-------------|-------|--------------|
| Lock Manager | `src/transaction/lock_manager.c` | 9,939 | `lock_object`, `lock_unlock_object` |
| Lock Compatibility | `src/transaction/lock_table.c` | 240 | `lock_Comp`, `lock_Conv` |
| Deadlock Detection | `src/transaction/wait_for_graph.c` | 2,434 | `wfg_detect_cycle` |
| MVCC Visibility | `src/transaction/mvcc.c` | 722 | `mvcc_satisfies_snapshot` |
| MVCC Active Tracking | `src/transaction/mvcc_active_tran.hpp` | 676 | `is_active(mvccid)` |
| MVCC Table | `src/transaction/mvcc_table.hpp` | 777 | `mvcc_trans_status` |
| Snapshot Lifecycle | `src/transaction/log_tran_table.c` | 6,309 | `logtb_get_mvcc_snapshot` |
| Version Chain | `src/storage/heap_file.c` | 26,759 | `heap_get_visible_version_from_log` |
| Vacuum | `src/query/vacuum.c` | 8,470 | `vacuum_process_log_block`, `vacuum_heap_page` |
| WAL Core | `src/transaction/log_manager.c` | 15,278 | `log_commit`, `log_abort` |
| Log Page Buffer | `src/transaction/log_page_buffer.c` | 11,598 | `logpb_flush_pages`, `logpb_checkpoint` |
| Log Records | `src/transaction/log_record.hpp` | 470 | `LOG_RECORD_HEADER`, `LOG_RECTYPE` |
| Recovery | `src/transaction/log_recovery.c` | 6,500 | `log_recovery_redo`, `log_recovery_undo` |
| Parallel Redo | `src/transaction/log_recovery_redo_parallel.hpp` | 206 | `redo_parallel` |
| Buffer Pool | `src/storage/page_buffer.c` | 16,931 | `pgbuf_fix`, `pgbuf_ordered_fix` |
| Critical Sections | `src/thread/critical_section.c` | 2,581 | `csect_enter`, `csect_exit` |
| Thread Entry | `src/thread/thread_entry.hpp` | 386 | `cubthread::entry` |
| Lock-Free Hashmap | `src/base/lockfree_hashmap.hpp` | 1,445 | `find`, `insert`, `erase` |
| Lock-Free Wrapper | `src/thread/thread_lockfree_hash_map.hpp` | 87 | `cubthread::lockfree_hashmap` |
| Lock-Free GC | `src/base/lockfree_transaction_descriptor.hpp` | 97 | `retire_node`, `reclaim_retired_list` |
| Lock-Free Freelist | `src/base/lockfree_freelist.hpp` | 101 | `claim`, `retire` |
